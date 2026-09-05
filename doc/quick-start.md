# OpenCode 项目代码快速上手指南

> 本文档面向初次接触本仓库的开发者，帮助你理解项目是什么、代码在哪里、如何跑起来、以及如何高效阅读和修改代码。

---

## 1. 项目是什么

**OpenCode** 是一个开源的 AI 编程助手（AI coding agent）：

- 终端 TUI 形态（也提供 Web / Desktop 形态），让 AI 读写代码、执行命令、回答问题
- 支持 75+ 家 LLM 提供商（Anthropic、OpenAI、Google、本地模型等）
- 支持 Agent（build / plan / 自定义）、MCP 工具、LSP、插件系统

- 官网：https://opencode.ai
- 仓库：https://github.com/anomalyco/opencode
- 默认分支：**`dev`**（不是 main）

---

## 2. 技术栈速览

| 技术 | 用途 |
| --- | --- |
| **Bun** (≥1.3) | 运行时、包管理、测试、构建（用 `bun` 代替 node/npm） |
| **TypeScript** | 全仓库语言，ESM（`"type": "module"`） |
| **Effect v4** (effect-smol) | 核心业务逻辑的 Effect 风格框架，`packages/core`、`packages/server` 大量使用 |
| **Drizzle ORM + SQLite** | 会话/消息等持久化存储 |
| **Hono + HttpApi** | HTTP API 服务器与类型安全接口定义 |
| **SolidJS** | TUI（基于 opentui）和 Web/Desktop 前端 |
| **TailwindCSS 4** | 前端样式 |
| **Turborepo** | monorepo 任务编排（typecheck 等） |

> ⚠️ Effect v4 是 beta 版本，如果没写过 Effect，先看 `.opencode/skills/effect/SKILL.md`（仓库内自带说明）再动 core/server 代码。

---

## 3. 目录结构：先搞清楚包的分层

这是一个 Bun workspace monorepo，所有代码在 `packages/` 下。**理解依赖方向是读懂代码的第一步**：

```
schema ──► core ──► server ──► sdk-next (嵌入式)
  │           │          │
  └──► protocol ┘        │
                         ▼
        client（网络客户端，可依赖 schema/protocol，禁止依赖 core/server）
```

AGENTS.md 中的硬性规则：**Schema → Core/Protocol → Server → Client，运行时依赖只能朝这个方向走**。

### 3.1 必须认识的包

| 包 | 说明 | 建议阅读顺序 |
| --- | --- | --- |
| `packages/opencode` | ⭐ 主包：业务逻辑 + CLI + API server + TUI | 从这里开始 |
| `packages/core` | 核心领域逻辑（Session、工具、权限、存储等），Effect 风格 | 第二步 |
| `packages/schema` | 共享领域模型（Schema.Struct 定义），最轻量的叶子包 | 需要时查 |
| `packages/protocol` | HTTP API 的路径/载荷/信封/错误组装 | 需要时查 |
| `packages/server` | HTTP API 服务器实现（HttpApi 为权威契约） | 需要时查 |
| `packages/client` | 生成的客户端 SDK（Promise + Effect 两套） | 改 API 后重新生成 |
| `packages/sdk` (js) | 旧版 JS SDK（改 HttpApi 后需 `bun run generate`） | 尽量别动 |
| `packages/tui` | 终端 UI（SolidJS + opentui） | UI 改动时 |
| `packages/app` | 共享 Web UI（SolidJS），desktop 也用它 | UI 改动时 |
| `packages/desktop` | Electron 桌面应用，包装 app | 桌面相关 |
| `packages/console` | 云端控制台（web 控制台 + 后端 worker） | 云服务相关 |
| `packages/plugin` | `@opencode-ai/plugin` 插件 API 源码 | 插件开发 |
| `packages/llm` | LLM 调用封装 | 模型调用相关 |
| `packages/function` / `packages/identity` / `packages/enterprise` | 云函数、认证、企业版 | 云服务相关 |

### 3.2 主包内的重要目录（`packages/opencode/src`）

```
session/     会话与消息循环（V1 loop / V2 runner）—— 最核心
agent/       Agent 定义与切换
tool/        内置工具（bash、edit、read、grep...）
provider/    模型提供商解析与鉴权
config/      opencode.json 配置解析
server/      HTTP server（基于 Hono）
cli/         CLI 命令与 TUI 入口（cli/cmd/tui/ 是 TUI 代码）
permission/  权限系统
mcp/         MCP 服务器接入
lsp/         语言服务协议
plugin/      插件加载
storage/     存储层（SQLite，#db 条件导入区分 bun/node）
bus/         事件总线
```

`packages/core/src` 里同理有 `session/`、`system-context/`（系统上下文组装）、`database/`、`filesystem/` 等，是 V2 架构的新家。

---

## 4. 环境搭建与运行

### 4.1 环境要求

- **Bun 1.3+**（必需）：https://bun.sh
- Windows/macOS/Linux 均可开发
- （可选）`gh` CLI 用于 GitHub 相关任务

### 4.2 安装依赖并启动

```bash
bun install          # 安装依赖（postinstall 会修复 node-pty）
bun dev              # 启动 OpenCode TUI（默认在 packages/opencode 目录运行）
```

常用 `bun dev` 用法（等价于生产环境的 `opencode` 命令）：

```bash
bun dev .            # 在当前仓库根目录启动 TUI
bun dev <目录>        # 在指定目录启动 TUI
bun dev --help       # 查看所有命令
bun dev serve        # 启动 headless API 服务器（默认端口 4096）
bun dev serve --port 8080
bun dev web          # 启动服务器 + 打开 Web 界面
```

其他形态：

```bash
bun run dev:desktop      # 桌面应用（Electron）
bun run dev:web          # Web 前端（需先 bun dev serve）
bun run dev:console      # 控制台
bun run dev:storybook    # 组件故事书
```

### 4.3 构建独立可执行文件

```bash
./packages/opencode/script/build.ts --single
# 产物：packages/opencode/dist/opencode-<platform>/bin/opencode
```

---

## 5. 验证你的改动

```bash
# 在仓库根目录
bun run lint                 # oxlint

# 类型检查：必须进入具体包目录，不要从根目录直接 tsc
cd packages/opencode && bun typecheck
cd packages/core && bun typecheck

# 测试：不要从根目录跑（根目录会直接报错退出）
cd packages/opencode && bun test
cd packages/opencode && bun test test/xxx.test.ts   # 跑单个文件
```

修改了公开的 `HttpApi`（protocol/server）之后：

```bash
cd packages/client && bun run generate   # 重新生成客户端，禁止手改 src/generated
```

---

## 6. 推荐的代码阅读路线

### 第一阶段：跑起来，建立直觉（半天）

1. `bun dev .` 在本仓库里启动 TUI，随便聊几句，观察它如何读文件、跑命令
2. 打开 `packages/opencode/src/index.ts`，看 CLI 入口
3. 看 `packages/opencode/src/cli/`，了解命令分发；`cli/cmd/tui/` 是 TUI 界面

### 第二阶段：理解一次对话的生命周期（1-2 天）

跟着一次用户提问走通整个链路：

```
用户输入
  → session 会话循环（packages/opencode/src/session/，V2 在 packages/core/src/session/）
  → provider 解析模型配置（packages/opencode/src/provider/）
  → LLM 请求（packages/llm）
  → 模型返回工具调用
  → 工具执行（packages/opencode/src/tool/，权限检查在 permission/）
  → 工具结果回传模型
  → 循环直到模型给出最终回答
  → 消息持久化（storage/ / core/src/database/）
```

关键概念对照：

| 概念 | 位置 |
| --- | --- |
| Session（会话） | `packages/opencode/src/session/`、`packages/core/src/session/` |
| Agent（build/plan/自定义） | `packages/opencode/src/agent/` |
| Tool（工具） | `packages/opencode/src/tool/` |
| Provider（模型提供商） | `packages/opencode/src/provider/` |
| 系统上下文（System Context） | `packages/core/src/system-context/`，术语见 `CONTEXT.md` |
| 权限（Permission） | `packages/opencode/src/permission/` |
| 配置（opencode.json） | `packages/opencode/src/config/` |

### 第三阶段：按你要做的改动选专题

| 想做的事 | 重点看 |
| --- | --- |
| 加/改内置工具 | `packages/opencode/src/tool/`，仿照现有工具写 |
| 加新模型提供商 | 通常无需改代码，先给 models.dev 提 PR；特殊适配在 `packages/llm` |
| 改 TUI 界面 | `packages/opencode/src/cli/cmd/tui/` + `packages/tui` |
| 改 Web/桌面 UI | `packages/app`（SolidJS）、`packages/desktop` |
| 改 HTTP API | `packages/protocol` 定义 + `packages/server` 实现，然后 `packages/client` 重新 generate |
| 写插件系统相关 | `packages/plugin` |
| 改持久化/数据库 | `packages/core/src/database/`、`packages/effect-drizzle-sqlite` |

---

## 7. 本仓库的开发约定（摘自 AGENTS.md，改代码前必读）

- **风格**：单函数优先，不为单次使用提前抽 helper；少用 `let`，多用 `const` 与三元/早返回；避免 `else`；避免 `try/catch`；不用 `any`
- **导入**：禁止 alias 导入（`import { a as b }`）和 star 导入（`import * as X`）；重模块用动态 `import()` 并放在真正需要的分支里
- **注释**：不加多余注释；只在"不明显的约束/意外行为"处加
- **Effect**：生成器里先把 service bind 到具名变量再调用，不要 `yield* (yield* Foo.Service).bar()`
- **Drizzle schema**：字段用 snake_case，不写字符串列名映射
- **测试**：避免 mock；不要从仓库根目录跑测试
- **类型检查**：进包目录跑 `bun typecheck`，别直接 `tsc`
- **提交**：conventional commit（`feat(core): ...`、`fix(tui): ...`）；默认分支是 `dev`

---

## 8. 常用资料索引

| 资料 | 位置 |
| --- | --- |
| 贡献指南（含各平台构建细节） | `CONTRIBUTING.md` |
| V2 会话核心架构与术语表 | `CONTEXT.md`（**强烈推荐通读一遍**） |
| Agent 仓库行为约定 | `AGENTS.md` |
| Effect 使用说明 | `.opencode/skills/effect/SKILL.md` |
| 用户文档（配置/agent/插件等） | https://opencode.ai/docs 与 `packages/docs` |
| 新提供商模型数据 | https://github.com/anomalyco/models.dev |

---

## 9. 十分钟速查卡

```bash
bun install                          # 装依赖
bun dev .                            # 跑 TUI
bun dev serve --port 4096            # 跑 API server
bun run lint                         # lint
cd packages/opencode && bun typecheck  # 类型检查（在包目录里）
cd packages/opencode && bun test     # 测试（在包目录里）
```

记住三件事：

1. **主包是 `packages/opencode`**，核心领域逻辑在 `packages/core`，API 契约在 `packages/protocol` + `packages/server`
2. **依赖方向**：Schema → Core/Protocol → Server，Client 不许依赖 Core/Server
3. **类型检查和测试都要在包目录里跑，根目录跑测试会被禁止**
