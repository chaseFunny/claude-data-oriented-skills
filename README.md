# claude-data-oriented-skills

> **"You need to be thinking bigger. 1% faster is not enough. This should be 10x faster."**
> ([Jarred Sumner](https://x.com/jarredsumner/status/2117046610404442215))

A performance-optimization skill for coding agents, built on data-oriented design. Inspired by [Jarred Sumner's prompts](https://x.com/jarredsumner/status/2117037940031496252) for doing perf work with Claude.

English | [简体中文](./README.zh.md)

> **Unofficial community project.** Not affiliated with or endorsed by Anthropic, Jarred Sumner, or Oven (Bun).

## The idea

From Jarred's [post](https://x.com/jarredsumner/status/2117037940031496252):

> how do we rewrite this to better leverage data-oriented design principles

> can you replace those hash tables with a flat array? can we make the pointers an index into the array? can we pack the most commonly used data into upper bits of the index?

> caching is cope for a slow design. do not add caches and delete all existing caches. instead, redesign it to be faster

> should this be a structure of arrays instead of an array of structures?

In his replies he added bitsets, inline-capacity stack lists, and arenas. On their own these prompts can still lead an agent to optimize by guesswork or delete a cache that was actually needed. This skill adds a measured workflow, advice on how to prompt, when each principle applies and when it doesn't, and guardrails, using ideas from the thread's replies.

## What's inside

| Section | What it tells the agent |
|---|---|
| **Why: cache lines** | Every principle loads fewer 64-byte lines and uses more of each one. |
| **0. Workflow** | Baseline → show hot path and allocations → redesign data → run tests → re-measure. No number, no claim. |
| **How to prompt** | Name the access pattern and a concrete layout, not "make it faster". Sample the diff as it goes. |
| **1. Data-oriented design** | Lay out data for the loop that uses it. |
| **2. Flat arrays, indices** | Arrays instead of hash tables, `u32` indices instead of pointers: movable, near-free serialization. |
| **3. Packed bits** | Put hot tags/flags in the upper bits of the index. |
| **4. SoA vs AoS** | Pick the layout that matches the access pattern. |
| **5. Bitsets** | One bit per flag, not one byte. |
| **6. Arena + stack lists** | Arena for long-lived data, inline-capacity stack lists for temporaries. |
| **7. Caching is cope** | Fix the slow path instead of hiding it behind a cache. |
| **Guardrails** | Keep network/IO/DB/CDN caches and pure memoization unless a benchmark with repeated lookups proves otherwise. Never mass-delete caches. Keep tests green. |
| **Quick prompts** | Jarred's prompts from the post and replies, verbatim, plus a measure-first prompt. |

Each principle has *use when*, *not when*, and a short before/after example in TypeScript, Rust, Zig, or C.

## Install

**Claude Code: plugin**

```
/plugin marketplace add chaseFunny/claude-data-oriented-skills
/plugin install claude-data-oriented-skills@claude-data-oriented-skills
```

**Claude Code: personal skill**

```bash
git clone https://github.com/chaseFunny/claude-data-oriented-skills
mkdir -p ~/.claude/skills
cp -r claude-data-oriented-skills/skills/data-oriented-perf ~/.claude/skills/
```

**Claude Code: per-project CLAUDE.md**

```bash
curl https://raw.githubusercontent.com/chaseFunny/claude-data-oriented-skills/main/CLAUDE.md >> CLAUDE.md
```

**Codex**

```bash
mkdir -p ~/.agents/skills   # or .agents/skills inside a repo
cp -r claude-data-oriented-skills/skills/data-oriented-perf ~/.agents/skills/
```

**Cursor**

Copy `.cursor/rules/data-oriented-perf.mdc` into your project's `.cursor/rules/`, or copy the skill folder into `~/.cursor/skills/`.

**Anything else:** paste `CLAUDE.md` into the agent's instruction file (`AGENTS.md`, etc.).

## Usage

The skill loads when you ask the agent to make something faster. You can also send any of the quick prompts directly, for example:

```
Profile the parser on bench/large.ts, then tell me whether the token list should be a structure of arrays.
```

## Repository layout

```
skills/data-oriented-perf/SKILL.md   # the skill (source of truth)
CLAUDE.md                            # same guidelines as a drop-in instruction file
.cursor/rules/data-oriented-perf.mdc # same guidelines as a Cursor rule
.claude-plugin/                      # Claude Code plugin + marketplace manifests
```

When editing, change `SKILL.md` first and keep `CLAUDE.md` and the Cursor rule in sync.

## Credits

- Prompts: [Jarred Sumner](https://x.com/jarredsumner): [original post](https://x.com/jarredsumner/status/2117037940031496252), [reply](https://x.com/jarredsumner/status/2117039564221256052), [follow-up](https://x.com/jarredsumner/status/2117046610404442215).
- Format inspired by [forrestchang/andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills).

## Thanks / ideas from the community

- [@bradleymeck](https://x.com/bradleymeck/status/2117038669202620791) and [Jarred](https://x.com/jarredsumner/status/2117039149379412270): arena first, stack for temporaries
- [@acdlite](https://x.com/acdlite/status/2117094796426653891): stacks and bitsets win through cache locality
- [@TinyRouterHQ](https://x.com/TinyRouterHQ/status/2117094059835589104), [@jimmyyangrz](https://x.com/jimmyyangrz/status/2117061430704996656): name the access pattern and layout, not the symptom
- [@amdmszOvO](https://x.com/amdmszOvO/status/2117076542627140039): show the hot path and allocations before changing anything
- [Jarred](https://x.com/jarredsumner/status/2117079330035728578): supervise by sampling the diff, like pausing a debugger
- [@zhao_spenc](https://x.com/zhao_spenc/status/2117073400938270930), [@perperp_er](https://x.com/perperp_er/status/2117063323820900649): indices survive reallocation and serialize almost for free
- [@dhillonpost](https://x.com/dhillonpost/status/2117088944831770728), [@John_zhong324](https://x.com/John_zhong324/status/2117070275070288176): benchmark cache removal with repeated lookups; weigh invalidation vs recomputation
- [@akashc777](https://x.com/akashc777): removing a Redis removes an ops burden for self-hosted customers
- [@kimindiehacker](https://x.com/kimindiehacker/status/2117081654653255937): DOD can hurt readability, so keep it to hot paths

## License

[MIT](./LICENSE)
