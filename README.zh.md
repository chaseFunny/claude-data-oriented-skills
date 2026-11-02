# claude-data-oriented-skills

一个给编程 Agent 用的性能优化技能，核心方法是面向数据的设计（data-oriented design）。灵感来自 [Jarred Sumner 分享的提示词](https://x.com/jarredsumner/status/2117037940031496252)，那是他用 Claude 做性能优化时常说的几句话。

[English](./README.md) | 简体中文

> **非官方社区项目**，与 Anthropic、Jarred Sumner 及 Oven（Bun）均无关联，也未获其认可。

## 思路

Jarred 原帖中的四句提示词：

> how do we rewrite this to better leverage data-oriented design principles

> can you replace those hash tables with a flat array? can we make the pointers an index into the array? can we pack the most commonly used data into upper bits of the index?

> caching is cope for a slow design. do not add caches and delete all existing caches. instead, redesign it to be faster

> should this be a structure of arrays instead of an array of structures?

这四句很好用，但单独丢给 Agent，它可能凭感觉乱改，或者把真正有用的缓存也删掉。这个技能补上了三样东西：先测量再动手的流程、每条原则的适用和不适用场景，以及明确的边界。

## 内容

| 部分 | 告诉 Agent 什么 |
|---|---|
| **0. 流程** | 建基线 → 跑 profile 找热点 → 改数据布局 → 跑测试 → 再测一遍。没有数字就不算优化。 |
| **1. 面向数据的设计** | 按使用数据的那个循环来组织数据。 |
| **2. 扁平数组和索引** | 用数组代替哈希表，用 `u32` 索引代替指针。 |
| **3. 高位打包** | 把最常用的标记放进索引的高位。 |
| **4. SoA 还是 AoS** | 按访问模式选结构体数组或数组结构体。 |
| **5. 缓存是逃避** | 把慢路径本身改快，而不是用缓存挡住它。 |
| **边界** | 网络、IO、数据库、CDN 缓存和纯函数的记忆化默认保留，除非基准测试证明重新设计更快；测试必须全部通过。 |
| **快捷提示词** | 原帖四句提示词，原文照录。 |

每条原则都写了什么时候用、什么时候别用，并配有 TypeScript、Rust、Zig 或 C 的改写前后小例子。

## 安装

**Claude Code 插件**

```
/plugin marketplace add OWNER/claude-data-oriented-skills
/plugin install claude-data-oriented-skills@claude-data-oriented-skills
```

**Claude Code 个人技能**

```bash
git clone https://github.com/OWNER/claude-data-oriented-skills
mkdir -p ~/.claude/skills
cp -r claude-data-oriented-skills/skills/data-oriented-perf ~/.claude/skills/
```

**按项目追加到 CLAUDE.md**

```bash
curl https://raw.githubusercontent.com/OWNER/claude-data-oriented-skills/main/CLAUDE.md >> CLAUDE.md
```

**Codex**

```bash
mkdir -p ~/.agents/skills   # 或者仓库里的 .agents/skills
cp -r claude-data-oriented-skills/skills/data-oriented-perf ~/.agents/skills/
```

**Cursor**

把 `.cursor/rules/data-oriented-perf.mdc` 复制到项目的 `.cursor/rules/`，或者把技能文件夹复制到 `~/.cursor/skills/`。

**其他工具**：把 `CLAUDE.md` 的内容贴进对应的指令文件（如 `AGENTS.md`）。

## 用法

让 Agent 优化性能时，这个技能会自动加载。也可以直接发快捷提示词，比如：

```
先在 bench/large.ts 上 profile 一下解析器，再告诉我 token 列表该不该改成结构体数组。
```

## 致谢

- 提示词来自 [Jarred Sumner](https://x.com/jarredsumner)，见[原帖](https://x.com/jarredsumner/status/2117037940031496252)。
- 仓库格式参考了 [forrestchang/andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills)。

## 许可证

[MIT](./LICENSE)
