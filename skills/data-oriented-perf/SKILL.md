---
name: data-oriented-perf
description: Data-oriented performance optimization workflow. Use when asked to make code faster, reduce memory or CPU, optimize a hot path, or review performance-sensitive code (runtimes, parsers, compilers, engines, simulations, servers). Profile first, then change data layout (flat arrays, indices instead of pointers, packed bits, struct-of-arrays) instead of adding caches, and prove every change with before/after benchmarks.
license: MIT
---

# Data-Oriented Performance

Guidelines for performance work, derived from [Jarred Sumner's prompts](https://x.com/jarredsumner/status/2117037940031496252) for optimizing with Claude. The core idea: **the shape of your data in memory decides your speed.** Fix the layout, not the symptoms.

**Tradeoff:** These rules bias toward redesign over quick patches. They pay off on measured hot paths, not on code that runs once.

## 0. Workflow: measure, redesign, measure

**Never optimize by guesswork. No number, no claim.**

1. **Baseline.** Write or find a benchmark that exercises the real workload. Record time, allocations, and memory (e.g. `hyperfine`, `perf stat`, `cargo bench`, `zig build -Doptimize=ReleaseFast` + timer, `bun --cpu-prof`, `node --cpu-prof`). Use release builds and realistic input sizes.
2. **Profile.** Find the hot path with a profiler (`perf`, Instruments, `samply`, Chrome DevTools, `pprof`). Optimize only what shows up. Say what you found before changing anything.
3. **Redesign the data.** Ask how the hot loop touches memory: what it reads, in what order, how many pointer hops, how many bytes per item. Apply the principles below.
4. **Verify correctness.** All existing tests must pass. Add a test if the rewrite touches untested behavior.
5. **Re-measure.** Run the same benchmark, same machine, several runs. Report before → after with numbers (median and spread). If it isn't faster, revert it.

Report results like: `parse 10k files: 412 ms → 268 ms (-35%), peak RSS 310 MB → 190 MB, 5 runs, ±2%`.

## 1. Data-oriented design

**Design around how data is accessed, not around what objects "are".**

- Group data by the loop that uses it. Hot fields together, cold fields elsewhere.
- Prefer contiguous arrays, linear scans, and batches over object graphs and one-at-a-time calls.
- Fewer, smaller, denser items = fewer cache misses.

**Use when:** a loop processes many items of the same kind (tokens, AST nodes, entities, rows, particles).
**Not when:** the code runs a handful of times, is I/O-bound, or clarity matters more than the measured gain.

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
- Replace pointers/references with `u32` indices into that array: half the size of a 64-bit pointer, no per-node allocation, trivially serializable, no lifetime/borrow fights.
- Free everything at once by dropping the array (arena style).

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
    kind: Kind, // enum(u4)
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

## 5. "Caching is cope for a slow design"

**Don't paper over a slow path with a cache. Make the path fast.**

- Before adding a cache, ask why the computation is slow: wrong data structure, repeated work, bad layout.
- Redesign so the answer is cheap to compute or computed once as part of the main data (an index, a table, a precomputed field).
- In-process caches add invalidation bugs, memory, and hidden state, and often hide the real problem.

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
- **Correctness first.** Keep every existing test passing. A faster wrong answer is a regression.
- **Stay in scope.** Change the hot path the profile pointed to; don't rewrite cold code for style.
- **Show the numbers.** Every claimed win includes the benchmark command and before/after results. If a change doesn't help, revert it and say so.
- **Keep it readable.** Name packed bit fields, document invariants (index limits, ownership of arrays), and add asserts.

## Quick prompts

Jarred Sumner's original prompts ([source](https://x.com/jarredsumner/status/2117037940031496252)), verbatim:

> how do we rewrite this to better leverage data-oriented design principles

> can you replace those hash tables with a flat array? can we make the pointers an index into the array? can we pack the most commonly used data into upper bits of the index?

> caching is cope for a slow design. do not add caches and delete all existing caches. instead, redesign it to be faster

> should this be a structure of arrays instead of an array of structures?

When the user sends one of these, apply the matching section above, and still follow the workflow and guardrails.
