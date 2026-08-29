# DeepSeek Harness（dsh）代码报告与使用指南

> 仓库：https://github.com/deepseek-ai/deepseek-harness
> 本地路径：`/Users/jyxc-dz-0100161/Desktop/DSH/deepseek-harness`
> 版本状态：0.1.2-alpha.1（开发者预览，会有破坏性变更）｜ MIT 协议

DeepSeek Harness（命令名 `dsh`）是 DeepSeek AI 开源的**智能体运行框架（agent harness）**。核心理念是 "everything-is-a-plugin"（一切皆插件），基于 Cordis 框架构建——包括模型适配器、工具注册表、会话日志、Agent 循环在内的所有能力都是插件，没有特权核心。

---

## 一、仓库整体结构

| 目录 | 内容 |
|---|---|
| `apps/` | 产品装配层：`apps/cli`（`dsh` 可执行入口，npm 包 `@deepseek-ai/dsh`）、`apps/web`（Vite+React 的 SPA 壳） |
| `packages/` | 约 50 组共 100+ 个子包（`packages/<组>/<包>`），全部是 Cordis 插件，是代码主体 |
| `python/` | Python SDK：`deepseek-harness-sdk`（stdio 上的 JSON-RPC 客户端 + 高层 turns API）、`deepseek-harness-runtime-bin`（打包 dsh 可执行文件的 wheel） |
| `native/` | 原生代码，目前主要是 `native/landlock-run`：Linux 上的 Landlock 沙箱自限制启动器（Rust/C） |
| `vendor/` | 内置的框架 fork，重命名为 `@deepseek-ai/*`：cordis、cosmokit、schemastery 及 cordis 插件（loader、hmr、timer 等），通过 pnpm overrides 锁定 |
| `website/` | 文档站构建器（deepseek-harness.github.io） |
| `docs/` | 架构文档、开发指南、用户指南、cookbook（均有中文对照版 `.zh.md`） |
| `scripts/` | 大量仓库工具：构建、门禁检查（run-gates）、代码生成器（gen-tool-catalog 等）、校验器、发布脚本 |
| `snapshots/` | 快照测试语料（acp、sdk、session、web） |
| 根目录 | pnpm workspace 配置、8 套 vitest 配置、双聚合 tsconfig（host/client）、tsdown 构建、lefthook 钩子 |

---

## 二、核心包详解（packages/）

每个包都是一个 Cordis 插件，向上下文 `ctx` 挂载一个服务键。

### 1. 会话与上下文（core 组）
- **`core/session`** — 追加式（append-only）`SessionEvent` 日志 + 内存存储，是模型上下文的**唯一事实来源**（"模型可见即可被记录"）。每轮对话通过 `deriveMessages()` 从事件日志推导出消息列表。
- **`core/system-prompt`** — 系统提示的分节（section）组装与工具 schema 汇总。
- **`core/tools`** — 带作用域的工具注册表和守卫式执行流水线：`tools/pre-execute → execute → post-execute`。工具用 `defineTool()`（`packages/core/tools/src/schema.ts:545`）定义：类型化参数 schema 编译为 JSON Schema，支持输出 schema、UI 呈现钩子、超时与并发安全标记。
- **`core/agent`** — `Agent` 接口、运行实例注册表、`agent/*` 事件。
- **`core/agent-loop`** — 默认驱动 `ReactLoopAgent`（`src/agent.ts`），工厂服务在 `src/index.ts:296`：
  ```ts
  export class AgentLoop extends Service implements AgentFactory {
    static inject = ['agents', 'sessions', 'llm', 'tools', 'systemPrompt']
    // create()/resume() 发布 ReactLoopAgent 句柄；卸载时效果自动回滚
  }
  export default AgentLoop
  ```

### 2. 模型接入（llm 组）
- **`llm/llm`** — LLM 消息/流式词汇表与适配器接缝（seam）。`LlmRuntime`（`src/index.ts:326`）提供 `ctx.llm.registerAdapter(providers, adapter)`；适配器实现 `stream(options): AsyncIterable<StreamChunk>`。
- **`llm/llm-deepseek`** — DeepSeek 官方 API 原生适配器（SSE、文件 API、图像 token、计价）。
- **`llm/llm-pi-ai`** — 基于 pi-ai 的多供应商适配器：Anthropic、OpenAI 兼容、Bedrock、Vertex、Azure、Codex OAuth 等目录化接入。
- 辅助包：`llm-retry`（重试策略）、`token-meter`（token 计量）、`deepseek-llm-api-extensions`。

### 3. 工具型能力包
| 组 | 说明 |
|---|---|
| `shell/` | Shell 接缝 `ctx.shell` + 后端 `bash-local` / `bash-sandbox` / `pwsh-local` / `pwsh-sandbox`，模型工具 `bash`、`bash-persistent`、`pwsh` 等 |
| `terminal/` | 持久 PTY 终端服务 `ctx.terminals`（`terminal-bash`、`tool-terminal`） |
| `lsp/` | 语言服务器协议接入（`lsp-stdio`、`tool-lsp`），代码诊断 |
| `web/` | `web` 搜索/抓取服务 + 模型工具 `web_search`/`web_fetch`；搜索后端支持 DeepSeek/Exa/Perplexity，抓取后端 `web-fetch-http` |
| `skill/` | 技能子系统（skill 服务、skill-filesystem、工具与徽章） |
| `plan/plan-mode` | 计划/todo 模式 |
| `compaction/` | 上下文压缩（basic、tool-result-pruner、`/compact` 命令） |
| `context/` | 模型可见上下文注入器：AGENTS.md 指令、文件引用、会话引用、时间、tmux 上下文 |
| `mcp/` | MCP（Model Context Protocol）客户端 |
| `subagent/`、`goal/`、`todo/`、`jobs/` | 子代理、目标跟踪、后台任务（`job_*` 工具） |
| `guard/` + `native/landlock-run` | 沙箱与审批策略 |
| `code-runtime/` | PTC "program-the-computer" 的 `run_code` 能力 |
| `preset/`、`bundle/` | 预设（agent-presets、persona）与 profile 装配包（base、web-app、headless、sdk-app、sdk-minimal、acp-app） |

### 4. Web UI（web / host / client 组）
- 前端：**React 18 + Vite**。`apps/web` 只是入口壳，功能在 `packages/client/ui-*`（ui-chat、ui-conversation、ui-session、ui-settings、ui-renderer 等），`packages/client/web` 为客户端组合，`connection` 为传输层。
- 后端：`packages/host/webserver` —— 普通 `node:http` 服务器，插件注册命名路由与升级路由，`frontend-static` 提供 SPA 静态回退。
- Host↔Client 通信：`/api` 桥 + **生成的 RPC 契约**——服务方法用 `@Remote`/`@RemoteScope` 注解，Host 构建期由 Typert（`packages/typert`，自研 TS 类型反射/网关代码生成）生成客户端类型与运行时，挂到客户端 `ctx.remote`。详见 `docs/api-gateway.md`。

---

## 三、插件架构（核心设计）

1. **插件即 Cordis 插件**：导出一个类（常 `extends Service`）或 `apply(ctx)` 函数；`static inject = [...]` 声明服务依赖；注册可逆，插件卸载时效果自动展开。
2. **组合而非硬编码**：启动时按"层树"装配。每个 bundle 附带 `cordis.patch.yml` 列出插件行（如 `- insert: [- id: llm, name: '@deepseek-ai/dsh-llm', config: ...]`，在各自 `package.json` 的 `dsh.bundle.patch` 中声明）；profile 声明 bundle 列表。加载由内置 `cordis-plugin-loader` 完成，`hmr` 支持热重载。
3. **分层 patch 合并**（`packages/boot/app-boot` + `cmdline`）：bundle 顺序 → profile 的 `cordis.patch.yml` → home 级 patch → 命令行 `--patch` 覆盖；同一行 id 后写胜出。
4. **三类事件作为扩展点**：持久化**会话事件**（`session/event`）、**agent 事件**（`agent/pre-step`、`agent/request`、`agent/turn-stopping` 等）、能力接缝上的事件（`fs/*`、`tools/*`、`telemetry/*`）；瀑布式事件必须 `next()` 委托。
5. **能力接缝（capability seams）三角色**：服务定义（接口）、提供者（实现）、消费者（通常是模型工具）。例如把 `ctx.fs`/`ctx.subprocess` 的提供者换成远程沙箱，Bash/PTY/LSP 工具随之整体迁移。

### Agent 循环（一轮对话的流水线）
一个 *step* = 一次模型请求 + 其工具调用；一个 *turn* = 若干 step。

```
turn/start → agent/pre-step → step/start → 追加用户消息
→ deriveMessages() → agent/request → llm/stream
→ assistant/chunk* → assistant/message
→ tool/call* → tools/pre-execute → tools/execute → tools/post-execute → tool/result
→ step/end → agent/turn-stopping → turn/end
```

实现位于 `packages/core/agent-loop/src/{agent.ts, index.ts, tool-calls.ts, runtime-context.ts}`。

### 代表性工具定义（`packages/shell/tool-bash/src/index.ts:242`）
```ts
ctx.tools.register(defineTool({
  name: 'bash',
  description: bashDescription(backgroundEnabled, escalationModes),
  parameters: {
    command: { type: 'string', required: true, description: 'The bash command to execute.' },
    // ...
  },
  output: { schema: { oneOf: [...] }, render(args, value) { /* UI 呈现 */ } },
  async execute(args, exec) { /* 走 ctx.shell 接缝 */ },
}))
```
同一插件里还注册了系统提示分节（`ctx.systemPrompt.section(...)`）。

---

## 四、CLI 入口与 profile 体系

`dsh` 二进制定义在 `apps/cli/package.json`（`bin → lib/bin.js`），源码 `apps/cli/src/bin.ts`，三种模式：

- **`dsh --profile <名>`** — 启动一个 profile，剩余参数传给 profile 的应用插件；
- **`dsh plugin`** — 管理 profile 的插件（转发给 pnpm 安装树外插件）；
- **`dsh --dump-config`** — 打印组合后的插件树（排障利器：`dsh --profile web --dump-config`）。

`dsh web` 是 `--profile web` 的硬编码别名。官方内置应用：

| profile | 用途 |
|---|---|
| `web` | Web UI（默认 `http://127.0.0.1:3080`） |
| `headless` | 一次性执行，不起服务器 |
| `sdk` / `sdk-minimal` | Python/TS SDK 子进程模式 |
| `acp` | Agent Client Protocol 服务器（编辑器自动化） |

常用启动旗标：`--profile`、`--patch <yml>`（可重复叠加）、`--no-open`（不自动开浏览器）。profile 存放在 Harness home，按 bundle 叠加（`dsh-base` 永远第一层）。

---

## 五、Python SDK 与原生组件

- `python/sdk`（`deepseek-harness-sdk`）：高层 turns API + stdio 上的换行分隔 JSON-RPC 客户端；总是以显式 Harness home 启动 `dsh --profile sdk`。`python/sdk-runtime` 把 dsh 可执行文件打包成 wheel。
- `native/landlock-run`：Linux 上的 "self-restrict-then-exec" Landlock 沙箱启动器，用于限制被孵化进程的文件系统权限，配合 `packages/guard` 的沙箱策略。

---

## 六、构建与测试体系

- **工具链**：pnpm workspace（Node ≥22.19 / ≥24，Corepack pnpm 11.7.0，Git 2.26+）；`minimumReleaseAge` 供应链策略；patch 过的 `node-pty`；lefthook 管理钩子。
- **构建顺序**：`tsc host 聚合 → tsdown host → tsc client 聚合 → tsdown client → build:web`。TypeScript 用**双聚合程序**（`tsconfig.host.json` / `tsconfig.client.json`），因为 Host 和 Client 以 declaration-merge 方式给 Cordis `Context` 挂不同的服务键。
- **主要脚本**：`build`、`typecheck`、`lint`（oxlint）、`duplication`（jscpd）、`knip`、`test`（vitest 单测）、`test:e2e`、`test:expected`、`test:snapshot`、`test:web[:perf|:stress]`、`mock:llm`（`packages/test-support/llm-mock-server` 提供 mock LLM 服务器）、`check:all`（`scripts/run-gates.ts` 门禁）。
- 从源码运行：`node --import tsx/esm apps/cli/src/bin.ts`（即根 `pnpm dsh`）。

---

# 使用指南

## 1. 最快上手（npm 直接跑）

```sh
npx @deepseek-ai/dsh web
```

启动 Web UI（默认 `http://127.0.0.1:3080` 并自动打开浏览器；SSH 场景只打印 URL）。加 `--no-open` 不开浏览器。

## 2. 从源码运行

```sh
cd deepseek-harness
pnpm install
pnpm run build
pnpm dsh web
```

要求：Node ≥22.19、Corepack 启用的 pnpm 11.7.0。

## 3. 配置模型与 API Key

- **推荐路径**：Web UI → **Settings → Models**，填入 DeepSeek API Key，选择模型和工作区目录即可开始任务。
- Key 是**只写**的，由 `packages/credentials` 存入 `$DSH_HOME/.credentials.yaml`，`$DSH_HOME/settings.yaml` 引用。
- 环境变量：`DEEPSEEK_API_KEY=sk-...`，可选 `DEEPSEEK_BASE_URL`（源码运行时支持根目录 `.env`，已被 gitignore）。
- **其他供应商**：内置目录支持 Anthropic、OpenAI 兼容、Bedrock、Vertex、Azure、Codex OAuth。自定义 OpenAI 兼容供应商在 `settings.yaml` 的 `llm-pi-ai.providers` 下写 YAML（baseURL、`api: openai-completions`、`apiKeyEnv`、模型列表、`input: [text, image]`）。
- Web 搜索/抓取后端可用 `DSH_WEB_SEARCH_PROVIDER` / `DSH_WEB_FETCH_PROVIDER` 切换。

## 4. 常用命令速查

| 命令 | 作用 |
|---|---|
| `dsh web` | 启动 Web UI |
| `dsh --profile headless ...` | 无服务器一次性执行 |
| `dsh --profile web --dump-config` | 打印组合后的插件树（排障） |
| `dsh plugin ...` | 安装/管理树外插件 |
| `dsh --profile web --patch my.yml` | 用 YAML 叠加覆盖配置 |

## 5. Python SDK

安装 `deepseek-harness-sdk`（`deepseek-harness-runtime-bin` 自带运行时），通过高层 turns API 或 JSON-RPC 驱动 dsh 子进程（`--profile sdk`）。详见 `docs/user/guide/python-sdk.md`。

## 6. 开发者指南

- 开发前读 `docs/development.md` 与 `docs/architecture.md`；面向 AI 代理的规范在 `AGENTS.md`。
- 入门 Cordis：`docs/cordis-primer.md` + 教程。
- **"新行为放哪里"速查**（来自架构文档）：加模型供应商 → `ctx.llm`；加工具 → `ctx.tools`；shell 行为 → `ctx.shell`；命令 → `ctx.commands`；后台任务 → `ctx.jobs`；文件策略 → `ctx.fs`；沙箱 → `ctx.sandbox`；UI → `ctx.agents` + `session/event`。
- Cookbook（`docs/cookbook/`）：新增包、新增工具、新增 LLM 适配器、新增设置卡片的完整示例。
- 生成的参考目录：`docs/tool-catalog.md`（工具目录）、`docs/config-catalog.md`（配置目录）。

## 7. 安全须知

- 项目处于开发者预览阶段，**会有破坏性变更**；运行前请先阅读 `SAFETY.md`。
- Web 服务器默认只绑 `127.0.0.1`（可改 `0.0.0.0`），**没有内置 TLS/鉴权**——不要暴露到不受信网络。
- 代理会执行模型要求的命令，生产使用建议开启沙箱/审批（`packages/guard`，Linux 上配合 Landlock）。

---

*报告基于 2026-08-29 的 main 分支浅克隆（约 8953 个文件）与 `docs/` 文档梳理生成。*
