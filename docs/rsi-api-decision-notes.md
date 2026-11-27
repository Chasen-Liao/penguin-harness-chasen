# RSI API 设计决策与核查备忘

> **用途**：记录当前设计（`rsi-api-design.md`）背后的决策来源、评审意见去向、本轮拍板结果与核查证据。本文是过程与决策账本，不是规范；规范以主设计文档与 v3 归档为准。
> **日期**：2026-11-27；**基线**：`main @ 08ee6d4d2`。
> **归档**：旧稿、评审清单与旧展示稿在 `docs/archive/`；外部讨论底稿在 `D:/tmp/`，见 §7。

## 0. 一页结论

- 评审清单 10 条中，9 条已在当前设计落实，1 条部分落实（AMCP 映射待样例）。
- 评审报告的 35 项决策（A/B/C 三组）中，A 组 9 项已确认；B 组大部分已写入，3 项为默认/部分（O(n) 哈希、派生字段校验、token scope）；C 组大部分已定或后置。
- 本轮收合逐项拍板 16 项（P1–P16），并确认 A1–A9；关键变化是新增 pull 结果的内部导入路径、单包 Skill 推送、回归为空的自动放行、子会话递归规则、缓冲上限与 token scope 的 v1-draft 标注。
- M0 仍未冻结：AMCP 样例未到，待决事项 owner/截止未填，`benchmark_content_sha256` 的 CASE fixture 未附。
- 示例哈希已独立复算：pull 快照、delta 合并、SkillBundle、Proof 与两个 evaluation_id 全部一致；详见 §4.3。

## 1. 本轮拍板结果（2026-11-27）

### 1.1 P1–P16

| 编号 | 决定 | 落点 |
|---|---|---|
| P1 | `traces.pull` 在 Adapter 侧纯读；平台把快照交给接收端内部导入，复用 `traces.push` 合并规则，保留 Adapter revision | 终稿 1.1、2.3.3.8、3.1 |
| P2 | Skill 推送每次恰好 1 个包；未提交的 name 不动；多 Skill 用多次 push，批量留后续版本 | 终稿 2.3.4.5、2.3.4.6 |
| P3 | v1 只支持 SkillBundle；instruction bundle（AGENTS.md 等）留后续版本，独立定义 | 终稿 2.3.4.1、2.3.4.8、四（开放问题） |
| P4 | 父快照只列直接子；工作台/Benchmark 按 `child_sessions` 递归拉取，visited set + 最大深度；跨来源子会话 `readable:false` | 终稿 2.3.3.1、四（验证场景） |
| P5 | 接受 `content_sha256` 每次 push O(n) 重算，配会话/批次上限；链式哈希留后续版本 | 终稿 2.3.3.3 |
| P6 | `approved` 必须 `regressions=[]`；runs 统一且两侧对称；`skill_usage` 不强制 available，但 proof 必须显示覆盖并限制结论措辞 | 终稿 2.3.5、四（首轮实验） |
| P7 | 黑盒评分由平台侧按 Benchmark 格式录入 scoreboard，不走 wire；v1 不进自动 gate | 终稿 2.4.1、四（开放问题） |
| P8 | v1 只 merge；首轮优化器不得删除/改名 Skill；新增用 `base_hashes[name]="absent"`；replace/prune 留后续版本 | 终稿 2.3.4.6、四（开放问题） |
| P9 | 仅出站连接 = collection-only；平台发起的方法不可用；工作台隐藏第二三步自动动作；命令通道留后续版本 | 终稿 2.4.2、3.1、四（开放问题） |
| P10 | 不假设 turn ID 稳定；delta 仅在“已存是当前前缀且 turn 可映射”时用；变化/revert/compact 一律 snapshot | 终稿 2.3.3.2、四（验证场景） |
| P11 | 当前 revision 长期保留；非 pin 默认 30 天可配置；pin 到引用对象删除；工作台只看当前，审计可读旧版 | 终稿 2.3.3.8、四（开放问题） |
| P12 | `local_buffer` 有字节/天数上限（建议 2GB/14 天）；到上限停采并报 `not_captured` gap；不静默丢弃 | 终稿 2.3.8.1 |
| P13 | 默认 404 `session_not_found`；`session_unreadable` 403 仅部署显式开启时返回 | 终稿 2.3.6 |
| P14 | 连接级 token scope 是设计目标，标 v1-draft；M1 实现签发/轮换/迁移；不把现有 admin token + 每 Agent key 说成已实现 | 终稿 2.1 I11、2.2、四（开放问题） |
| P15 | 数据治理章节后置到 v1 之后；终稿只记录为后续事项 | 终稿四（开放问题） |
| P16 | v1 以 AMSP 为唯一规范；AMCP 样例列 M0 阻塞输入；owner/日期暂空；M0 标未冻结 | 终稿四（开放问题） |

### 1.2 A1–A9 确认

| 编号 | 决策 | 状态 |
|---|---|---|
| A1 | 首轮锁定 Codex 真机评测（`codex exec`/App Server + 临时 `CODEX_HOME`，`eval_environment=target_harness`）；runner 交付不了才降级 `platform_mirror` 并同步改文案 | 已确认 |
| A2 | 拆 M2a（协议验证）与 M2b（E1 + E2）；M2 退出 = M2a + M2b | 已确认 |
| A3 | Codex Skill：pull 读 user/repo，push 首轮只写 user；`agent_id` 仅作 RSI 绑定；`trust_state` 表示生效；`capabilities.skills` 含 `scopes`/`writable` | 已确认 |
| A4 | v1 不支持 instruction bundle；若只能改 AGENTS.md，后续版本独立 InstructionBundle | 已确认 |
| A5 | 首轮实验契约：隔离工作区修复预置失败 TS 测试；review-helper/minimal-fix；6–8 case（3 dev + 3 held-out + 2 negative）；同 provider/model/thinking；统一 runs；预冻结阈值与证据清单 | 已确认 |
| A6 | Codex 19 种 ThreadItem 逐项映射到 AMSP payload 或丢弃+gap；`tool_call_id` 配对；NormalizedSessionMeta；Adapter 组件；golden fixture | 已确认 |
| A7 | Hook 信任与实时触发 B1（App Server 通知流）/B2（command Hook + `/hooks` 信任）；HookStatus 含 `trust_status`/`current_hash`；不可用时降级 pull-only | 已确认 |
| A8 | delta 仅在可证明前缀 + turn 可映射时使用；变化/插入/revert/compact 一律 snapshot；H 必测；首轮允许 snapshot-only | 已确认 |
| A9 | 定义 `benchmark_content_sha256`/`evaluation_id` 算法与向量；proof builder；Codex runner；候选 Skill out-of-band 安装；E1/E2 拆分 | 已确认 |

## 2. 评审清单 10 条去向

| # | 议题 | 当前状态 | 落点 / 差异 |
|---|---|---|---|
| 1 | 首轮闭环 | 已落实 | 终稿四（首轮实验）：任务、Skill、Benchmark、runner、runs、阈值、证据清单 |
| 2 | Session 拉取 | 已落实 | 双路径 + P1 内部导入；pull 纯读、push 主动推送 |
| 3 | Trace 用途 | 已落实 | 终稿 1.2、2.3.8.2：三层证据；单条 trace 不证明效果 |
| 4 | 子 Agent | 已落实（本轮补规则） | P4：父只列直接子；递归拉取带 visited/深度；不可读子保留引用 |
| 5 | 发现、分页与去重 | 已落实 | `traces.list` 页码分页、无公开 cursor；身份 + revision；合并规则 |
| 6 | Hook、范围与来源 | 已落实 | 连接绑定 Project，不逐次传 scope；`raw_path`；三种连接形态 |
| 7 | Skill 与评测 | 部分落实 | SkillBundle/Proof/帕累托/人工放行已写；AMCP 映射仍待样例（M0 阻塞）；merge 由“并集”改为单包 + 多次 push（P2） |
| 8 | 适配范围及配套说明 | 已落实 | 删除 Cursor；方法表/错误恢复/工作台已更新；HTML 待重生成 |
| 9 | API 只能返回部分信息 | 已落实 | 五域四态 + gaps + sync；unknown 不写空；Skill 证据分层 |
| 10 | 离线保证 | 已落实（本轮补上限） | 五档矩阵；P12 本地缓冲上限与缺口报告 |

## 3. 评审报告 35 项决策去向

### 3.1 A 组（首轮）

| 编号 | 决策 | 状态 |
|---|---|---|
| A1 | Codex 真机评测 | 已确认 |
| A2 | M2a/M2b 拆分 | 已确认；M2a 附加检查见终稿四（验证场景）；M2b = E1+E2 |
| A3 | Codex Skill 作用域与生效验证 | 已确认 |
| A4 | instruction bundle 边界 | 已确认；后续版本 |
| A5 | 首轮实验契约 | 已确认；runs 统一按 P6 |
| A6 | ThreadItem 映射与 NormalizedSessionMeta | 已确认 |
| A7 | Hook 信任与 B1/B2 | 已确认 |
| A8 | turn ID 与 snapshot fallback | 已确认；按 P10 |
| A9 | proof/runner/E1/E2 | 已确认 |

### 3.2 B 组（协议与治理）

| 编号 | 决策 | 状态 / 落点 |
|---|---|---|
| B1 | 可见性 revision 级，升级/降级都发完整 snapshot | 已写入 2.3.3.7、2.3.8.1 |
| B2 | pin 旧 revision 读取路径；pull 纯读 | 已写入 2.3.3.8；P1 补内部导入 |
| B3 | SyncState 与 DeliveryStatus 拆分；pending_unit/stale | 已写入 2.3.2、2.3.8.1 |
| B4 | metadata_sha256 幂等键与字节原样重发 | 已写入 2.3.3.3、2.3.3.6 |
| B5 | payload 级允许表与脱敏 | 已写入 2.3.3.2；`session_meta` 改条件保留 |
| B6 | native/normalized 两层与 source 映射 | 已写入 2.3.3.1 |
| B7 | Proof 完整契约、override 对象、逐题回归 | 已写入 2.3.5；override 移除自引用 `proof_sha256` |
| B8 | 自动放行 predicate、runs 可比、held-out | 已写入 2.3.5、四（首轮实验）；P6 补 `regressions=[]` 与统一 runs |
| B9 | skills.push 幂等顺序与 request_id | 已写入 2.3.4.5、2.3.4.6 |
| B10 | 写回提交协议、字节保真、mode、大小上限 | 已写入 2.3.4.3、2.3.4.6、2.3.4.7 |
| B11 | content_sha256 增量性 | 默认：接受 O(n) + 上限；链式哈希后续版本（P5） |
| B12 | 派生元数据校验 | 部分：机械校验与 summary 域依赖已补；`skill_usage=available` 的证据仍偏启发式 |
| B13 | AMCP 与 AMSP 规范关系 | 待决：AMSP 唯一规范；AMCP 样例 M0 阻塞（P16） |
| B14 | 分层冻结与 v1-draft | 已写入四（开放问题） |
| B15 | canary 语义与 backup/rollback | 已写入 2.3.4.5、2.3.4.6；canary 是审计窗口，不是流量切分 |
| B16 | 仅出站 collection-only 与 Hook 路径 | 已写入 2.4.2、3.1（P9） |
| B17 | 连接级 token scope | 部分：设计目标 + v1-draft；当前实现仍 admin token + 每 Agent key（P14） |

### 3.3 C 组（后续）

| 编号 | 决策 | 状态 |
|---|---|---|
| C1 | 历史 revision 保留期 | 默认 30 天 + pin 保护（P11）；owner 待指派 |
| C2 | 数据治理：保留/删除/导出/脱敏/Project 隔离 | 后置到 v1 之后（P15） |
| C3 | 子会话递归与跨来源 | 已定：工作台递归、visited/深度、跨来源 `readable:false`（P4） |
| C4 | 交付粒度与本地缓冲 | 已定：turn 级 + 缓冲上限 + `not_captured` gap（P12） |
| C5 | 黑盒评分进入平台 | 已定：平台侧 scoreboard，不走 wire，不进自动 gate（P7） |
| C6 | HTML 同步 | 待办：终稿定稿后重生成；owner 待指派 |
| C7 | 不存在与无权读 session 的返回 | 默认 404，403 需显式开启（P13） |
| C8 | 一批多 session 推送 | 非目标：一请求一 session；Adapter 可并发，但同一 session 串行 |
| C9 | replace/prune | v1 只 merge；replace/prune 后续版本（P8） |

## 4. 本轮核查发现与修正

### 4.1 高风险问题

| # | 问题 | 修正 |
|---|---|---|
| 1 | M2a/M2b 引用未定义的 `L1/I1/S1/H1/S2/P1/C1/I/J` | 终稿新增四（验证场景） 定义七项检查；M2b 统一为 E1+E2，删除 I/J |
| 2 | 正文引用 Q10，但 Q 表无 Q10；Q1/Q3/Q4/Q8/Q12 缺失 | 终稿不再使用 Q 编号；本备忘 §5 补全 Q1–Q14，Q10 由 Codex Skill 作用域决策关闭 |
| 3 | `traces.pull` 纯读与“两路汇入同一存储/工作台导入”冲突 | P1：新增接收端内部导入路径；pull 仍纯读，导入复用 push 合并规则 |
| 4 | `score_strict` 自动放行未要求 `regressions=[]` | P6：`approved` 与 Adapter 校验都要求回归为空 |
| 5 | `sourceKinds` 违反 snake_case | 协议字段改 `source_kinds`；Codex `sourceKinds` 由 Adapter 映射 |
| 6 | `session_meta` 允许表与示例/native 表冲突 | 改为“最小字段 + 按 messages/raw_log 条件保留”；示例哈希仍成立 |
| 7 | 合并规则第 12 行“晚到 pull”与纯读矛盾 | 删除该行；pull 不入库，导入走内部路径 |
| 8 | `subagent_tokens`、`normalization.omitted_items`、`agent_state_version`、`connection` 未定义 | 删除 `subagent_tokens`；`omitted_items` 定位为 Adapter 日志；`agent_state_version` 定义；`connection` 结构固定 |
| 9 | 工作台“三步 / 三标签 / 五标签”混用 | 终稿 3.1 增加“三步 ↔ 三标签”映射表 |
| 10 | I11 token scope 写成既有事实 | 标 v1-draft，写明当前实现是 admin token + 每 Agent key |
| 11 | 数据治理章节缺失 | P15：后置 v1 之后，终稿四（开放问题） 记录 |
| 12 | workbench 包在本仓库不存在 | 标为 PR #1899 外部依赖，M3 前核实 |
| 13 | 四（验证场景）中的 N+1→`revision_gap` 与规则 1 冲突 | 改为 `stale_ignored` |
| 14 | `override.proof_sha256` 自引用 | 从 override 对象移除；外层 `proof_sha256` 覆盖 |
| 15 | `backup_before_write=false` 与 canary/full 强制 true 冲突 | v1 只接受 true；传 false 返回 400 |
| 16 | `visibility` 必填与缺键按 unknown 冲突 | 整对象缺失 400；对象内缺域键才按 unknown |
| 17 | 每 case 2–3 runs 与单个 `runs_per_case` 不匹配 | v1 要求同一 Benchmark 内统一且两侧对称的 ≥2 整数 |
| 18 | `benchmark_content_sha256` 未附 CASE fixture，无法复算 | 终稿附录 C 记录为待补测试向量 |
| 19 | session/skill 哈希无测试向量 | 终稿附录 C 记录；M1 conformance 补 |
| 20 | A1–A9/P0 写“已拍板”但无会议记录 | 本备忘 §1 记录 2026-11-27 逐项确认；终稿不再写“已拍板” |
| 21 | M0 退出条件要求 owner/截止，但全部待指派 | 终稿四（开放问题） 明确 M0 未满足；本备忘 §5 标 owner |

### 4.2 保留的已知缺口

- AMCP 与 AMSP 的映射未完成（Q7）；AMCP 样例是 M0 阻塞输入。
- `benchmark_content_sha256`、session/skill 哈希的测试向量未全部附 fixture。
- `skill_usage=available` 的“已检查”仍依赖 Adapter 声明，缺少 wire 级 `checks` 字段；当前以 `skill_evidence` 与 gaps 约束。
- 数据治理、连接级 token scope、HTML 复核均为后续/未实现项。
- 工作台包 `@penguinharness/rsi-workbench` 0.2.0 未在当前 worktree 核实。

### 4.3 示例哈希复算

独立脚本按文档算法复算，全部与文档值一致：

- `traces.pull` 快照：`content_sha256`、`metadata_sha256`。
- `traces.push` delta：7 条已存记录 + 6 条 `added_records`，合并后 `content_sha256`。
- `SkillBundle` v0：三文件按 `rel_path` 排序，`content_sha256`。
- `VerificationProof`：`proof_sha256`、baseline/candidate 的 `evaluation_id`。

`benchmark_content_sha256` 与 v1 SkillBundle 摘要因缺 CASE fixture/原文未能复算，见终稿附录 C。

## 5. 待决事项 Q1–Q14

| 编号 | 事项 | 状态 | 默认/结论 | owner | 时限 |
|---|---|---|---|---|---|
| Q1 | 子会话是否递归拉取、不可读如何表示 | 已关闭 | P4：父不内联；工作台递归；不可读保留引用 + reason | — | — |
| Q2 | 非 pin 历史 revision 保留期与删除 | 待决 | 默认 30 天、可配置；pin 到引用对象删除 | 待指派 | M1 |
| Q3 | 运行中推送的实时粒度 | 默认 | turn 级；token 级留后续版本 | 待指派 | M2 |
| Q4 | 一次性数据是否要求外围落盘、缓冲上限 | 已关闭/默认 | `source_replay=false` 强制 local_buffer；P12 缓冲上限 | 待指派 | M2 |
| Q5 | 黑盒评分如何进入 scoreboard/proof | 待决/默认 | 平台侧录入，不走 wire；v1 不进自动 gate | 待指派 | M0 |
| Q6 | HTML 展示稿何时、由谁同步 | 待决 | 终稿定稿后重生成；M3 复核 | 待指派 | M3 |
| Q7 | AMCP 样例与 AMSP 映射 | 待决 | v1 以 AMSP 为唯一规范；样例列 M0 阻塞 | 待指派 | M0 |
| Q8 | 不存在与无权读 session 是否统一 404 | 已关闭/默认 | 默认 404；403 仅显式开启 | — | M1 前确认 |
| Q9 | instruction v1.1 的路径/merge/hash/rollback | 待决 | v1 不做；后续独立 InstructionBundle | 待指派 | 后续版本 |
| Q10 | Codex 用户级/仓库级 Skill 作用域 | 已关闭 | 由 A3/P3 覆盖：Adapter 声明来源作用域，不是 RSI 全局作用域 | — | — |
| Q11 | push-only 命令通道 | 待决 | v1 不做；命令通道留 v2 | 待指派 | M0 |
| Q12 | 一请求多 session 批量推送 | 已关闭 | 非目标：一请求一 session；同一 session 串行 | — | — |
| Q13 | turn ID 跨重连/fork/revert/compact 稳定性 | 待决 | 不假设稳定；变化即 snapshot；四（验证场景）必测 | 待指派 | M2 |
| Q14 | replace/prune | 待决 | v1 不引入；后续需删除清单 + 逐目录 hash + dry-run + 整包回滚 | 待指派 | v2 |

## 6. 文件与归档

| 文件 | 角色 |
|---|---|
| `docs/rsi-api-design.md` | 当前主 Design 文档（沿用 V1 骨架 + 金字塔原则精简，面向团队方案评审） |
| `docs/rsi-api-decision-notes.md` | 本备忘：决策、评审去向、核查证据 |
| `docs/archive/rsi-api-architecture-v1.md` | 初稿归档 |
| `docs/archive/rsi-api-architecture-v2.md` | V2 技术底稿归档 |
| `docs/archive/rsi-api-architecture-v3.md` | V3 完整技术规范归档（含完整 JSON 示例、哈希算法测试向量与 Codex 19 项映射） |
| `docs/archive/rsi-api-review-notes-2026-11-27.md` | 评审清单归档 |
| `docs/archive/rsi-api-architecture-v2.html` | 旧展示稿归档 |
| `docs/archive/rsi-api-architecture-reading.html` | 旧阅读版归档 |

## 7. 证据来源

| 来源 | 用途 |
|---|---|
| `D:/tmp/rsi-discussion-digest.md` | 评审清单 10 条逐项结论、仓库与 Codex 事实、设计建议、待决问题 |
| `D:/tmp/rsi-v2-design-brief.md` | 设计简报：覆盖表、事实分级、逐节写作要求 |
| `D:/tmp/rsi-v2-critique.md` | 独立评审：20 条 findings、35 项决策（A/B/C） |
| `D:/tmp/rsi-v2-rewrite-brief.md` | A1–A9 与 P0 修正的落点表 |
| `D:/tmp/rejected-v2.md` | 早期 V2 草稿（未采用） |
| `D:/tmp/rsi-v2-a-group-backup.md` | A 组决策前置的 V2 备份 |
| `D:/tmp/codex-msgs-full.md`、`D:/tmp/codex-schema/` | Codex 0.172.0 实测与 schema 证据 |
| 本轮核查 workflow | 评审覆盖、内部一致性、V1→V2 变化、对抗核查 |
