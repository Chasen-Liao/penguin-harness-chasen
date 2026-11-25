# App 自省与 Session 凭据收窄

> 对应 PR：[Prism-Shadow/penguin-harness#1791](https://github.com/Prism-Shadow/penguin-harness/pull/1791)
> 分支：`feat/cli-app-self-inspection`
> 状态：设计说明，供合并前讨论
> 日期：2026-11-24
>
> 时间有限时，先读第 1、2、9、11 节。第 6 节是逐项决策记录，第 7、8 节是 Review 时对照的状态与权限表。

## 1. 结论

这个 PR 做三件事：让 Agent 在 Session 内用 `penguin app` 自查 App；把 Session 子进程的凭据从启动级 admin token 换成按 Session 签发、限定 Project 的 member token；把 App 的状态、历史和坏 manifest 变成可诊断的数据。

建议分三层看：

1. **自省主体可以合并。** `app ls`、`app info`、`app history`、只读 `status`、Session `appId`、坏 manifest 诊断，边界清楚，收益直接。
2. **`app call` 先按实验性接口对待。** 它让 Agent 直接调用 App 的 `/api/*`，但当前没有调用者身份，没有 App 级开关，没有递归限制，非 2xx 的自定义错误体也会丢。代码可以保留，文档和 Skill 要明确标注，不宜把它当成稳定协议。
3. **member 级 token 是有意选择，不是只读凭证。** 它能建 Session、跑任务、改工作区，也能调用现有 member 级 App 路由，包括 reload、rollback、delete。真正的隔离边界是 Project，不是“只读”。如果产品要的是只读自省，需要另做一套凭据。

如果只能给一个合并建议：**主体合并，`app call` 标实验性或拆出，关键边界问题在合并前写清楚。**

## 2. 三个关键判断

### 2.1 入口用 CLI 加 Skill，不用动态 Tool Schema

Agent 需要知道有哪些 App、当前 App 是否在服务、版本历史。把这些做成 `penguin app` 命令，由 `penguin-orchestration` Skill 教会 Agent，按需调用。

这样做的好处是不向全局 Tool Schema 或系统提示注入动态定义，零 prompt cache 损耗；与现有 `penguin` CLI 和 Skill 的分发方式一致；App 的能力可以独立演进，不必每次改平台提示词。

代价是发现性依赖 Skill 文档，命令本身不自我描述。`app call` 尤其如此：Agent 只能按 App 作者约定的路径调用，平台不提供 `/_agent` 自描述路由。

### 2.2 状态用独立只读端点，不复用列表

旧的 `GET /workbenches?id=<id>` 会走 `settle`，发现源码变化就加载 App。Agent 只想看一眼状态，不应该因此触发编译、加载、watcher 或版本提交。

新端点 `GET /api/projects/:p/agents/:a/workbenches/:id/status` 由 `Workbenches.inspect` 支撑，不迁移、不加载、不编译、不写版本。它把四个问题拆开回答：开关是否打开，实例是否在服务，最近一次构建是什么结果，失败原因是什么。

代价是新增 API、类型和测试；`inspect` 仍要读源文件计算 revision，不是零成本；它跳过迁移，未迁移的旧布局可能返回 404。

### 2.3 凭据用 per-Session token，不用 admin token

改动前，Session 工具子进程拿到的是服务启动时的本地 API token，等同 admin。这个 token 在两个方向上都不合适：在非 admin 用户的 Project 里，admin 没有访问权，Agent 的 CLI 调用会 404；在 admin 自己的 Project 里，它又是 owner 级，还带管理端权限，权限过大。

改动后，`controlEnv` 为每个 Session 注入 `psess_<sessionId>.<mac>`。mac 用启动密钥对 Session id 做 HMAC-SHA256。服务端校验签名，再查活的 Session 行，构造合成身份 `session:<projectId>:<sessionId>`，只授予该 Project 的 member 权限。

好处是任意用户 Project 都能用；不新增 token 表；不依赖 HMR 推送；服务重启自动轮换；Session 删除后 token 失效。

代价是 token 没有显式过期，撤销依赖 Session 行和重启；bearer 一旦泄露，在失效前可重放；它和本地 admin token 共用 `sessionVia: "token"`，每一条 token-only 路由都要记得区分。

## 3. 问题背景

这个 PR 不是凭空加功能，它补的是五个具体缺口。

**Agent 没有 App 自省入口。** App 的列表、状态、版本历史只在 Web UI 和设置页可见。Agent 在 Session 里不知道自己有哪些 App，当前 App 是否在服务，最近一次构建为什么失败。

**admin token 在两种 Project 里都不合适。** 非 admin 用户的 Project，admin 进不去；admin 自己的 Project，Agent 又拿到 owner 级和管理端权限。两种都不符合“Agent 只应操作自己所在 Project”的边界。

**自省会触发加载。** 旧的列表接口以 UI 为中心，发现源码变化就加载。Agent 查状态不应产生编译、加载、watcher 或版本记录。

**Session 与 App 的归属不持久。** Workbench 创建的 Session 没有记录是哪个 App 开的。跨 Agent、子代理、服务重启之后，这个关系就丢了。

**坏 manifest 会拖垮列表。** `package.json` 解析成 `null` 或非对象时，`readFolder` 和 `readManifests` 会抛 `TypeError`，一个坏 App 让整个 App Center 或状态读取失败。

## 4. 术语与身份模型

| 术语 | 含义 | 容易混淆的点 |
| --- | --- | --- |
| App | 产品层的叫法，等同于代码里的 Workbench | 文档和 CLI 用 App，代码和 HTTP 路由用 Workbench |
| App ID | `<ownerAgentId>/<workbenchId>` | owner 是 App 所属 Agent，不是执行 Session 的 Agent |
| 执行 Agent | `PENGUIN_AGENT_ID`，Session 行里的 `agent_id` | 跨 Agent App Session 中，执行 Agent 可以不是 App owner |
| Session 归属 | `sessions.app_id`，由 `WorkbenchHost.createSession` 写入 | 子代理继承根 Session 的归属，不重新按执行 Agent 计算 |
| per-Session token | `psess_<sessionId>.<mac>`，绑定一个 Session | 它不是只读 token，也不是 admin token |

这个区分是后面几个决策的基础。没有它，跨 Agent 场景会把 App 的 owner 和执行者混为一谈。

## 5. 设计总览

一条完整的自查链路如下：

1. Agent 在 Session 内执行 `penguin app ls`、`app info`、`app history` 或 `app call`。
2. CLI 从 `PENGUIN_API_TOKEN` 读取 `psess_` token，作为 Bearer 发给服务端。
3. `authMiddleware` 先试本地 API token，再试用 Session token 验证器。
4. 验证器用启动密钥校验 HMAC，读取活的 Session 行，构造合成身份 `session:<projectId>:<sessionId>`。
5. `ProjectAccess` 只把这个身份映射到该 Session 所属 Project，角色固定为 member。
6. 路由读取 App Center、status、history，或把 `app call` 交给 `WorkbenchMain.handle`。
7. Workbench 处理器调用 `host.createSession` 时，服务端写入 `app_id`；`controlEnv` 再把它注入为 `PENGUIN_APP_ID`。

```
Agent Session
  └─ penguin app ...
       └─ Bearer psess_<sessionId>.<mac>
            └─ authMiddleware
                 ├─ HMAC 校验
                 ├─ 读取 Session 行
                 ├─ 合成身份 session:<projectId>:<sessionId>
                 └─ ProjectAccess: 仅该 Project 的 member
                      └─ App Center / Workbench routes
                           ├─ status: 只读快照
                           └─ call: WorkbenchMain.handle
App 处理器
  └─ host.createSession
       └─ sessions.app_id = <ownerAgentId>/<workbenchId>
            └─ controlEnv 注入 PENGUIN_APP_ID
```

把 PR 按设计层拆开，三块职责不同：

| 设计层 | PR 内容 | 对应决策 |
| --- | --- | --- |
| 入口与归属 | `penguin app`、规范 App ID、`sessions.app_id`、`PENGUIN_APP_ID`、`app-center?all=1`、`session ls --app` | D1-D4、D6 |
| 凭据与只读 | per-Session token、合成身份、只读 `status`、`history requireExists`、enable 失败报告 | D5、D7-D10 |
| 诊断与分发 | 坏 manifest 诊断、Skill 与 CLI 文档 | D11-D12 |

## 6. 决策日志

### A. 入口与身份

#### D1 入口形态：`penguin app` CLI 加 Skill

- **选择**：新增 `penguin app` 命令组，由 `penguin-orchestration` Skill 教 Agent 使用；不把 App 路由注入全局 Tool Schema 或系统提示。
- **为了**：Agent 按需发现 App 能力，平台不为每个 App 维护动态工具定义。
- **好处**：零 prompt cache 损耗；与现有 CLI 和 Skill 分发一致；App 可以独立演进。
- **代价**：发现性依赖 Skill 文档；命令多，帮助文本和 i18n 要维护；`app call` 没有自描述。
- **附带清理**：`app-id.ts` 不再静态引入 `isomorphic-git`，`penguin app` 启动不会拉起未使用的 Git 依赖。

#### D2 App ID：`<ownerAgentId>/<workbenchId>`

- **选择**：完整键作为规范 ID；裸 `<workbenchId>` 只在 `--agent-id` 或 `PENGUIN_AGENT_ID` 提供 Agent 时接受；App Session 内可省略参数并回退到 `PENGUIN_APP_ID`；完整 session id 会被拒绝并给出提示。
- **为了**：一个 App 可能由别的 Agent 打开 Session，只用 workbenchId 不足以定位；sessionId 和 appId 不能混用。
- **好处**：唯一、可跨 Agent 表达；与 App Center 和 URL `?app=` 使用同一个键；错误提示能纠正 Agent。
- **代价**：跨 Agent 时 `PENGUIN_AGENT_ID` 是执行者，裸 ID 可能指错；必须用完整键或省略参数。

#### D3 Session 归属：新增 `sessions.app_id`

- **选择**：`sessions` 表增加 nullable 的 `app_id`；`WorkbenchHost.createSession` 写入 `<ownerAgentId>/<workbenchId>`；`registerChildSession` 把根 Session 的 `appId` 写到子 Session 行；迁移标记为 swap-safe，旧行保持 NULL。
- **为了**：归属要持久化，服务重启后还能恢复；子代理也要能追溯。
- **好处**：`session ls --app` 可查询，并自动包含默认归档的 App Session 和跨 Agent Session；直接续跑子 Session 仍能注入 `PENGUIN_APP_ID`；旧行和旧构建兼容。
- **代价**：新列和迁移；旧 workbench 行没有归属；只记录创建者，不记录后续调用者。

#### D4 环境变量：`PENGUIN_APP_ID` 注入工具子进程

- **选择**：`controlEnv` 从根 Session 的 `appId` 取值，注入 `PENGUIN_APP_ID`；`PENGUIN_AGENT_ID` 仍表示执行 Agent。
- **为了**：App Session 内单 App 命令可以省略参数；子代理自然继承。
- **好处**：`app info`、`app call /path` 可直接用；跨 Agent 也能表达 App 身份。
- **代价**：两个 ID 容易混淆；只对 App Session 有值；Skill 必须讲清区别。

### B. 只读状态与列表

#### D5 只读状态：新增 `/status` 和 `Workbenches.inspect`

- **选择**：新增 `GET /:id/status`；`penguin app info` 改走这个端点；`inspect` 不迁移、不加载、不编译、不写版本。
- **为了**：Agent 查状态不能产生副作用，也不能因为查状态而触发构建费用。
- **好处**：安全、便宜、可重复；状态字段能区分从未加载、已停用、正常服务、服务旧版但新构建失败。
- **代价**：新增 API、类型和测试；`inspect` 仍读源文件计算 revision；跳过迁移，旧布局可能 404；与 `list` 的语义分叉。

#### D6 App 列表：复用 App Center，增加 `?all=1`

- **选择**：`GET /api/projects/:p/app-center?all=1` 返回所有已安装 App，包括隐藏、停用和坏 manifest；`enabled` 和 `hidden` 只在这个视图出现。
- **为了**：不复制一套 App Center 投影逻辑；Agent 需要看到全量，而不只是 UI 展示的那部分。
- **好处**：与 Web UI 同一数据源；不编译、不加载；坏条目也能被看到。
- **代价**：`app-center` 仍会执行一次性迁移并安装 watcher，不是严格零副作用；`all` 模式与 UI 视图语义耦合。

#### D7 历史：`history?requireExists=1`

- **选择**：`penguin app history` 始终带 `requireExists=1`；App 不存在返回 404，存在但没有版本返回空数组。
- **为了**：区分“App 不存在”和“App 存在但还没有版本记录”。
- **好处**：404 与空历史语义清楚；Agent 不会把空历史当成 App 丢失。
- **代价**：跳过迁移；旧布局可能 404，需要先跑一次 `app ls` 触发迁移；历史读取的只读性和迁移语义需要文档说明。

#### D8 启用：如实报告“开关已开但加载失败”

- **选择**：`app enable` 返回后同时检查开关和加载结果；加载失败时报告两项结果并以 1 退出。
- **为了**：避免“开关成功等于 App 可用”的假成功。
- **好处**：脚本和 Agent 能区分开关状态与构建状态；用户能看到具体错误。
- **代价**：退出码语义变化；调用方要处理“已启用但失败”的中间状态。

### C. 凭据与权限

#### D9 凭据：per-Session HMAC token

- **选择**：`psess_<sessionId>.<mac>`，mac 用启动密钥对 `session-token:v1:<sessionId>` 做 HMAC-SHA256；服务端校验签名后再查活的 Session 行。
- **为了**：不暴露启动级 admin token；不新增 token 表；不依赖 HMR 推送；服务重启自动轮换。
- **好处**：任意用户 Project 都能用；token 不可伪造；Session 删除后失效；不需要清理任务。
- **代价**：没有显式过期；撤销依赖 Session 行和重启；bearer 可重放；与本地 admin token 共用 `sessionVia: "token"`，每条 token-only 路由都要审计。

#### D10 权限：合成身份加 Project member

- **选择**：合成 `userId = session:<projectId>:<sessionId>`，`isAdmin = false`；`ProjectAccess` 只把它映射到该 Project，角色固定 member。
- **为了**：把“请求属于哪个 Project”写进身份，而不是依赖调用方传参；跨 Project 直接 404。
- **好处**：跨 Project 访问 404；owner-only 写 403；admin、用户账号写、远程会话换取拒绝；大部分路由沿用现有 Project 权限判断。
- **代价**：member 不是只读，reload、rollback、delete 等 member 路由仍可用；合成 userId 可能进入日志和 UI；未来新增的 token-only 路由需要记得检查。

### D. 健壮性与分发

#### D11 坏 manifest：降级为可诊断状态

- **选择**：`readFolder` 和 `readManifests` 容忍 `null` 或非对象 JSON；`readAppCenterManifest` 返回 `packageError`；`status` 报告 `build: "failed"` 和可读错误。
- **为了**：一个坏 App 不能拖垮整个 App Center 或状态读取。
- **好处**：坏 App 仍可见、可诊断；Agent 能告诉用户具体错误；列表不崩。
- **代价**：坏 App 仍出现在列表；错误文本来自 loader 或 manifest reader，措辞需要保持一致。

#### D12 分发：更新 Skill 与 CLI 文档

- **选择**：更新 `penguin-orchestration` Skill 和 CLI 中英文文档，补充命令、两个 ID 的区别、状态字段含义。
- **为了**：Agent 按需学习，不占系统提示；人和 Agent 看同一套语义。
- **好处**：渐进式披露；减少误用；状态字段有明确解释。
- **代价**：Skill 与 CLI 可能漂移；跨 Agent 示例容易写错，本次已修正 `app ls` 的示例。

## 7. 状态模型

`status` 的核心设计是把几个问题拆开，而不是合成一个“状态”字段。

- `enabled`：App 自己的开关。
- `agentEnabled`：Agent 总开关。
- `serving`：内存里是否有实例在服务。
- `servingRevision`：正在服务的实例对应哪个磁盘 revision。
- `build`：最近一次构建或加载的结果。
- `error`：最近一次失败的原因。
- `loadedAt`：当前实例的加载时间，或磁盘上最后一次加载检查的时间。
- `hints`：非致命提示。

常见组合如下：

| 场景 | enabled | agentEnabled | serving | servingRevision | build | error |
| --- | --- | --- | --- | --- | --- | --- |
| 从未加载 | true | true | false | null | not_loaded | null |
| App 开关关闭 | false | true | false | null | 视上次结果 | null |
| Agent 总开关关闭 | true | false | false | null | 视上次结果 | null |
| 正常服务 | true | true | true | 等于 revision | ready | null |
| 服务旧版，新构建失败 | true | true | true | 不等于 revision | failed | 具体原因 |
| 坏 manifest，无旧实例 | true | true | false | null | failed | package.json 错误 |
| 构建中 | true | true | 视旧实例 | 旧 revision 或 null | building | 旧错误或 null |

这个表解释了为什么不能只返回一个 `enabled` 或一个 `ready`。开关打开不等于实例在服务；实例在服务不等于最新构建成功；最新构建失败也不等于旧实例不能用。

## 8. 权限矩阵

per-Session token 的边界不是“只读”，而是“这个 Project 的 member”。下面的矩阵是 Review 时最需要确认的一张表。

| 操作 | 项目 owner | 项目 member | per-Session token |
| --- | --- | --- | --- |
| 读 App Center、status、history | 可以 | 可以 | 可以，仅本 Project |
| 调用 App 的 `/api/*` | 可以 | 可以 | 可以，仅本 Project |
| reload、rollback、delete、sidebar、app-center show | 可以 | 可以 | 可以 |
| app enable/disable、Agent 总开关 | 可以 | 不可以 | 403 `owner_required` |
| 创建 Session、跑任务、创建 Agent | 可以 | 可以 | 可以 |
| 写工作区文件 | 可以 | 可以 | 可以 |
| 删除 Project、改 models、改成员 | 可以 | 不可以 | 403 |
| 管理端路由 | 不可以 | 不可以 | 403 |
| 远程会话换取 | 不可以 | 不可以 | 401 |
| HMR 路由 | 不可以 | 不可以 | 401 |
| 用户账号写、创建 Project | 可以 | 可以 | 403 `session_token_forbidden` |

启动级 admin token 只在 admin 自己拥有的 Project 内有效，在 Project 内按 owner 权限处理，并额外拥有管理端权限。它不适用于非 admin 用户的 Project，因此没有放进这张表。

这张表说明两件事。

第一，PR 的标题是“自省”，但凭据是 member 级。这是有意的：Agent 本来就需要建 Session、跑任务、写工作区，只读 token 会让正常编排做不了。

第二，member 级也包含一些破坏性路由。`reload`、`rollback`、`delete` 在现有设计里就是 member 级，PR 没有收窄它们。`penguin app` 只暴露了 reload，没有暴露 delete，但持有 token 的 Agent 可以直接请求 HTTP 路由。这一点必须在合并前被明确接受，或者单独加限制。

## 9. 已知限制与后续

### 9.1 `app call` 还不是稳定的 Agent 接口

`penguin app call` 最终打到 `/:id/api/*`，交给 `WorkbenchMain.handle`。当前有三个缺口。

**没有调用者身份。** `WorkbenchRequest` 会剥离 `cookie` 和 `authorization`，也没有 `caller` 字段。App 处理器无法区分请求来自浏览器里的人，还是来自某个 Session 的 Agent。

**没有递归限制。** 任何处理器都可以调用 `host.createSession` 和 `host.run`。Agent 调一个 `/ask` 或 `/jobs` 路由，就可能产生 `Agent → App → 新 Agent` 的递归链，持续消耗额度。当前没有深度限制、链路标识或 App 级开关。

**非 2xx 的自定义错误体会丢。** CLI 的 `raw()` 在非 2xx 时抛 `ApiError`，`toError()` 只识别平台标准的 `{ error: { code, message } }`。App 常见的 `{ error: "no such deck" }` 或 `{ rev, text }` 会被压成通用的 `HTTP 409 http_error`，Agent 拿不到原因，无法自纠。

路径层面已经做了防护：`validateAppApiPath` 要求以 `/` 开头，拒绝 `//`、`..`、编码后的斜杠、反斜杠、控制字符和空白，URL 规范化无法逃出 `/workbenches/:id/api`。这挡住路径穿越，但不解决调用者身份和递归。

建议：把 `app call` 标为实验性；后续至少补三件事，App manifest 里的显式开关、可信的 `caller` 透传、递归深度或链路限制。非 2xx 响应体原样输出也应该在转正前修掉。更完整的接口设想见 `docs/app-agent-api-design.md`，本文不重复。

### 9.2 member 级包含破坏性路由

`reload`、`rollback`、`delete`、`sidebar`、`app-center show` 都是 member 级。per-Session token 因此可以删除或回滚 App，即使 CLI 没有暴露 delete。

建议：如果产品接受“Agent 是 Project 内的自治成员”，就明确写下这个边界；如果不接受，给 Session token 加一条破坏性路由的 deny list，或者另发一套只读检查 token。不要在文档里把它说成“只读自省”。

### 9.3 `app ls` 有副作用，`status` 和 `history` 跳过迁移

`app ls` 走 App Center 投影，会执行一次性布局迁移并安装 watcher。它不编译、不加载，但不是严格零副作用。真正只读的是 `status`。

反过来，`status` 和 `history?requireExists=1` 跳过迁移。在尚未迁移的旧布局上，`app info` 和 `app history` 都可能 404。正常入口是先跑 `app ls`，它会触发迁移；但直接调用 `app info` 的 Agent 可能遇到这个边界。

建议：在文档和 Skill 里把 `app ls` 写成入口命令；如果要求 `app info` 在旧布局上也可靠，需要在 `inspect` 前做一次只读的迁移检查，或者确保 Agent 列表已经触发迁移。

### 9.4 跨 Agent 时两个 ID 容易混

`PENGUIN_AGENT_ID` 是执行 Agent，`PENGUIN_APP_ID` 的 owner 前缀是 App 所属 Agent。跨 Agent App Session 中两者可以不同。

原来的 Skill 示例用 `penguin app ls -a --agent-id "$PENGUIN_AGENT_ID"`，会列执行 Agent 的 App，漏掉当前 App 的 owner。本次已改成 `penguin app ls -a --json`，先看全 Project，再用 `PENGUIN_APP_ID` 定位当前 App。

裸 `<workbenchId>` 也有同样的问题：它按 `--agent-id` 或 `PENGUIN_AGENT_ID` 补全，跨 Agent 时可能指向执行者的同名 App。跨 Agent 场景应使用完整键，或者省略参数让 `PENGUIN_APP_ID` 生效。

### 9.5 `sessionVia: "token"` 混合了两类凭据

per-Session token 复用了 `sessionVia: "token"`。这让所有原本判断“这是本机机器凭据”的路由，都可能把 Session token 当成 admin token。

PR 已经覆盖了关键位置：`apiTokenOnly`、远程会话换取、`requireHuman`、CLI 命令调用方、用户账号写、Project 创建、App Center layout。但这是一个需要持续审计的边界。更稳的做法是给 Session token 一个独立的 `via` 值，或者在 auth 层提供统一的 `isSessionPrincipal` 检查，避免未来新增路由时漏掉。

建议：短期保留现状并加测试；中期考虑独立的 `via: "session"`，让类型系统帮助发现遗漏。

## 10. 验证证据

PR 自带的验证结果如下。

| 检查 | 结果 |
| --- | --- |
| `pnpm --filter @prismshadow/penguin-cli test` | 43 个文件，660 通过，8 跳过 |
| server 定向测试 `session-token`、`app-center`、`workbenches` | 107 通过，1 跳过 |
| db 迁移测试 `db-migration-steps`、`db-upgrade` | 52 通过 |
| CLI 和 server typecheck | 通过 |
| Prettier 检查全部改动文件 | 通过 |
| `check-plugin-versions.mjs upstream/main` | 通过 |
| 手动 E2E | `app ls/info/history`、`PENGUIN_APP_ID` 回退、enable 失败退出码、token 的 404/403/401 矩阵、重复 status 不修改 `.build/status.json` |

未在本机运行全量 `pnpm build`、`pnpm test` 和安装器测试，这些由 CI 覆盖。

新增的 `packages/server/test/session-token.test.ts` 覆盖了以下关键场景：

- token 的签发、校验、篡改拒绝、错误密钥拒绝。
- 非 admin 用户 Project 中的跨 Agent Workbench Session 自查。
- 跨 Project 404、owner-only 403、admin 401/403、远程换取 401、用户写 403 的隔离矩阵。
- `status` 不创建 `.build/`，未加载的 App 报告 `not_loaded`。
- 坏 `package.json` 报告 `failed` 和可读错误。
- 旧实例仍在服务时，新构建失败同时报告 `serving: true` 和 `build: failed`。
- 子代理的 `appId` 持久化，服务重启后直接续跑仍注入 `PENGUIN_APP_ID`。

Review 时优先看这些文件：

| 文件 | 作用 |
| --- | --- |
| `packages/server/src/auth/session-token.ts` | token 的签发、校验、合成身份 |
| `packages/server/src/auth/middleware.ts` | Bearer 认证顺序和 Session token 验证器 |
| `packages/server/src/http/app.ts` | 合成身份构造、`apiTokenOnly` 和 token gate 的边界 |
| `packages/server/src/services/project-access.ts` | Session principal 到 Project member 的映射 |
| `library/session/src/repos/sessions.ts` | `appId` 的行映射和插入 |
| `library/workbench/src/workbench.ts` | `WorkbenchStatus`、`WorkbenchBuildState` 类型 |
| `packages/server/src/workbenches/service.ts` | `inspect`、`history requireExists`、`appId` 写入 |
| `packages/server/src/workbenches/routes.ts` | `status` 路由和 App Center 的 `all` 模式 |
| `packages/server/src/workbenches/app-center.ts` | 全量视图、坏 manifest 诊断 |
| `packages/server/src/cli/commands/app.ts` | `penguin app` 命令组 |
| `packages/server/src/cli/app-id.ts` | App ID 解析和路径校验 |
| `packages/server/src/runtime/session-manager.ts` | `controlEnv`、子 Session 继承 |
| `library/db/src/migrations/steps/2026-11-24-sessions-app-id.ts` | `sessions.app_id` 迁移 |
| `plugins/agent-development/skills/penguin-orchestration/SKILL.md` | Agent 使用说明 |

## 11. 待拍板问题

**Q1. `app call` 是合并前加闸门，还是先标实验性？**
建议先标实验性，保留代码，在文档和 Skill 里写明缺口；如果 upstream 的安全标准要求 App 级 opt-in，就把它拆成独立 PR，不要阻塞自省主体。

**Q2. member 级 token 能否接受包含 reload、rollback、delete？**
建议接受，并把“Agent 是 Project 内的自治成员”写成明确前提；如果不接受，需要给 Session token 加破坏性路由的 deny list，或者另做只读检查 token。

**Q3. `app ls` 的迁移和 watcher 副作用是否可接受？**
建议接受，但把 `app ls` 写成入口命令，把 `status` 和 `history` 的迁移边界写清楚；如果要求严格零副作用，需要新增一个真正的只读列表接口。

**Q4. 旧布局上的 `app info` 和 `app history` 要不要自动迁移？**
建议至少在文档里说明“先跑 `app ls`”；如果 Agent 可能在旧布局上直接调用，考虑在 `inspect` 前做一次只读迁移检查。

**Q5. 是否要给 Session token 独立的 `via` 值？**
建议中期做。短期靠 `isSessionPrincipal` 和测试兜底；长期让类型系统区分本地 admin token 和 Session token，避免新增路由时误判。

## 12. 附录

### 错误码

| 错误码 | 含义 | 触发场景 |
| --- | --- | --- |
| `401 unauthorized` | 凭据缺失或无效 | token 签名错误、Session 已删除、远程换取或 HMR 拒绝 Session token |
| `403 owner_required` | 需要 Project owner | Session token 调用 app enable/disable、Agent 总开关、删除 Project、改 models |
| `403 session_token_forbidden` | Session token 不允许的写操作 | 用户账号写、创建 Project、修改 App Center layout |
| `404 not_found` | App 或路由资源不存在 | `status` 找不到 App、`history?requireExists=1` 找不到 App |
| `404 project_not_found` | Project 不存在或无权访问 | Session token 跨 Project |
| `404 session_not_found` | Session 不存在或不属于该 Project | Session token 访问别的 Session |

### 关键提交

| Commit | 内容 |
| --- | --- |
| `6da65c0bb` | `penguin app` CLI、Session `appId`、`PENGUIN_APP_ID`、`app-center?all=1`、`session ls --app` |
| `508d0c8c2` | per-Session token、只读 `status`、`history requireExists`、坏 manifest 诊断、Skill 更新 |
| `93eaf26f4` | CLI 中英文文档补充 `penguin app` 和 `session ls --app` |
