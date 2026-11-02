# CLAUDE.md

> "You need to be thinking bigger. 1% faster is not enough. This should be 10x faster." ([Jarred Sumner](https://x.com/jarredsumner/status/2117046610404442215))

Guidelines for performance work, derived from [Jarred Sumner's prompts](https://x.com/jarredsumner/status/2117037940031496252) for optimizing with Claude and the replies to them. The core idea: **the shape of your data in memory decides your speed.** Fix the layout, not the symptoms. Small tweaks give 1%; a better layout gives 10x.

**Tradeoff:** These rules bias toward redesign over quick patches. They pay off on measured hot paths, not on code that runs once.

## Why layout wins: cache lines

The CPU loads memory in 64-byte cache lines, and a cache miss can cost about 100x more than an L1 hit. Every principle below tries to put more useful data in each line you load and to load fewer lines:

- Flat arrays and SoA: the loop streams through contiguous lines and reads every byte it loads.
- Indices instead of pointers: 4 bytes instead of 8, no pointer chasing across the heap.
- Bitsets: one line holds 512 flags instead of 64 bools.
- Stack and arena memory: already hot in cache and contiguous, with no allocator bookkeeping between items.

Stacks and bitsets are fast mostly because of cache locality ([@acdlite](https://x.com/acdlite/status/2117094796426653891)), not because they avoid some special cost.

## 0. Workflow: measure, redesign, measure

**Never optimize by guesswork. No number, no claim.**

1. **Baseline.** Write or find a benchmark that exercises the real workload. Record time, allocations, and memory (e.g. `hyperfine`, `perf stat`, `cargo bench`, `bun --cpu-prof`, `node --cpu-prof`). Use release builds and realistic input sizes.
2. **Profile and report before touching code.** Find the hot path with a profiler (`perf`, Instruments, `samply`, Chrome DevTools, `pprof`). First show the hot path and allocations per iteration, then propose a change. Optimize only what shows up.
3. **Redesign the data.** Ask how the hot loop touches memory: what it reads, in what order, how many pointer hops, how many bytes per item, how many allocations. Apply the principles below.
4. **Verify correctness.** All existing tests must pass. Add a test if the rewrite touches untested behavior.
5. **Re-measure.** Run the same benchmark, same machine, several runs. Report before → after with numbers (median and spread). If it isn't faster, revert it.

Report results like: `parse 10k files: 412 ms → 268 ms (-35%), peak RSS 310 MB → 190 MB, allocs/iter 18 → 0, 5 runs, ±2%`.

## How to prompt the agent

**Name the access pattern and a concrete layout, not a vague goal.**

- "Make it faster" gets a cache bolted on. "Use data-oriented design" may get the same array of structs, renamed ([@jimmyyangrz](https://x.com/jimmyyangrz/status/2117061430704996656), [@TinyRouterHQ](https://x.com/TinyRouterHQ/status/2117094059835589104)).
- Say instead: "2 hot fields, 1M structs, one loop: make it a structure of arrays." Or: "replace this `HashMap<u32, Node>` with a `Vec<Node>` and `u32` indices."
- Make it measure first: "before changing anything, show me the hot path and allocations per iteration" ([@amdmszOvO](https://x.com/amdmszOvO/status/2117076542627140039)).
- Supervise by sampling: look at the diff at random points while the agent works, like pausing and resuming a debugger, and redirect early ([Jarred](https://x.com/jarredsumner/status/2117079330035728578)).

When you are the agent: if the request is vague, state the access pattern you found in the profile and the layout you plan to use before you write code.

## 1. Data-oriented design

**Design around how data is accessed, not around what objects "are".**

- Group data by the loop that uses it. Hot fields together, cold fields elsewhere.
- Prefer contiguous arrays, linear scans, and batches over object graphs and one-at-a-time calls.
- Fewer, smaller, denser items mean fewer cache misses.

**Use when:** a loop processes many items of the same kind (tokens, AST nodes, entities, rows, particles).
**Not when:** the code runs a handful of times, is I/O-bound, or isn't in a measured hot path. DOD can hurt readability ([@kimindiehacker](https://x.com/kimindiehacker/status/2117081654653255937)), so spend it only where a profile says it pays.

```ts
// Before: one object per particle, scattered across the heap
type P = { x: number; y: number; vx: number; vy: number; name: string };
for (const p of particles) { p.x += p.vx * dt; p.y += p.vy * dt; }

// After: contiguous typed arrays, the loop touches only what it needs
const x = new Float64Array(n), y = new Float64Array(n);
const vx = new Float64Array(n), vy = new Float64Array(n);
for (let i = 0; i < n; i++) { x[i] += vx[i] * dt; y[i] += vy[i] * dt; }
```

## 2. Flat arrays over hash tables, indices over pointers

**If keys are dense or you control the IDs, an array is the hash table.**

- Assign sequential IDs and store items in one `Vec`/array; look up with `items[id]`.
- Replace pointers/references with `u32` indices into that array: half the size of a 64-bit pointer, no per-node allocation, no lifetime/borrow fights.
- The backing array can grow, move, or reallocate without rewriting a single reference, as long as element order stays fixed ([@zhao_spenc](https://x.com/zhao_spenc/status/2117073400938270930)).
- Serialization is almost free: write the arrays out as-is, because indices stay valid ([@perperp_er](https://x.com/perperp_er/status/2117063323820900649)).
- Free everything at once by dropping the array.

**Use when:** IDs are assigned by you, the set is mostly append-only, or a hash lookup shows up in the profile.
**Not when:** keys are sparse or external (user strings, UUIDs), items are frequently deleted out of order (unless you add a free list or generation counter), or the table is small and cold.

```rust
// Before: hash lookups and boxed nodes
struct Node { kind: Kind, children: Vec<Box<Node>> }
let symbols: HashMap<String, Box<Symbol>> = HashMap::new();

// After: one array per type, u32 indices as references
#[derive(Clone, Copy)] struct NodeId(u32);
struct Node { kind: Kind, first_child: NodeId, next_sibling: NodeId }
struct Ast { nodes: Vec<Node> }            // ast.nodes[id.0 as usize]
struct Symbols { names: Vec<StrId>, decl: Vec<NodeId> } // SymbolId = index
```

## 3. Pack hot data into the upper bits of the index

**An index rarely needs all 32 bits. Put the most-checked data in the spare ones.**

- Store a small tag (kind, flags, generation) in the top bits so the common check needs no memory load at all.
- Decide the bit budget up front and assert the limit (e.g. 28-bit index = 268M items).

**Use when:** the hot path branches on a small enum or flag before dereferencing, and you have spare bits.
**Not when:** the item count could exceed the reduced range, the tag needs more than a few bits, or the code is not hot (bit tricks cost readability).

```zig
// Before: load the node just to check its kind
if (ast.nodes[ref].kind == .identifier) { ... }

// After: kind lives in the reference itself
const Ref = packed struct(u32) {
    index: u28, // max 268,435,455 items -- assert on insert
    kind: Kind, // enum(u4), upper 4 bits
};
if (ref.kind == .identifier) { ... } // no memory access
```

## 4. Structure of arrays vs array of structures

**Ask which fields the hot loop reads.**

- **SoA** (one array per field) when loops touch a few fields of many items: denser cache lines, easy SIMD.
- **AoS** (array of structs) when code usually reads all fields of one item together, or does random single-item access.
- Hybrid is fine: split hot fields (SoA) from cold fields (separate array or AoS).

**Use SoA when:** a profile shows a loop reading 1–2 fields out of a wide struct.
**Not when:** every access needs the whole record, or the count is small.

```c
// Before (AoS): summing prices drags 64-byte records through cache
struct Item { uint64_t id; double price; char name[40]; uint32_t qty; };
for (size_t i = 0; i < n; i++) total += items[i].price;

// After (SoA): the loop reads exactly the bytes it needs
struct Items { uint64_t *id; double *price; char (*name)[40]; uint32_t *qty; size_t len; };
for (size_t i = 0; i < it.len; i++) total += it.price[i];
```

## 5. Bitsets instead of arrays of bools

**A bool is 8 bits holding 1 bit of information.**

- Store flags in a bitset (`u64` words): 8x less memory, 512 flags per cache line, and whole-set operations (union, count, find-first) run 64 flags per instruction.

**Use when:** you keep many flags (visited, dirty, live, reachable) over dense IDs.
**Not when:** there are only a few flags, or several threads write neighboring flags at once (they would contend on the same word; use atomics or per-thread sets).

```rust
// Before: 1 byte per flag
let mut visited = vec![false; n];
visited[i] = true;

// After: 1 bit per flag
let mut visited = vec![0u64; (n + 63) / 64];
visited[i / 64] |= 1 << (i % 64);
let seen = visited[i / 64] & (1 << (i % 64)) != 0;
```

## 6. Arena first, stack for temporaries

**Allocate long-lived data from an arena; keep short-lived lists on the stack.**

Jarred usually starts with an arena, and uses stack-allocated lists for temporaries because an arena only frees in bulk ([exchange with @bradleymeck](https://x.com/jarredsumner/status/2117039149379412270)).

- **Arena:** everything that lives as long as a phase (a parse, a request, a frame) comes from one arena and is freed in one call. No per-object free, no fragmentation, objects packed together.
- **Stack lists with inline capacity (small-vector):** a temporary list that is usually small lives on the stack and spills to the heap only when it outgrows its inline capacity. No allocation in the common case, and the memory is already hot in cache. Don't put such temporaries in the arena: they would stay alive until the whole arena is freed.

**Use when:** many objects share a lifetime (arena), or a hot function builds a short list it throws away (stack list).
**Not when:** objects have mixed or unpredictable lifetimes and must be freed one by one (arena grows unbounded), or a list is usually large (inline capacity just wastes stack space).

```zig
// Before: one heap allocation per node, freed one by one
const node = try gpa.create(Node);
defer gpa.destroy(node);

// After: arena for the whole parse, freed in one call
var arena = std.heap.ArenaAllocator.init(gpa);
defer arena.deinit();
const node = try arena.allocator().create(Node);
```

```rust
// Before: heap allocation on every call for a usually tiny list
let mut args: Vec<NodeId> = Vec::new();

// After: inline capacity of 8 on the stack, heap only if it grows past 8
let mut args: SmallVec<[NodeId; 8]> = SmallVec::new(); // smallvec crate
```

## 7. "Caching is cope for a slow design"

**Don't paper over a slow path with a cache. Make the path fast.**

- Before adding a cache, ask why the computation is slow: wrong data structure, repeated work, bad layout.
- Redesign so the answer is cheap to compute or computed once as part of the main data (an index, a table, a precomputed field).
- In-process caches add invalidation bugs, memory, and hidden state, and often hide the real problem.
- Fewer caches can also mean less to operate: in self-hosted software, removing a required Redis removes an ops burden for every customer (@akashc777).

**Use when:** a cache sits in front of in-process, hot-path computation that a better layout would make cheap.
**Not when:** see Guardrails.

```ts
// Before: cache in front of an O(n) scan
const cache = new Map<number, Pos>();
function position(offset: number): Pos {
  let hit = cache.get(offset); if (hit) return hit;
  hit = scanFromStart(source, offset); // O(n)
  cache.set(offset, hit); return hit;
}

// After: record line starts once while lexing, then binary search: O(log n), no cache
const lineStarts: number[] = [0]; // push(i + 1) on every '\n' during lexing
function position(offset: number): Pos {
  let lo = 0, hi = lineStarts.length - 1;
  while (lo < hi) { const mid = (lo + hi + 1) >> 1; if (lineStarts[mid] <= offset) lo = mid; else hi = mid - 1; }
  return { line: lo + 1, column: offset - lineStarts[lo] + 1 };
}
```

## Guardrails

- **"Delete all caches" targets in-process, hot-path data structures in low-level or runtime code** (parsers, interpreters, bundlers, engines). It is not a universal rule.
- **Do not delete** caches for network, disk, or database results, HTTP/CDN caches, build caches, or memoization of expensive pure computations, **unless a benchmark proves the redesign is faster on the real workload**. Ask before removing any of these.
- **Benchmark cache removal on a realistic workload that includes repeated lookups.** A redesign can win a single-pass benchmark and lose when the same keys come back ([@dhillonpost](https://x.com/dhillonpost/status/2117088944831770728)). Weigh invalidation cost against recomputation cost ([@John_zhong324](https://x.com/John_zhong324/status/2117070275070288176)).
- **Agents with write access must never mass-delete caches.** Remove one cache at a time, each with a benchmark that shows the win.
- **Correctness first.** Keep every existing test passing. A faster wrong answer is a regression.
- **Stay in scope.** Change the hot path the profile pointed to; don't rewrite cold code for style.
- **Show the numbers.** Every claimed win includes the benchmark command and before/after results. If a change doesn't help, revert it and say so.
- **Keep it readable.** Name packed bit fields, document invariants (index limits, array ownership, arena lifetimes), and add asserts.

## Quick prompts

Jarred Sumner's prompts, verbatim. From the [original post](https://x.com/jarredsumner/status/2117037940031496252):

> how do we rewrite this to better leverage data-oriented design principles

> can you replace those hash tables with a flat array? can we make the pointers an index into the array? can we pack the most commonly used data into upper bits of the index?

> caching is cope for a slow design. do not add caches and delete all existing caches. instead, redesign it to be faster

> should this be a structure of arrays instead of an array of structures?

From [his reply](https://x.com/jarredsumner/status/2117039564221256052):

> replace the heap-allocated lists with a stack allocated one with an inline capacity

> do not use an array of bools. Use a bitset instead

From [his follow-up](https://x.com/jarredsumner/status/2117046610404442215):

> You need to be thinking bigger. 1% faster is not enough. This should be 10x faster.

Measure first, from [@amdmszOvO](https://x.com/amdmszOvO/status/2117076542627140039):

> before changing anything, show me the hot path and allocations per iteration

When the user sends one of these, apply the matching section above, and still follow the workflow and guardrails.
