# claude-data-oriented-skills

A performance-optimization skill for coding agents, built on data-oriented design. Inspired by [Jarred Sumner's prompts](https://x.com/jarredsumner/status/2117037940031496252) for doing perf work with Claude.

English | [简体中文](./README.zh.md)

> **Unofficial community project.** Not affiliated with or endorsed by Anthropic, Jarred Sumner, or Oven (Bun).

## The idea

From Jarred's post:

> how do we rewrite this to better leverage data-oriented design principles

> can you replace those hash tables with a flat array? can we make the pointers an index into the array? can we pack the most commonly used data into upper bits of the index?

> caching is cope for a slow design. do not add caches and delete all existing caches. instead, redesign it to be faster

> should this be a structure of arrays instead of an array of structures?

Four prompts are great, but on their own an agent may optimize by guesswork, or delete a cache that was actually needed. This skill adds a measured workflow, explains when each principle applies and when it doesn't, and sets guardrails.

## What's inside

| Section | What it tells the agent |
|---|---|
| **0. Workflow** | Baseline → profile → redesign data → run tests → re-measure. No number, no claim. |
| **1. Data-oriented design** | Lay out data for the loop that uses it. |
| **2. Flat arrays, indices** | Replace hash tables with arrays and pointers with `u32` indices. |
| **3. Packed bits** | Put hot tags/flags in the upper bits of the index. |
| **4. SoA vs AoS** | Pick the layout that matches the access pattern. |
| **5. Caching is cope** | Fix the slow path instead of hiding it behind a cache. |
| **Guardrails** | Keep network/IO/DB/CDN caches and pure memoization unless a benchmark proves otherwise. Keep tests green. |
| **Quick prompts** | The four original prompts, verbatim. |

Each principle has *use when*, *not when*, and a short before/after example in TypeScript, Rust, Zig, or C.

## Install

**Claude Code: plugin**

```
/plugin marketplace add OWNER/claude-data-oriented-skills
/plugin install claude-data-oriented-skills@claude-data-oriented-skills
```

**Claude Code: personal skill**

```bash
git clone https://github.com/OWNER/claude-data-oriented-skills
mkdir -p ~/.claude/skills
cp -r claude-data-oriented-skills/skills/data-oriented-perf ~/.claude/skills/
```

**Claude Code: per-project CLAUDE.md**

```bash
curl https://raw.githubusercontent.com/OWNER/claude-data-oriented-skills/main/CLAUDE.md >> CLAUDE.md
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

- Prompts: [Jarred Sumner](https://x.com/jarredsumner), [original post](https://x.com/jarredsumner/status/2117037940031496252).
- Format inspired by [forrestchang/andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills).

## License

[MIT](./LICENSE)
