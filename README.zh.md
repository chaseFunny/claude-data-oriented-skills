# claude-data-oriented-skills

> **格局要大：快 1% 不够，要快 10 倍。**
>
> "You need to be thinking bigger. 1% faster is not enough. This should be 10x faster."（[Jarred Sumner](https://x.com/jarredsumner/status/2117046610404442215)）

一个给编程 Agent 用的性能优化技能，核心方法是面向数据的设计（data-oriented design）。灵感来自 [Jarred Sumner 分享的提示词](https://x.com/jarredsumner/status/2117037940031496252)，那是他用 Claude 做性能优化时常说的几句话。

[English](./README.md) | 简体中文

> **非官方社区项目**，与 Anthropic、Jarred Sumner 及 Oven（Bun）均无关联，也未获其认可。

## 思路

Jarred [原帖](https://x.com/jarredsumner/status/2117037940031496252)中的四句提示词：

> how do we rewrite this to better leverage data-oriented design principles

> can you replace those hash tables with a flat array? can we make the pointers an index into the array? can we pack the most commonly used data into upper bits of the index?

> caching is cope for a slow design. do not add caches and delete all existing caches. instead, redesign it to be faster

> should this be a structure of arrays instead of an array of structures?

他在回复里又补充了位集、带内联容量的栈上列表和 arena。这些提示词单独丢给 Agent，它仍可能凭感觉乱改，或者把真正有用的缓存也删掉。这个技能结合评论区的讨论，补上了先测量再动手的流程、怎么给 Agent 下指令、每条原则的适用和不适用场景，以及明确的边界。

## 内容

| 部分 | 告诉 Agent 什么 |
|---|---|
| **原理：缓存行** | 每条原则都是为了少加载 64 字节的缓存行，并让每一行里有用的数据更多。 |
| **0. 流程** | 建基线 → 先列出热点和每次迭代的分配次数 → 改数据布局 → 跑测试 → 再测一遍。没有数字就不算优化。 |
| **怎么下指令** | 说清访问模式和具体布局，别只说“让它更快”；边做边抽查 diff。 |
| **1. 面向数据的设计** | 按使用数据的那个循环来组织数据。 |
| **2. 扁平数组和索引** | 用数组代替哈希表，用 `u32` 索引代替指针：底层数组能整体搬移，序列化几乎零成本。 |
| **3. 高位打包** | 把最常用的标记放进索引的高位。 |
| **4. SoA 还是 AoS** | 按访问模式选结构体数组或数组结构体。 |
| **5. 位集** | 每个标记占 1 位，而不是 1 个字节。 |
| **6. Arena 加栈上列表** | 长生命周期的数据用 arena，临时列表用带内联容量的栈上列表。 |
| **7. 缓存是逃避** | 把慢路径本身改快，而不是用缓存挡住它。 |
| **边界** | 网络、IO、数据库、CDN 缓存和纯函数的记忆化默认保留，除非包含重复查询的基准测试证明重新设计更快；禁止批量删缓存；测试必须全部通过。 |
| **快捷提示词** | Jarred 原帖和回复里的提示词原文照录，外加一句“先测量”的提示词。 |

每条原则都写了什么时候用、什么时候别用，并配有 TypeScript、Rust、Zig 或 C 的改写前后小例子。

## 安装

**Claude Code 插件**

```
/plugin marketplace add chaseFunny/claude-data-oriented-skills
/plugin install claude-data-oriented-skills@claude-data-oriented-skills
```

**Claude Code 个人技能**

```bash
git clone https://github.com/chaseFunny/claude-data-oriented-skills
mkdir -p ~/.claude/skills
cp -r claude-data-oriented-skills/skills/data-oriented-perf ~/.claude/skills/
```

**按项目追加到 CLAUDE.md**

```bash
curl https://raw.githubusercontent.com/chaseFunny/claude-data-oriented-skills/main/CLAUDE.md >> CLAUDE.md
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

- 提示词来自 [Jarred Sumner](https://x.com/jarredsumner)：[原帖](https://x.com/jarredsumner/status/2117037940031496252)、[回复](https://x.com/jarredsumner/status/2117039564221256052)、[补充](https://x.com/jarredsumner/status/2117046610404442215)。
- 仓库格式参考了 [forrestchang/andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills)。

## 感谢社区的想法

- [@bradleymeck](https://x.com/bradleymeck/status/2117038669202620791) 和 [Jarred](https://x.com/jarredsumner/status/2117039149379412270)：先用 arena，临时数据放栈上
- [@acdlite](https://x.com/acdlite/status/2117094796426653891)：栈和位集快，主要靠缓存局部性
- [@TinyRouterHQ](https://x.com/TinyRouterHQ/status/2117094059835589104)、[@jimmyyangrz](https://x.com/jimmyyangrz/status/2117061430704996656)：说清访问模式和布局，而不是症状
- [@amdmszOvO](https://x.com/amdmszOvO/status/2117076542627140039)：动手前先列出热点和每次迭代的分配次数
- [Jarred](https://x.com/jarredsumner/status/2117079330035728578)：边做边抽查 diff，像反复暂停调试器
- [@zhao_spenc](https://x.com/zhao_spenc/status/2117073400938270930)、[@perperp_er](https://x.com/perperp_er/status/2117063323820900649)：索引在数组重新分配后依然有效，序列化几乎零成本
- [@dhillonpost](https://x.com/dhillonpost/status/2117088944831770728)、[@John_zhong324](https://x.com/John_zhong324/status/2117070275070288176)：删缓存前用包含重复查询的负载测试，权衡失效成本和重算成本
- [@akashc777](https://x.com/akashc777)：去掉 Redis，自托管的客户就少了一项运维负担
- [@kimindiehacker](https://x.com/kimindiehacker/status/2117081654653255937)：面向数据的设计可能影响可读性，只用在热点上

## 许可证

[MIT](./LICENSE)
