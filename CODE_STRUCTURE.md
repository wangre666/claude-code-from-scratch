# 代码结构梳理

## TypeScript 版本代码位置

TypeScript 版本的源码在仓库根目录的 `src/` 目录下：

```text
src/
├── cli.ts
├── agent.ts
├── tools.ts
├── prompt.ts
├── session.ts
├── ui.ts
├── memory.ts
├── skills.ts
├── subagent.ts
├── mcp.ts
└── frontmatter.ts
```

相关配置文件：

- `package.json`：定义 npm 脚本和 CLI 入口，`mini-claude` 指向编译后的 `dist/cli.js`。
- `tsconfig.json`：指定 `rootDir` 为 `src`，编译输出到 `dist`。
- `package-lock.json`：锁定 Node 依赖版本。

常用命令：

```bash
npm install
npm run build
npm start
```

## 顶层目录

```text
.
├── src/                 # TypeScript 版本核心实现
├── python/              # Python 版本实现
├── test/                # 手动测试资源、测试脚本、测试技能和 MCP server
├── assets/              # README 和教程使用的图片资源
├── README.md            # 中文项目说明
├── README_EN.md         # 英文项目说明
├── CLAUDE.md            # 项目级 Agent 指令
├── package.json         # TypeScript 项目元信息和 npm 脚本
└── tsconfig.json        # TypeScript 编译配置
```

## TypeScript 核心模块

| 文件 | 职责 |
| --- | --- |
| `src/cli.ts` | 命令行入口。解析参数，选择 API 后端，启动一次性 prompt 或交互式 REPL，处理 `/clear`、`/plan`、`/cost`、`/compact`、`/memory`、`/skills` 等命令。 |
| `src/agent.ts` | 核心 Agent Loop。负责调用 Anthropic/OpenAI 流式接口、管理消息历史、执行工具、处理并行工具调用、预算限制、上下文压缩、Plan Mode、子 Agent、MCP 工具和会话自动保存。 |
| `src/tools.ts` | 内置工具系统。定义工具 schema，并实现 `read_file`、`write_file`、`edit_file`、`list_files`、`grep_search`、`run_shell`、`web_fetch`、`skill`、`agent`、Plan Mode 工具和 `tool_search`。也包含权限模式、危险命令检测、settings allow/deny 规则和结果截断。 |
| `src/prompt.ts` | System Prompt 构建。加载 `CLAUDE.md`、`.claude/rules/*.md`、Git 上下文、记忆、技能、子 Agent 描述和 deferred tools 信息，并替换模板变量。 |
| `src/ui.ts` | 终端展示层。负责欢迎语、用户提示符、工具调用/结果展示、错误、费用、重试、spinner、Plan 审批和子 Agent 状态输出。 |
| `src/session.ts` | 会话持久化。保存、加载、列出会话，并支持恢复最近一次会话。 |
| `src/memory.ts` | 记忆系统。以文件形式保存 user、feedback、project、reference 四类记忆，维护 `MEMORY.md` 索引，并通过 side query 选择与当前请求相关的记忆注入上下文。 |
| `src/skills.ts` | 技能系统。发现用户级和项目级 `.claude/skills/*/SKILL.md`，解析 frontmatter，支持 inline 和 fork 两种执行上下文。 |
| `src/subagent.ts` | 子 Agent 配置。提供内置 `explore`、`plan`、`general` 三类 Agent，并支持从 `.claude/agents/*.md` 加载自定义 Agent。 |
| `src/mcp.ts` | MCP 客户端。读取 `.claude/settings.json`、`~/.claude/settings.json` 或 `.mcp.json`，通过 stdio JSON-RPC 连接 MCP server，发现并转发 MCP 工具调用。 |
| `src/frontmatter.ts` | 共享 frontmatter 解析和格式化工具，被 memory、skills、subagent 复用。 |

## 运行入口和主流程

TypeScript 版本的入口是 `src/cli.ts`。构建后生成 `dist/cli.js`，`package.json` 中的 bin 配置把它暴露为 `mini-claude`。

主流程：

1. `cli.ts` 解析命令行参数和环境变量。
2. 根据 `ANTHROPIC_API_KEY`、`ANTHROPIC_BASE_URL`、`OPENAI_API_KEY`、`OPENAI_BASE_URL` 决定使用 Anthropic 格式还是 OpenAI 兼容格式。
3. 创建 `Agent` 实例。
4. 如果带 `--resume`，从 `session.ts` 恢复最近会话。
5. 如果命令行传入 prompt，执行一次性 `agent.chat(prompt)`。
6. 否则进入 REPL，循环读取用户输入并调用 `agent.chat(input)`。

## Agent Loop 内部结构

`src/agent.ts` 是项目的核心。它大致分成几层：

- 模型层：封装 Anthropic 和 OpenAI 两种后端，并维护各自的 message history。
- 工具层：把 `tools.ts` 的工具定义传给模型，收到 tool call 后执行工具。
- 权限层：调用 `checkPermission` 判断工具是否允许执行，必要时触发确认。
- 上下文层：维护 token 统计，执行 budget 截断、stale snip、microcompact、auto compact 和大结果落盘。
- 记忆层：每轮用户输入前启动 memory prefetch，把相关记忆注入本轮上下文。
- Plan Mode：进入只读规划状态，把计划写入临时 plan 文件，并在退出时触发用户审批。
- Sub-agent：通过 `agent` 工具创建隔离上下文的子 Agent，完成后把结果返回主 Agent。
- MCP：初始化外部 MCP server，把 MCP tool 转成普通工具供模型调用。

## 工具系统

内置工具定义集中在 `src/tools.ts` 的 `toolDefinitions` 中。

主要工具：

- `read_file`：读取文件并返回带行号内容。
- `write_file`：创建或覆盖文件。
- `edit_file`：按精确字符串替换编辑文件，包含引号归一化、唯一性检查、mtime 防护和 diff 输出。
- `list_files`：按 glob 列出文件。
- `grep_search`：搜索文件内容。
- `run_shell`：执行 shell 命令。
- `web_fetch`：抓取 URL 文本内容。
- `skill`：调用已注册技能。
- `enter_plan_mode` / `exit_plan_mode`：进入和退出 Plan Mode。
- `agent`：启动子 Agent。
- `tool_search`：延迟加载工具 schema。

权限模式：

- `default`：读工具自动允许，写文件和危险命令需要确认。
- `plan`：只读规划模式。
- `acceptEdits`：自动允许文件编辑，危险 shell 仍需确认。
- `bypassPermissions`：跳过确认。
- `dontAsk`：需要确认的操作自动拒绝，适合 CI。

## Prompt、记忆、技能和子 Agent 的关系

`prompt.ts` 构建基础 System Prompt 时，会把以下信息拼接进去：

- 当前工作目录、日期、平台和 shell。
- Git 分支、最近提交和工作区状态。
- 从当前目录向上查找的 `CLAUDE.md`。
- `.claude/rules/*.md` 中的项目规则。
- `memory.ts` 生成的记忆说明。
- `skills.ts` 发现的技能说明。
- `subagent.ts` 发现的自定义 Agent 类型。
- `tools.ts` 中 deferred tools 的提示。

这意味着技能、记忆、子 Agent 和工具不是独立入口，而是都会进入 Agent 的系统上下文或工具列表，由 `agent.ts` 在对话过程中统一调度。

## TypeScript 与 Python 版本

本仓库同时有 TypeScript 和 Python 两套实现：

- TypeScript：`src/`
- Python：`python/mini_claude/`

两者模块划分基本对应，例如：

| TypeScript | Python |
| --- | --- |
| `src/agent.ts` | `python/mini_claude/agent.py` |
| `src/tools.ts` | `python/mini_claude/tools.py` |
| `src/cli.ts` | `python/mini_claude/__main__.py` |
| `src/session.ts` | `python/mini_claude/session.py` |
| `src/memory.ts` | `python/mini_claude/memory.py` |
| `src/skills.ts` | `python/mini_claude/skills.py` |
| `src/subagent.ts` | `python/mini_claude/subagent.py` |
| `src/mcp.ts` | `python/mini_claude/mcp_client.py` |
| `src/prompt.ts` | `python/mini_claude/prompt.py` |
| `src/ui.ts` | `python/mini_claude/ui.py` |
| `src/frontmatter.ts` | `python/mini_claude/frontmatter.py` |

如果只看 TypeScript 版本，可以优先按这个顺序阅读：

1. `src/cli.ts`：了解程序如何启动。
2. `src/agent.ts`：理解 Agent Loop。
3. `src/tools.ts`：理解工具能力和权限控制。
4. `src/prompt.ts`：理解系统提示词如何组装。
5. `src/memory.ts`、`src/skills.ts`、`src/subagent.ts`、`src/mcp.ts`：理解进阶能力。
6. `src/session.ts`、`src/ui.ts`、`src/frontmatter.ts`：理解支撑模块。
