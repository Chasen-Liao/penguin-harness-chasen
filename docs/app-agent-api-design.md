# App（Workbench）向 Agent 开放接口的设计构想

记录日期：2026-11-24  
状态：草案（未提交）

## 1. 背景与现状

近期提交已经完成了 Agent 对 App（Workbench）自省与调用的基础链路：

1. **会话级最小权限凭据**：`packages/server/src/auth/session-token.ts` 为每个运行中的 Session 签发 `psess_` 前缀的 Bearer Token，限定在当前 Project 的成员级访问权限内。
2. **App 上下文归因**：Workbench 打开的 Session（及其派生的子代理）自动注入 `PENGUIN_APP_ID=<ownerAgentId>/<workbenchId>`，Session 记录也持久化了 `appId`。
3. **CLI 操控与直连入口**：`packages/server/src/cli/commands/app.ts` 提供了 `penguin app ls/info/history/reload/call`，其中 `penguin app call` 可直接请求 `/api/projects/:p/agents/:a/workbenches/:id/api/*` 并交给 `WorkbenchMain.handle(req)` 处理。

**当前缺口**：现有内置 App（如 `slides`、`support`、`images`）的 `/api/*` 均按「前端页面单向调用 Handler → Handler 调 `host.run()` 唤醒 Agent」设计，尚未针对 Agent 反向调用做协议约定、身份区分与错误处理适配。

---

## 2. 核心定位：从「UI 外壳」升级为「人机共享的有状态领域服务」

让 App 适配并向 Agent 开放接口后，Workbench 同时承担两面职责：

- **面向人类**：提供可视化 Tab 页面，响应交互并通过 SSE/AMSP 实时展示状态。
- **面向 Agent**：通过 `penguin app call` 提供受控的业务读写、状态查询、UI 联动与常驻资源代理，且**完全通过 CLI 按需调用，不向全局 Tool Schema 或系统提示词注入动态定义，零 Prompt Cache 损耗**。

### 可解锁的 4 类能力

1. **安全读写 `state.json` 与业务状态机**
   - 现有规范禁止 Agent 用文件工具直接读写 `state.json`（无版本历史，且会与 Handler 内存队列竞态导致丢更新）。
   - App 开放业务接口后（例如 `support` 开放工单筛选、状态流转、批量归档接口），Agent 可复用 Handler 内部的串行锁与校验逻辑安全变更业务状态。

2. **细粒度原子修改、Schema 校验与乐观锁**
   - 替代「Agent 直接裸写整份大 JSON 文件」的模式。
   - 以 `slides` 为例：目前只有前端走 `PUT /decks/:id` 享受 `base` revision 冲突检测与历史备份，Agent 直接改磁盘文件既耗费 Token，又容易写坏结构或覆盖用户刚做的编辑。开放如 `PATCH /decks/:id/slides/:slideId` 等语义化接口后，校验、历史快照与前端 SSE 通知可一次完成。

3. **双向实时 UI 协同（Co-pilot 联动）**
   - **Agent → UI 推送**：Handler 常驻内存并持有前端 SSE 流（如 `GET /events`）。Agent 在长任务途中调用 `/ui/focus`、`/ui/preview` 等接口即可驱动前端切页、高亮元素或展示阶段结果。
   - **UI → Agent 拉取**：前端将当前打开的文档、选区、过滤条件同步在 Handler 内存态中，Agent 可随时调用 `GET /context` 读取最新现场，无需仅依赖提问瞬间的静态 Prompt 拼接。

4. **跨 Agent 复用常驻重资源**
   - Agent 的单次 Shell 命令是短生命周期子进程，而 Workbench Handler 常驻服务端进程，且具备 `host.dataPaths(level)` 与 `host.socketPath(name, level)`。
   - App 可在后台维护本地索引、数据库连接池或常驻守护进程，通过 `penguin app call <agentId>/<workbenchId> /...` 供同一 Project 下的任意 Agent 按需调用。

---

## 3. 需要补齐的 3 个关键机制

### 3.1 接口自发现约定（按需加载，不污染上下文）

Agent 知道 `penguin app call` 语法，但不知道目标 App 暴露了哪些路由。建议采用与 Skill 类似的渐进式发现机制：

- **统一自描述端点**：约定支持 Agent 调用的 App 实现 `GET /_agent`，返回结构化接口说明：
  ```json
  {
    "summary": "Manage Bento slide decks and push live editor focus.",
    "endpoints": [
      {
        "method": "GET",
        "path": "/decks",
        "summary": "List all decks with title, slide count and mtime"
      },
      {
        "method": "PATCH",
        "path": "/decks/:id/slides/:slideId",
        "summary": "Atomically update one slide with schema validation and history backup",
        "body": { "base": "<rev>", "slide": {} }
      }
    ]
  }
  ```
- **轻量元数据提示（可选）**：在 `package.json#penguin` 中允许声明简短标识（如 `"agentApi": true` 或一句话说明），在 `penguin app ls` / `penguin app info` 中展示，引导 Agent 仅在需要时调用 `penguin app call /_agent`。

### 3.2 调用者身份透传与防递归死循环（安全边界）

当前 `packages/server/src/workbenches/routes.ts` 剥离了 `cookie` 与 `authorization` 请求头，`WorkbenchRequest`（`library/workbench/src/workbench.ts`）不包含任何调用方身份信息，Handler 无法区分请求来自「浏览器里的人」还是「某个 Session 中的 Agent」。

这会带来两类风险：
- **递归任务死循环**：`slides` 的 `POST /ask`、`support` 的 `POST /tickets`、`images` 的 `POST /jobs` 内部都会调用 `host.createSession()` + `host.run()`。若 Agent 误调此类端点，会产生 `Agent → App → 新 Session Agent` 的递归调用链并持续消耗额度。
- **人机权限混同**：部分破坏性或外发类操作可能只允许人类在 UI 触发，不应允许 Agent 调用。

**改进方案**：
- 在 `routes.ts` 中复用 `parseSessionPrincipal(c.var.user.userId)`，向 `WorkbenchRequest` 注入可信的 `caller` 字段：
  ```ts
  export type WorkbenchCaller =
    | { readonly kind: "human"; readonly userId: string }
    | { readonly kind: "session"; readonly sessionId: string }
    | { readonly kind: "local_token" };
  ```
  （若为兼容旧 `.harness/plugin.d.ts` 也可同步注入服务端清洗后的只读头 `x-penguin-caller` / `x-penguin-session-id`。）
- App Handler 中所有会调用 `host.createSession()` / `host.run()` 派生新 Agent 任务的路由，默认拒绝 `caller.kind === "session"`（返回 `403` 说明原因），除非显式设计为受控编排。

### 3.3 `penguin app call` 的错误透传与大负载输入

当前 CLI 实现存在两个影响 Agent 实战调用的细节问题：

1. **非 2xx 自定义错误体丢失**：
   - `packages/server/src/cli/commands/app.ts` 使用 `client.raw()` 发送请求；当状态码非 2xx 时，`library/cli/src/client.ts` 的 `toError()` 只提取平台标准格式 `{ error: { code, message } }`。
   - 但 Workbench Handler 通常返回自定义 JSON（例如 `slides` 返回 `404 { error: "no such deck" }`，或冲突时返回 `409 { rev, text }`）。经 `toError()` 转换后，原始响应体被吞掉，只打印笼统的 `HTTP 409 (http_error)`，导致 Agent 拿不到错误原因与最新 `rev`。
   - **修复方向**：`penguin app call` 捕获非 2xx 响应时保持 `exitCode = 1`，同时将 Handler 返回的 JSON/文本响应体原样输出，便于 Agent 自纠重试。
2. **大 JSON 参数输入不便**：
   - 目前 `-d, --data <json>` 只能通过命令行参数传入，跨平台 Shell（尤其 `pwsh` 与 `sh`）转义长文本极易出错。
   - **修复方向**：支持 `-d @<file>` 从文件读取 JSON，或 `-d -` 从标准输入读取；必要时补充查询参数透传与 SSE 流式读取选项。

---

## 4. 建议演进步骤

1. **阶段一：完善 CLI 与网关底座**
   - 增强 `penguin app call`：完整输出非 2xx 响应体，支持 `-d @file` / `-d -`。
   - 在 `WorkbenchRequest`（及网关层）补充可信 `caller` 身份透传，补齐单测。
2. **阶段二：改造 1～2 个内置 App 样板**
   - 在 `slides` 和 `support` 中为 `POST /ask`、`POST /tickets` 增加 `caller.kind === "session"` 防递归拦截。
   - 实现 `GET /_agent` 自描述路由，并补齐面向 Agent 的原子读写与 UI 联动接口。
3. **阶段三：沉淀到开发与编排 Skill**
   - 更新 `penguin-plugin` 与 `penguin-orchestration` Skill：将 `GET /_agent` 自描述约定、`caller` 防递归守卫以及人机双端 API 设计定为新建 Workbench 的标准范式。
