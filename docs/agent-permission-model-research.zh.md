# Penguin App-Use 权限模型调研

> Agent 自主操作 Penguin 的边界、凭据与隔离
>
> 日期：2026-11-25 ｜ 状态：调研稿，供架构决策讨论 ｜ 作者：Chasen

---

## 0. 一页结论

**一句话结论：Agent 完全自主操作 Penguin 是可行的，但关键不在再发一个 token，而在把「租户边界 / 能力授权 / 执行隔离」三层分开建设，并同时堵住三条现有绕过路径——读数据根即 admin、同源 App 页面借用查看者 cookie、自报身份被当作授权。CDP 可抄的是协议形状，不可抄的是它的安全模型。**

### 0.1 问题（SCQA）

- **Situation**：Penguin 的页面正在 App（Workbench）化，产品希望 Agent 像人一样操作整个应用，并且多用户、不越权。
- **Complication**：现有授权只有 Project owner/member 两档；Agent 的凭据要么是启动级 admin token，要么是 Project member 级的 Session token；App 页面和 App handler 又各自借用别人的权限。三套东西都不是「能力」。
- **Question**：怎样让 Agent 自主操作，又不越权，并在多用户下成立？
- **Answer**：三层模型 + 三条堵漏 + 分阶段路线（见 0.2–0.4）。

### 0.2 五条核心结论

1. **协议形状可借鉴 CDP，安全模型必须自建。** 用「Project = 隔离域，App = Target，Session = 授权主体，Domain = 能力分类」表达对象与能力；但 CDP 本身没有认证、没有逐命令授权，「能连上 browser target 就等于能控制浏览器」。Penguin 不能复制这个前提。
2. **权限必须三层同时成立，缺一层就会被绕过。** 租户边界（Project ACL）回答「这是不是你的 Project」；能力授权（capability）回答「这个主体能不能做这个动作」；执行隔离（sandbox + egress + 凭据隔离）回答「进程实际能碰到什么」。三者不能互相替代。
3. **现状最大的风险不是 psess 太宽，而是三条绕过路径。** 读 `<root>/api-token` 即 admin；同源 Workbench 页面带着查看者 cookie 可以调用全部 API；组织/浏览器等路由把调用方自报的 `sessionId`/`agentId` 当授权依据。不堵这三条，任何 token 设计都无效。
4. **多用户下要显式建模主体、委托、执行点。** 子主体权限只能取交集：`child_scope = parent_scope ∩ requested`；admin/bypass 永不继承；每一次决策都要能回答「谁、用什么凭据、代表谁、做了什么、谁批准」。
5. **落地顺序：P0 堵漏 → P1 建模 → P2 演进。** 最需要产品拍板的不是技术选型，而是「完全自主」与「高风险动作需人在线批准」的边界。

### 0.3 三个必须先拍板的前提

| # | 决策 | 选项 | 影响 |
|---|---|---|---|
| 1 | 「完全自主」的边界 | A. 所有动作都可自主，只事后审计；B. 低风险自主，高风险 step-up；C. 只读自主，写操作一律批准 | 决定是否需要 in-task 审批状态机、审批 inbox、无人值守策略 |
| 2 | App 是否按不可信第三方代码对待 | A. 内置/用户 App 都可信；B. 用户 App 不可信，需进程/容器隔离；C. 分级：内置可信、第三方不可信 | 决定 Workbench 页面是否迁独立 origin、handler 是否出进程、manifest 是否有强制力 |
| 3 | 部署形态与威胁模型 | 单机单 OS 账号多用户 / per-user OS 账号 / per-project 容器或 microVM；防误操作 / 防 prompt injection / 防本机恶意用户 | 决定沙箱强度与「多用户隔离」能不能成立 |

**这三条不拍板，后面的协议与 token 设计会反复返工。**

### 0.4 建议路线（摘要）

| 阶段 | 目标 | 关键动作 | 验收 |
|---|---|---|---|
| **P0 堵漏** | 让现有凭据不再被一次文件读取或一次自报身份绕过 | 沙箱默认 mask `api-token`、`cli-session.json`；给 psess 加 `exp/aud/scope/jti` 与撤销；App 页面迁独立 origin 或用受限 App token；所有自报身份改成 token→session 查表 | 红队用例：Agent 从 workspace 读不到 admin 凭据；伪造 `sessionId` 无效；跨 Project 的 sessionId 无效 |
| **P1 建模** | 把「App 能力」变成一等公民 | `package.json#penguin.permissions` + AppInstallation 表 + 逐动作 capability + step-up 审批 + 统一 authz 审计 | 能回答「这个 App 被批准了哪些能力、谁批的、哪个版本」；高风险动作有 approvalId |
| **P2 演进** | 支撑跨 Project / 跨组织 / 对外开放 | `sub/act` 委托链、ReBAC（OpenFGA/SpiceDB 思路）做签发前 Check、per-project 容器/microVM、egress allowlist 代理、对外映射 MCP/A2A | 跨 Project 共享 App 可审计；多租户执行隔离有明确边界 |

---

## 1. 背景与问题定义

### 1.1 目标

产品负责人的原话可以拆成四个约束：

1. **全应用可操作**：Penguin 里的每个页面以后都会变成类似 App 的形态，Agent 要能操作它们。
2. **完全自主**：Agent 通过一个协议自主完成操作，不需要人逐步点击。
3. **不越权**：Agent 只能做人允许它做的事，不能借操作链拿到更高权限。
4. **多用户**：权限模型要考虑多用户，类似沙箱。

「类似 CDP」是协议形态的类比：一个统一的控制协议，把被操作对象抽象成 target，把能力分组，把一次控制会话抽象成 session。**不是**说安全模型也照搬 CDP。

### 1.2 为什么这不是「加一个 token」的问题

因为 Penguin 现在同时存在四种「身份来源」，它们互相不统一：

| 身份来源 | 代表谁 | 当前强度 |
|---|---|---|
| cookie 登录（`auth_sessions`） | 人 | 30 天、可撤销、可续期 |
| 启动级 `api-token` | 本机 admin | 无过期、重启轮换、能读文件即拥有 |
| per-Session `psess_` | 合成 Project member | 无过期、Project 级、`via: "token"` |
| Agent API key | 外部程序 | 无过期、按 Agent |

再加两个「不经过 token」的通道：同源 Workbench 页面直接带查看者 cookie；App handler 以 server 进程内的宿主能力行动。**结果是：即使把 psess 设计得很完美，Agent 仍可以走另外三条路拿到更高权限。** 这就是为什么必须三层一起看。

### 1.3 调研范围与方法

本调研覆盖六块：

1. CDP / WebDriver / 浏览器自动化的协议与安全模型；
2. Agent / tool 协议的授权与同意（MCP、A2A、编码 Agent 的权限模式）；
3. 能力与委托模型（object-capability、macaroon/Biscuit/UCAN、OAuth token exchange、ReBAC、AWS IAM）；
4. 多用户产品的应用/插件权限先例（GitHub Apps、Slack、Google、Notion、Figma、浏览器扩展、移动权限）；
5. OS/容器沙箱与 egress 控制；
6. PenguinHarness 现有授权面（代码审计）。

**不在本轮范围**：具体 UI 交互设计、计费模型、对外商业化条款、Windows 沙箱后端的选型验证（列为 P2 待验证项）。

---

## 2. 从 CDP 能学到什么、不能学什么

**结论：CDP 的可迁移部分是「对象—能力—连接」的协议分层；不可迁移部分是「客户端可信、端点不可达即安全」的前提。**

### 2.1 CDP 的协议形状

CDP 用四层表达控制关系：

- **Domain**：按能力分组（`Page`、`Network`、`DOM`、`Target`、`Browser`），每个 Domain 定义 command、event、type。
- **Target**：被操作对象，带 UUID、type（page/tab/iframe/worker/browser 等）、`parentId`、`browserContextId`。
- **Session**：一个客户端到某个 Target 的附着；`Target.attachToTarget(flatten=true)` 返回 `sessionId`，后续命令在消息顶层带 `sessionId` 多路复用。
- **Browser Target**：浏览器级操作，影响所有页面和 profile。

这套分层解决的是「**怎么描述对象和能力**」，不解决「**谁有资格调用**」。CDP 官方文档明确：协议客户端被假定为可信，因为客户端可以导航到任意 origin、访问所有 origin 的数据；`chrome.debugger` 之所以给部分方法加限制，是因为它把 CDP 暴露给了扩展，不是为了给普通 CDP 客户端做 ACL。`Browser.grantPermissions` 是给 web origin 授权，不是给 CDP 客户端授权。

### 2.2 CDP 的安全模型：连上即全权

- 默认只监听 `127.0.0.1` / `::1`，HTTP/WS 端点没有认证层；`/json/version` 返回带不可猜 UUID 的 browser WebSocket URL。
- 一旦连上 browser target，就能 `Runtime.evaluate` 执行页面 JS、`Storage` 读 cookie、`Page.navigate` 控制 UI、`Target.createTarget/createBrowserContext` 创建新页面或上下文。
- Host/Origin 校验是浏览器侧的 CSRF/DNS rebinding 防御，不是用户认证；无 Origin 的本地进程默认放行。
- 历史上有大量暴露的 Selenium Grid / CDP 端点被用来挖矿、偷 cookie 的案例。
- Chrome 136 起默认 profile 不再接受远程调试开关；Chrome 144 起默认关闭，并要求逐连接 Allow/Deny + 可见 banner——这是产品层在补「同意」，不是协议层在补授权。

**结论：CDP 是控制通道，不是沙箱，也没有 per-command ACL。**

### 2.3 映射到 Penguin

| CDP 概念 | Penguin 对应 | 注意 |
|---|---|---|
| BrowserContext | Project（租户/隔离域） | 前提是 Project 有真正的数据/执行隔离，而不只是 ACL |
| Target | App（Workbench），App ID = `<ownerAgentId>/<workbenchId>` | App 内的 view/panel 可作 child target；admin/browser 级 target 不暴露 |
| Session（连接） | 一次 Agent 的 attach | **不要**和 Penguin 的 Session（执行主体）混名；建议协议里叫 `attachment` |
| Domain | 能力分类（`app.*`、`session.*`、`agent.*`、`org.*`） | Domain 只是分类，权限要逐 command 判定 |
| Browser Target | 平台管理面（admin.*） | 永不由 Agent 自主 |

一个可落地的协议形状：

```
penguin.target.list                       # 列出可附加的 App / view / session
penguin.target.attach {target, grant}     # 返回 attachId
penguin.app.read   {attachId, path}       # 只读
penguin.app.invoke {attachId, route, method, body}  # 调 App 声明的 agent-callable 路由
penguin.session.read/create/steer {attachId, ...}
penguin.agent.run  {attachId, ...}
penguin.workbench.install {attachId, ...} # 高风险，默认需批准
penguin.admin.*                            # Agent 不可用
```

`attachId` 是路由标识，不是权限凭据；服务端必须在每次 command 上检查 `grant` 与 target 的匹配。

### 2.4 结论

- 抄：Domain/Target/Attachment 的分层、能力可发现、事件模型、flat session 多路复用。
- 不抄：无认证端点、客户端可信假设、browser target 全权、把 loopback/不可猜 URL 当安全。
- 补：认证、逐 command 授权、审批、审计、速率/递归/预算限制。

---

## 3. 权限模型的三个层次

**结论：三层不可互相替代。只做 token 会被「读数据根即 admin」和「同源页面借用 cookie」绕过；只做沙箱则无法区分「哪个 App、哪个动作、代表谁」。**

### 3.1 第一层：租户边界

回答「这是不是你的 Project」。Penguin 已有实现：`ProjectAccess` 的 owner/member 判断，失败统一 404，不泄露资源存在性；admin 不是旁路。

这一层的价值是**隔离**，不是**授权**。它适合决定「你能不能进这个 Project」，不适合决定「你能不能删这个 App」「你能不能代表这个员工发消息」。现状的问题是把 member 当成了能力集合：member 里包含代码执行、文件系统、组织管理、审批等全部高危动作。

### 3.2 第二层：能力授权

回答「这个主体能不能对这个资源做这个动作」。一个 capability 至少要包含：

| 字段 | 含义 |
|---|---|
| `sub` | 被代表的主体（user / session / installation） |
| `act` | 代理链（session → subagent → app），只用于归因，不参与授权收缩 |
| `aud` | 目标受众（`penguin-api` 或 `app:<id>`），防止 token 被拿去调别的服务 |
| `project` / `app` / `session` | 资源绑定 |
| `scope` | 动作集合，如 `app.read`、`app.invoke:decks.*`、`session.self.write` |
| `exp` / `nbf` / `iat` / `jti` / `parent_jti` | 生命周期与撤销 |
| `approval` | 高风险动作的批准记录 ID（可选） |

核心性质：**不可伪造、可衰减、可过期、可撤销、可离线或在线验证**。子主体的 scope 只能取交集，不能取并集。

### 3.3 第三层：执行隔离

回答「进程实际能碰到什么」。沙箱不是授权的替代品，而是授权的兜底：即使模型被 prompt injection，进程也读不到不该读的文件、连不到不该连的域名。

Penguin 现状的关键事实：

- 沙箱默认只限制写，**读默认不受限**；`maskPaths` 为空。
- `<root>/api-token` 是启动级 admin 凭据，`<root>/cli-session.json` 是人的 30 天会话；同一 OS 用户可读。
- 因此「Agent 读 token 文件后以 admin 调本地 API」是一条现实的提权链。

执行隔离的正确形态是三层同时成立：

1. **最小文件视图**：默认不挂 data root，只挂 workspace/scratchpad；敏感文件 deny/mask。
2. **默认拒绝 egress**：出站走沙箱外代理，按 hostname allowlist 判定，先判后解析，阻断私网/元数据地址。
3. **凭据不进 Agent 可读面**：短时凭据、代理注入、credential masking；长期 secret 只存在于 broker。

### 3.4 三层关系

| 层 | 回答的问题 | Penguin 现状 | 不能替代谁 |
|---|---|---|---|
| 租户边界 | 这是不是你的 Project | 有，质量较好 | 不能替代能力授权；member 不是能力集合 |
| 能力授权 | 你能不能做这个动作 | 弱；psess 是 member 身份，不是 capability | 不能替代沙箱；token 无法阻止进程读文件 |
| 执行隔离 | 进程实际能碰到什么 | 弱；默认读全开、egress open | 不能替代授权；沙箱不知道「代表谁」 |

---

## 4. Penguin 现状

**结论：psess 是一次正确的净收窄，但它是「Project member 身份」，不是「会话能力」。现状有 12 个已确认缺口，其中 3 个属于「绕过级」。**

### 4.1 主体与凭据矩阵

| 主体 | 凭据 | 生命周期 | 实际权限 | 问题 |
|---|---|---|---|---|
| 人类用户 | `auth_sessions` cookie | 30 天，滑动续期，可撤销 | 按 Project role | 无法表达 App/Agent 的细粒度授权 |
| 本机 admin | `<root>/api-token` | 本次 boot，无过期 | admin + 自己 Project 的 owner | 读文件即 admin；多用户同机无法隔离 |
| Agent Session | `psess_<sessionId>.<mac>` | 本次 boot + session 行存在 | 该 Project 的 member | 名为会话，实为 Project member；无 exp/aud/scope/jti |
| 外部程序 | Agent API key | 无过期 | 按 Agent | 无过期、无调用审计 |
| 远程会话 | 1 小时换取的人类会话 | 1 小时 | 人类权限 | 可被 boot token 换取 |
| desktop/setup | 特殊 cookie 会话 | 短 | 受限 | 与 Agent 无关 |

### 4.2 资源层级

```
user ── member/owner ──> project ──> agent ──> workbench/app ──> app route
                                         └──> session ──> subagent session
project ──> org ──> employee / channel / ticket / calendar / handbook
```

已持久化的关系：project membership、agent 归属、`sessions.app_id`、org/employee。未持久化的：Session 创建者（`created_by_user_id`）、完整 actor chain、App installation/grant。

### 4.3 已确认的缺口（按严重度）

#### 严重：三条绕过路径

1. **读数据根即 admin。** `<root>/api-token` 是 boot 级随机 token，Bearer 匹配即内置 admin；沙箱默认读全开、`maskPaths` 为空。Agent 可以 `cat` 后以 admin 调本地 API。psess 只替换了 `PENGUIN_API_TOKEN`，没有消除磁盘上更高权的凭据。
2. **同源 Workbench 页面以查看者身份全权运行。** Workbench iframe 是 `allow-scripts allow-same-origin`，SDK 用 `credentials: same-origin`；页面请求自带查看者 HttpOnly cookie，可以调用任意 `/api` 路由，包括 owner-only 与 admin 路由。App handler 又拿不到调用者身份，且 `createSession` 默认 `allow-all`。「Agent 写一个 App 诱导 owner 打开」即构成完整越权链。
3. **自报身份被当作授权。** 组织路由在 `sessionVia === "token"` 时直接采信 body/query 的 `sessionId`/`agentId`；浏览器自动化在带 `sessionId` 时把 actor 替换成该 Session 的 driver，再按 driver 做 Project 校验。`sessionId` 没有和 token 自身绑定。项目 A 的 psess 若知道项目 B 某个运行中 session id，就能以那个人身份操作浏览器；同 Project 内可冒名任意员工发频道消息、建工单。

#### 高：权限与生命周期

4. **psess 实际等于 Project member。** 可读/改/删/steer 同 Project 任意 Session（含人正在跑的）、创建 Agent 与 Session、写 Agent config/AGENTS.md、装 hooks/插件（等于代码执行）、导入/重载/删除 Workbench、通过 workspace-files 以任意绝对路径读写、创建/修改组织与员工。只有 3 处显式拒绝合成主体（me 非 GET、建 Project、App Center layout）。
5. **App 是持有项目级权力的 confused deputy。** 任何 member（含 psess）都能触发 `/:id/api/*` 执行 App 服务端代码；App 持有 `listAgents`、`createSession`（默认 `allow-all`）、`run`、`dataPaths`、`socketPath`；只校验同 Project；Agent → App → 新 Agent 可递归；无并发/深度/预算上限。
6. **凭据生命周期缺失。** psess 无 `exp`/`jti`，有效期等于本次 boot + session 行存在；`agent_api_keys` 无 `expires_at`；SSE 对 Bearer 请求把 `sessionIsLive` 恒置 true，session 删除后流仍继续写。
7. **没有授权审计。** HTTP access log 只有 method/path/status/耗时，无 userId/credential 类型；`error_records` 只记错误；`usage_records` 只记模型用量；psess 不落表；组织/浏览器/Workbench 动作没有统一 actor 审计。

#### 中：策略与部署

8. **审批模式可被同 Project 主体改。** `PATCH /api/sessions/:id` 只要求项目访问，`approvalMode` 直接接受请求值；`POST /:id/steer` 在 token 或他人 driver 时会清空 runDriver。低权主体可把他人运行中会话改成 `allow-all` 再注入 steering。
9. **App 没有 manifest、安装记录与版本化授权。** 只有 `package.json` 与导入 zip；没有 `penguin.permissions`、没有 AppInstallation 表、没有 grantedScopes/approvedBy/version 绑定；升级不触发重新批准。
10. **egress 默认 open。** 沙箱 presets 网络多为 open；没有域名 allowlist、没有沙箱外代理、没有凭据注入；Workbench handler 可直接 fetch/socket。模型 API 本身也是沙箱外通道。
11. **多用户同机的 OS 边界未定义。** OS 沙箱的边界是 OS 账号；同一 OS 账号跑 server 与所有 Agent 时，bwrap/Seatbelt/AppContainer 不能把产品用户 A 与 B 隔开；Windows 原生 ACL 后端只封写、不能读隐藏。若做托管多用户，需要 per-project 容器/microVM 或 per-user OS 账号。
12. **App ID 只是 Project 内唯一。** 规范键 `<ownerAgentId>/<workbenchId>` 不含 projectId；跨 Project 聚合会撞键。它满足「与 Session ID 不冲突」的硬要求，但不是全局身份；如果未来需要跨 Project 的 App 身份，需要加 projectId 或不可变 `appId` 字段。

### 4.4 PR #1791 的位置

PR #1791 把 Session 子进程的凭据从启动级 admin token 换成按 Session 签发、限定 Project 的 member token；同时增加了 `penguin app ls/info/history/reload/call`、只读 `status`、`sessions.app_id`、`PENGUIN_APP_ID`、坏 manifest 诊断。

它的价值是**真实的净收窄**：在非 admin 用户的 Project 里，Agent 从「完全不可用」变成「Project member」；在 admin 自己的 Project 里，从「owner + 管理端」降到「member」。但它只完成了能力模型的第一步，且把「Session token」这个名字用在了「Project member 凭据」上，导致后续所有 `via === "token"` 的路由都需要重新审计。**它不是一个完整的权限模型，而是一次必要的止血。**

---

## 5. 外部先例速览

**结论：主流做法不是「给 Agent 一个万能 token」，而是「安装实例 + 声明式能力 + 运行时收窄 + 执行隔离 + 审计」。**

### 5.1 Agent / tool 协议

| 机制 | 代表 | 可借鉴 | 不适用 |
|---|---|---|---|
| OAuth 2.1 + Resource Indicators | MCP Authorization | token 绑定 audience/resource；scope 最小化；用户同意；禁止 token passthrough | 本地 CLI 不需要完整 OAuth AS |
| 权限模式阶梯 | Claude Code / Codex / Cursor | read-only / ask / allow-list / auto / bypass；deny > ask > allow；sandbox 与 approval 正交 | bypass 不能给 Agent 自选；子代理不继承 |
| 任务内授权 | MCP / OAuth step-up | 权限不足返回 `403 insufficient_scope`，client 走 step-up；长任务用 `auth_required` + 幂等重试 | 无人值守会卡住，需要 fail-closed 策略 |
| Lethal trifecta | Simon Willison | 私密数据 + 不可信内容 + 外发能力 = 高危组合；必须切断至少一条腿 | 不能只靠分类器 |

编码 Agent 行业的共同结构已经很清晰：**mode ladder + 按工具/路径/命令的规则 + 独立 sandbox + approval 通道 + 可选 classifier**。其中 deny 规则和 sandbox 是硬边界，allowlist/auto-review 只是降摩擦。

### 5.2 能力与委托

| 机制 | 代表 | 可借鉴 | 代价 |
|---|---|---|---|
| Object-capability / POLA | E、Capsicum、Fuchsia | 默认无 ambient authority；按需授予；子主体只能拿子集 | 需要运行时强制，HTTP/JSON 不天然适配 |
| Macaroon / Biscuit / UCAN | 去中心化凭证 | 可衰减（caveat/block/delegation）、可离线验证、可按链撤销 | 规范复杂；Penguin 集中签发时收益不如 JWT claims |
| OAuth Token Exchange (RFC 8693) | `sub` / `act` / `may_act` | 保留用户身份、记录代理链、防 confused deputy | `act` 只做归因，不做权限收缩；收缩要靠签发方 |
| ReBAC / Zanzibar | OpenFGA / SpiceDB | 表达 Project/Org/App/Session 的层级与继承；Check/ListObjects | 额外服务与一致性成本；不产生委托凭证 |
| AWS IAM | policy evaluation | default deny；identity + resource policy；permissions boundary 取交集；ExternalId 防 confused deputy | 策略语言复杂；union 会放大权限 |

**对 Penguin 的结论**：先做 capability token（JWT/JWS 或 HMAC claims + server-side session 查表），ReBAC 后置，Biscuit/UCAN 只在需要离线或跨信任域委托时再评估。

### 5.3 多用户应用权限先例

| 产品 | 机制 | 对 Penguin 的启示 |
|---|---|---|
| GitHub Apps | installation token，绑定 installation，1 小时过期，细粒度 permissions，安装时选 repository | **installation-as-tenant**：App 安装到 Project 产生 AppInstallation，token 绑定 installation |
| Slack | app manifest + OAuth scopes + workspace admin 审批 | 双层同意：安装时批准 tenant 级 scope，高风险动作再审批 |
| Google Workspace | OAuth scopes（sensitive/restricted）+ admin allowlist + 验证 | admin 管全局 allowlist；高风险 scope 需要平台审核 |
| Notion | integration capabilities（read/update/insert）+ 页面级共享 | manifest 声明能力，资源选择器收窄 |
| Figma | plugin manifest + network allowlist + organization 管理 | 网络域名要声明；组织可禁用插件 |
| Chrome 扩展 | manifest permissions + optional permissions + host_permissions | 必选/可选权限分离；运行时 step-up |
| Android / iOS | 安装时声明 + 运行时权限 + 特殊权限 | 敏感能力单独闸门；默认关闭 |
| VS Code | workspace trust（粗粒度，信任后扩展全权） | **反例**：不能拿 workspace/Project 级 trust 当 App 权限模型 |

五个先例一致验证了这套形状：**manifest 是请求，grant 是事实；安装时同意 tenant 级能力，运行时按需 step-up；高风险能力单独闸门；升级新增 scope 要重新批准。**

### 5.4 沙箱与 egress

| 机制 | 边界 | 对 Penguin 的适用性 |
|---|---|---|
| seccomp-bpf | syscall | 加固层，不单独承担隔离 |
| Landlock / Seatbelt | 文件路径、端口 | Linux/macOS 主力；需补默认隐藏与 egress |
| bubblewrap / mount namespace | 文件视图、网络 namespace | Linux 主力；默认不挂 data root |
| AppContainer / restricted token / Job | Windows 文件/注册表/网络/进程 | Windows 长期方向；短期 WSL 更稳 |
| gVisor | 用户态内核 | 高安全可选 backend |
| Firecracker / microVM | 硬件级隔离 | 托管多租户的正确档位 |
| egress proxy | 域名 allowlist、凭据注入、审计 | **Penguin 完全缺失，应作为默认** |
| credential masking | 真实凭据不进沙箱 | 直接替代 `PENGUIN_API_TOKEN` 与 vault 明文 env |
| short-lived credential exchange | 分钟级 lease | psess 应改为短时 + 可撤销 |

关键判断：**沙箱的边界是 OS 账号，不是产品用户。** 如果多个产品用户共享一个 OS 账号，LSM/namespace 不能把他们隔开；要做真正的多用户隔离，需要 per-project 容器/microVM 或 per-user OS 账号。Windows 原生若做不到读隐藏，产品必须诚实标注「非保密边界」。

---

## 6. 目标权限模型建议

**结论：用六维模型定义权限，用 capability 承载运行时授权，用 sandbox + egress 兜底，用 approval + audit 形成闭环。**

### 6.1 六维模型

| 维度 | 定义 | Penguin 落点 |
|---|---|---|
| Subject | 谁在行动 | human(userId)、session(psess)、subagent、app-installation、org-employee、admin、machine |
| Action | 做什么 | `app.read` / `app.operate` / `app.delete` / `app.invoke:<route>` / `session.read` / `session.steer` / `session.create` / `agent.run` / `workbench.install` / `org.employee.write` / `net.egress:<domain>` / `admin.*` |
| Resource | 对什么做 | project / app(`agentId/workbenchId`) / session / agent / org / route / file-scope |
| Delegation | 代表谁 | `sub` + `act`；child = parent ∩ requested；admin/bypass 永不继承 |
| Enforcement | 在哪强制 | 网关 authorize（complete mediation）+ OS sandbox（FS/网络）+ 沙箱外 egress 代理 |
| Audit | 怎么追责 | actor、credential 类型与 jti、主体链、资源、动作、decision、approvalId、时间 |

### 6.2 主体与凭据设计

建议把 psess 从「sessionId + MAC 的身份」升级为「带 claims 的 capability」：

```jsonc
{
  "v": 1,
  "sub": "user:<userId>",              // 被代表的人；纯 Agent 自主时可为 "session:<rootSessionId>"
  "act": ["session:<root>", "session:<current>"],  // 代理链，只用于归因
  "aud": "penguin-api",                // 或 "app:<agentId>/<workbenchId>"
  "project": "<projectId>",
  "app": "<agentId>/<workbenchId>",    // 可选，App-scoped token
  "session": "<sessionId>",
  "scope": ["app.read", "app.invoke:decks.*", "session.self.read"],
  "exp": 1760000000,
  "jti": "<uuid>",
  "parent_jti": "<uuid>",
  "approval": "<approvalId>"           // 可选，高风险动作
}
```

服务端验证顺序：签名 → `exp/nbf` → `aud` 匹配 → `scope` 覆盖 → session 行存活 → revocation epoch → 资源归属。**任何一步不通过都拒绝，不做自动降级。**

### 6.3 动作与默认策略

| 动作类 | 示例 | 默认策略 | 谁能批准 |
|---|---|---|---|
| 读 App 状态/历史 | `app.read` | Agent 自主 | — |
| 调 App 只读路由 | `app.invoke:GET /decks` | manifest 声明 + Agent 自主 | 安装时 owner |
| 调 App 写路由 | `app.invoke:PATCH /decks/:id` | manifest 声明；首次 step-up | Project owner 或会话所有者 |
| 创建 Session / 跑 Agent | `session.create` / `agent.run` | 默认 ask；可配置 per-App | Project owner |
| reload / rollback | `app.reload` / `app.rollback` | owner 或 step-up | Project owner |
| delete App / 装插件 / hooks | `app.delete` / `workbench.install` | owner only + 二次确认 | Project owner |
| 代表用户发消息 | `message.send_as_user` | 单独同意 | 该用户本人 |
| egress | `net.egress:<domain>` | 默认 deny；manifest 声明 | Project owner / admin |
| admin.* | 用户管理、平台设置 | 永不开放 | — |

### 6.4 委托与衰减

- `child_scope = parent_scope ∩ requested_scope`，由签发方强制，不能由子主体自己声明。
- `act` 只记录代理链，不参与授权；授权看 `sub` + 当前 subject 的交集。
- 子代理、App 创建的 Session、schedule 驱动的 Session 都必须显式携带 scope，不能默认继承全量。
- admin/bypass 永不继承；App 不能自选 `allow-all`。
- 递归限制：App → Session → App 的深度、并发、per-root 预算都要有上限。

### 6.5 执行点

1. **网关**：所有 API/协议命令先过 `authorize(actor, action, resource)`；complete mediation，不能有「handler 自己记得检查」的路径。
2. **沙箱**：默认不挂 data root；`api-token`、`cli-session.json`、`web.db` 列入 deny/mask；按平台选择 Seatbelt/bwrap/AppContainer/WSL。
3. **egress 代理**：所有出站走沙箱外代理；hostname allowlist；先判后解析；阻断私网/元数据地址；记录 destination/bytes/decision。
4. **凭据代理**：长期 secret 只存在于 broker；Agent 拿短时 token 或 sentinel，由代理在出站时替换。

### 6.6 同意与审批

三层同意：

1. **安装时**：Project owner 批准 App manifest 的 tenant 级 scope（read/operate/invoke/egress/spawn）。
2. **运行时**：Agent 触发 optional scope 时，平台向在线人类发 approval card；无人在线则 fail closed（或按预授权窗口）。
3. **单次高风险**：delete/rollback/represent-user/egress 走 step-up，返回 `auth_required` + request id，客户端幂等重试。

所有同意都要落 `approval` 记录：谁、对哪个 App/动作、什么 scope、有效期、批准时间。

### 6.7 审计

新增 `authz_audit`：

| 字段 | 内容 |
|---|---|
| `tenant` | projectId / orgId |
| `actor` | userId / sessionId / appId |
| `credential_type` | cookie / api-token / psess / agent-key / remote |
| `jti` / `parent_jti` | 凭据与链 |
| `resource` | app / session / agent / route |
| `action` | `app.invoke:...` 等 |
| `decision` | allow / deny / ask |
| `approval_id` | 若走了审批 |
| `ts` | 时间 |

HTTP log 至少加 `userId` 与 `credential_type`；Agent 默认不可读审计。保留期与访问控制需要产品定义。

---

## 7. 路线图

### P0：堵漏（先让现有边界成立）

| # | 动作 | 第一步 | 验收 |
|---|---|---|---|
| 1 | 沙箱 mask 数据根凭据 | 在 Seatbelt/bwrap/dsh 的默认策略里 deny `api-token`、`cli-session.json`、`web.db` | Agent 在 workspace 内 `cat` 不到；安全测试用例 |
| 2 | psess 升级为 capability | 加 `exp/aud/scope/jti`；保留 session 行校验；mint 时写 revocation epoch | 伪造 scope/aud 被拒；过期被拒；删除 session 后立即失效 |
| 3 | App 页面与 handler 拆身份 | 页面迁独立 origin（或去掉 `allow-same-origin`，走 postMessage/host bridge）；handler 注入可信 `WorkbenchCaller` | App 页面不再自动带人的 cookie；handler 能看到 caller 类型 |
| 4 | 自报身份改查表 | 抽 `assertSessionBinding(token, sessionId)`；组织/浏览器/plugin/schedule 全部入口强制 | 伪造 sessionId/agentId 返回 404/403 |
| 5 | 审计最小版 | `authz_audit` 表 + HTTP log 主体字段 | 能回答「哪个 session 在何时做了什么」 |

### P1：建模（让 App 能力成为一等公民）

| # | 动作 | 第一步 | 验收 |
|---|---|---|---|
| 6 | App manifest | `package.json#penguin.permissions`：capabilities、egress、spawnSession、destructive | 未声明的路由不可被 Agent 调用 |
| 7 | AppInstallation | 新表：projectId/appId/version/manifestHash/grantedScopes/approvedBy/status | 能列出「这个 App 被批准了什么」 |
| 8 | 逐动作 capability | 网关按 `app.invoke:<route>` 检查；manifest 与 registry 冲突时以 registry 为准 | 越权 route 被拒 |
| 9 | step-up 与审批 | 高风险动作返回 `auth_required`；CLI/UI 弹出批准 | 有 approvalId、有效期、可撤销 |
| 10 | createSession 默认收紧 | App 创建的 Session 默认 `approvalMode ≤ ask`；递归深度/并发/预算上限 | 递归链可被截断；费用可归因 |

### P2：演进（多租户与对外）

| # | 动作 | 第一步 | 验收 |
|---|---|---|---|
| 11 | 委托链 | `sub/act` 进 token；子会话 scope 取子集 | actor chain 可审计 |
| 12 | ReBAC PDP | 用 SQL/关系表先实现 Check；规模上来再评估 OpenFGA/SpiceDB | 跨 Project/Org 的授权可解释 |
| 13 | 多租户执行隔离 | per-project 容器/microVM；Windows 用 WSL/AppContainer 兜底 | 产品用户之间无共享 OS 边界 |
| 14 | egress allowlist | 沙箱外代理 + manifest 域名 + 默认 deny | 未声明域名被阻断 |
| 15 | 对外协议 | 若要开放 MCP/A2A，再补 OAuth 2.1 + RFC 8707/9396 映射 | 第三方可安全接入 |

---

## 8. 待产品负责人拍板的问题

1. **「完全自主」是否包含高风险动作？** 如果要求所有动作都能无人值守完成，就不能默认 deny egress / 强制 step-up；如果接受高风险动作需人在线批准，需要审批 inbox 与 fail-closed 策略。
2. **App 是否按不可信第三方代码对待？** 这决定 Workbench 页面是否必须迁独立 origin、handler 是否必须出进程。当前同源 iframe + 查看者 cookie 的设计，在「不可信 App」前提下是不成立的。
3. **部署形态是什么？** 单机单 OS 账号多用户，还是 per-project 容器/microVM？这决定「沙箱」能不能成为多用户边界，也决定 Windows 是否必须降级。
4. **psess 的目标作用域是「本 Session / 本 App」还是「整个 Project member」？** 代码与测试按 member 设计；命名与产品目标暗示应更窄。这个决定直接影响 capability 的 claim 设计和现有 App 的迁移。
5. **App 能否创建 Session、调用其他 Agent、递归派生？** 如果可以，深度/并发/预算上限是多少？费用记到 Project、App installation、调用 Session，还是公司员工预算？
6. **是否需要对外暴露为 MCP server / A2A agent / 第三方 App 生态？** 如果只服务内部，自定义 capability 足够；如果对外开放，需要完整的 OAuth 2.1、audience、客户端注册与 scope 映射。
7. **撤销延迟目标是多少？** 秒级还是分钟级？是否接受 holder binding（DPoP/mTLS）来防 token 复制？这决定无状态 HMAC 短 TTL 还是有状态 jti/epoch。
8. **审计保留多久、谁能看？** Agent 是否允许读自己的审计？App payload 中的敏感数据是否要脱敏后再落日志？

---

## 9. 假设、证据不足与验证方式

| # | 事项 | 状态 | 验证方式 |
|---|---|---|---|
| 1 | Chrome 144 第三方工具可用 UI Automation 自动点 Allow | 来自 GitHub issue，未独立验证 | 本地复现或查 Chrome 发布说明 |
| 2 | CDP 不是稳定公开 API，Penguin 借其形状需自行版本化 | 文档明确 | 设计时定义自己的协议版本 |
| 3 | 「每个 Penguin 页面都能映射成 target/command」 | 未证明；Workbench 后端能力需单独 domain | 选 2–3 个内置 App 做协议 PoC |
| 4 | MCP 规范版本与 OAuth 2.1 draft 编号会变化 | 截至 2026-11 调研 | 落地时重新核对 |
| 5 | Windows 原生「不能读隐藏」是 Penguin 实现现状，不是平台不可实现 | 需验证 | 用 AppContainer/restricted token 做 PoC |
| 6 | 「审批疲劳」「无人值守」的实际产品约束 | 判断，无数据 | 访谈 + 埋点 |
| 7 | 读 `<root>/api-token` 即 admin 在目标部署中可达 | 代码事实 + 沙箱默认读全开 | 红队用例 |
| 8 | 同源 App 页面以查看者 cookie 调 admin 路由 | 代码事实 | 端到端验证 |

---

## 附录 A：主要来源

**CDP / 浏览器自动化**

- CDP 官方文档：https://chromedevtools.github.io/devtools-protocol/
- CDP 贡献指南（客户端可信假设）：https://chromium.googlesource.com/chromium/src/+/HEAD/third_party/blink/public/devtools_protocol/README.md
- Target / attachToTarget / createBrowserContext：https://chromedevtools.github.io/devtools-protocol/tot/Target/
- Chrome 远程调试默认限制：https://developer.chrome.com/blog/remote-debugging-port
- Chrome DevTools MCP 的逐连接批准：https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session
- Chrome 多客户端：https://developer.chrome.com/blog/new-in-devtools-63#multi-client
- WebDriver 安全章节：https://w3c.github.io/webdriver/#security
- Selenium Grid 被挖矿案例：https://www.wiz.io/blog/seleniumgreed-cryptomining-exploit-attack-flow-remediation-steps

**Agent / tool 协议**

- MCP Authorization：https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization
- MCP 安全最佳实践：https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices
- MCP Tools：https://modelcontextprotocol.io/specification/2026-07-28/server/tools
- A2A 规范：https://a2a-protocol.org/latest/specification/
- Claude Code 权限：https://code.claude.com/docs/en/permissions
- Claude Code sandbox：https://code.claude.com/docs/en/sandboxing
- Codex sandbox / approvals：https://learn.chatgpt.com/codex/sandboxing
- Cursor run modes：https://cursor.com/docs/agent/security/run-modes
- Lethal trifecta：https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/

**能力 / 委托**

- RFC 8693 Token Exchange：https://www.rfc-editor.org/rfc/rfc8693.html
- RFC 8707 Resource Indicators：https://www.rfc-editor.org/rfc/rfc8707.html
- RFC 9396 Rich Authorization Requests：https://www.rfc-editor.org/rfc/rfc9396.html
- Macaroons 论文：https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/41892.pdf
- Biscuit 规范：https://github.com/eclipse-biscuit/biscuit/blob/main/SPECIFICATIONS.md
- UCAN 规范：https://github.com/ucan-wg/spec
- Zanzibar 论文：https://www.usenix.org/system/files/atc19-pang.pdf
- OpenFGA 概念：https://openfga.dev/docs/concepts
- SpiceDB 一致性：https://authzed.com/docs/spicedb/concepts/consistency
- AWS IAM policy evaluation：https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html
- AWS confused deputy：https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html

**多用户应用权限**

- GitHub Apps installation token：https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-an-installation-access-token-for-a-github-app
- GitHub Apps permissions：https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/choosing-permissions-for-a-github-app
- Slack OAuth 安装：https://docs.slack.dev/authentication/installing-with-oauth
- Google OAuth scopes：https://developers.google.com/identity/protocols/oauth2/web-server
- Notion capabilities：https://developers.notion.com/reference/capabilities
- Figma plugin manifest：https://developers.figma.com/docs/plugins/manifest/
- Chrome 扩展权限：https://developer.chrome.com/docs/extensions/develop/concepts/declare-permissions
- Android 权限：https://developer.android.com/guide/topics/permissions/overview
- VS Code workspace trust：https://code.visualstudio.com/api/extension-guides/workspace-trust

**沙箱 / egress**

- Landlock：https://docs.kernel.org/userspace-api/landlock.html
- seccomp：https://docs.kernel.org/userspace-api/seccomp_filter.html
- bubblewrap：https://github.com/containers/bubblewrap
- Windows AppContainer：https://learn.microsoft.com/en-us/windows/win32/secauthz/appcontainer-isolation
- gVisor security：https://gvisor.dev/docs/architecture_guide/security/
- Firecracker：https://firecracker-microvm.github.io/
- Claude Code sandboxing：https://www.anthropic.com/engineering/claude-code-sandboxing
- Codex sandbox：https://openai.com/index/inside-the-codex-sandbox/

## 附录 B：Penguin 代码证据索引

| 主题 | 文件 |
|---|---|
| 启动级 admin token | `packages/server/src/auth/api-token.ts` |
| per-Session token | `packages/server/src/auth/session-token.ts` |
| 认证顺序与 `via` | `packages/server/src/auth/middleware.ts`、`packages/server/src/http/app.ts` |
| Project ACL | `packages/server/src/services/project-access.ts` |
| Session 路由（读/fork/删/steer） | `packages/server/src/http/routes/sessions.ts` |
| 组织路由的自报身份 | `packages/server/src/http/routes/organizations.ts`、`packages/server/src/runtime/organization/service.ts` |
| 浏览器自动化的 actor 替换 | `plugins/browser-automation/src/server/routes.ts` |
| Workbench 页面与 handler | `packages/web/src/lib/workbench-theme.ts`、`packages/server/src/workbenches/routes.ts`、`packages/server/src/workbenches/service.ts` |
| 沙箱策略 | `packages/server/src/sandbox/service.ts`、`rfcs/PRFC-0009-sandbox.zh.md` |
| App ID 解析 | `packages/server/src/cli/app-id.ts` |
| Skill 分发 | `plugins/agent-development/skills/penguin-orchestration/SKILL.md`、`packages/core/src/plugin/library-reader.ts` |
| 授权审计现状 | `packages/server/src/http/app.ts`、`library/db/src/schema.ts` |

---

**文档结束。** 本文是调研与建议，不是最终架构决策；第 8 节的问题需要产品负责人确认后，才能进入协议与数据模型的详细设计。
