# 实现规划：23 天有界 MVP（拓扑 B）

> 状态：已选定设计，待原型和实现验证。23 天是四人团队的实施预算，不是完整生产级交付保证。技术细节参考 [agent-runtime.md](agent-runtime.md)、[sandbox-tracks.md](sandbox-tracks.md)；验收门禁见 [feasibility-review.md](feasibility-review.md) §6。

## 一、首版范围与组件选型

首版限定：一个明确支持的无本地持久状态 HTTP 应用、单 Linux LTS/单架构、一个通过隔离门禁的工具沙箱与构建执行器组合、一个 CI 错误自修场景。默认迁移模式 `none`；如演示持久数据，数据库独立，兼容性与备份恢复不可省略。

后置：第二后端完整交付、Go/Node/Spring 三语言同时覆盖、五枚云端 Skills、值守 Agent、复杂迁移、多架构、任意项目自动改造、完整离线部署。HTTP/容器契约保持语言无关，不将这一点等同于已支持任意应用。

| 组件 | 选型 | 职责与边界 |
| --- | --- | --- |
| Agent 主循环 | KSADK + LangGraph / AgentEngine | 受信编排、模型调用、持久会话；不执行任意代码 |
| 固定工具适配器 | 与运行时集成的受信代码 | 服务身份、会话映射、参数与预算；shell 远程进入沙箱 |
| Proposal Broker / MCP | Go 服务 + MCP 适配 | 四动作、GitHub App、授权、审计；不执行仓库代码 |
| SandboxProvider | 平台 SDK 适配 | KCI 主线；local 为通过门禁后的可选后端 |
| BuildExecutor | Go 控制端 + 独立临时环境 | 不可变快照、固定 preset、资源控制、取消和回收 |
| Deployer | Go CLI + systemd timer | manifest 全量校验、蓝绿状态机、journal、本地恢复 |
| Bootstrap / GitHub setup | Bash / Python + gh/API | 受信管理员执行，初始化主机与真实门禁配置 |
| 验收 | Python pytest + Go tests | API/网络/文件边界、故障注入和证据包 |
| 生产代理 | Caddy | 固定模板、受限管理口、切流、TLS/CORS 档位 |

优先团队熟悉的 Go，不维护 Go/Rust 两套实现。Rust 仅作为有经验团队的互斥备选。Go 依赖拟用 go-github、JSON Schema 校验库、log/slog、chi、testify，实施时锁定版本；先用固定 argv 调受控 Compose CLI，不引入通用 shell 执行接口。

## 二、应用契约

| 契约 | 黑盒验收 |
| --- | --- |
| GET /livez | 进程存活，不依赖 DB |
| GET /readyz | 依赖就绪，503/200 语义正确 |
| GET /version | source_sha、build_id，不输出环境和秘密 |
| SIGTERM | 先置未就绪，处理在途请求，有限时间退出 |
| HTTP 超时 | 读头/请求/空闲及外部依赖超时 |
| 双槽位兼容 | N/N−1 可读写当前 schema，业务 smoke 无破坏性 |

先交付一个应用模板。内存会话、WebSocket/长连接、队列消费者与定时任务双跑需专项设计；未验证则不纳入支持范围。数据库和持久文件不随槽位清理，候选不能挤占旧版本生存资源。

## 三、拓扑 B 与 Proposal Broker

运行时持有模型/平台调用所需身份；固定适配器通过 mTLS 服务身份 + 短期会话材料调用 Broker。材料不进入模型上下文或 guest。Broker 根据服务端会话决定 app 仓库、基线、PR、workspace/snapshot 与预算，不接受模型选择权限。平台 MCP 网关如参与认证终止，纳入可信边界并先验证会话隔离。

```text
broker/
  internal/api/        # repo.read / proposal.submit / ci.read / sandbox.build
  internal/auth/       # 服务身份 + 短期会话、撤销、资源归属和配额
  internal/schema/     # 固定 Schema、嵌套 additionalProperties=false、大小限制
  internal/policy/     # 路径、Git tree/模式、最终 diff、控制文件保护
  internal/gitops/     # API 写 blob/tree/commit/ref、受管分支/PR 幂等
  internal/cifilter/   # 当前 PR/head/workflow/run 关联、日志截断与脱敏
  internal/buildsched/# 不可变快照、作业预算、取消/回收
  internal/audit/      # 意图/结果、规范化 JSON 哈希链、持久 outbox 与外部锚定
  internal/token/      # App installation token 缓存/轮换/撤销
```

策略要求：

- GitHub App 仅安装 app 仓库，按需 Contents/PR 写、Actions 读，无管理和 bypass。GitHub token 不天然限分支、也不天然无 merge 能力；真实测试平台规则与 Broker 策略，不用工具缺失冒充 App 权限缺失。
- patch 首版仅支持普通 UTF-8 文件新增/修改/删除；拒绝 symlink、gitlink、二进制和模式变化。精确基线应用后验证最终树，禁止 hook/filter/仓库脚本在 Broker 环境执行。
- `.github`、CODEOWNERS、发布和安全策略路径仅可提交文本建议。不能为了“自动修复 CI”扩大控制面权限。
- 会话固定基线；更新 PR 需要预期 head、归属和并发检查，main 漂移时重新核对。API 超时先查询已有副作用，再决定重试。
- JSON 入口拒绝重复键；规范化格式固定并用跨语言向量验证，不把普通 JSON 序列化当作规范化。
- 哈希链用于篡改检测，不是不可篡改保证。关键写操作先持久审计意图；远端不可达写 outbox，本地失败/积压达上限则拒绝新写并告警。

## 四、SandboxProvider 与 BuildExecutor

两者独立接口，详见 [sandbox-tracks.md](sandbox-tracks.md)。工具沙箱不直连 Broker/模型/平台管理 API；受信控制端交换文件、执行命令并冻结快照。Broker 只接受已登记、归属正确的不可变 snapshot_id，不接受 URL 或宿主目录。

BuildExecutor 在专用非生产节点/云端独立临时环境中构建；guest 无模型、平台、Broker、GitHub/生产凭据和宿主 socket。KCI 构建不满足要求时使用临时 VM，不为跑通而加 privileged 或开放任意网络。

构建优先预置依赖离线执行；需下载时单独无秘密阶段经受控代理，随后离线构建。缓存分信任域，限制 DNS、解析后地址、重定向、IPv6 和内部网络。rootless 不替代网络、PID、挂载及资源验收。

作业协议包含幂等键、session/snapshot、preset、CPU/内存/PID/磁盘/输出预算和 deadline。超时真正终止作业并 inspect；服务重启回收孤儿资源。测试 shell 与 Dockerfile RUN 两层边界。沙箱镜像仅用于反馈，正式候选由 main CI 重建。

## 五、CI、发布授权与独立人类门禁

1. PR checks 在独立无生产秘密的临时 runner 执行；不使用高权限 `pull_request_target` 运行 PR 代码，不在生产机跑不可信 runner。
2. 维护者审查合并精确 app 提交；main CI 构建，隔离推送 job 只处理固定产物、不执行 app 脚本，产生 source_sha/run/artifact/digest 绑定及扫描元数据。
3. 维护者在 ops 仓库提发布请求 PR，固定候选与迁移策略；release workflow 验证来源，生产审批 job 不执行 app 代码。
4. 实际 GitHub 账号配置 required reviewers、防自审批、禁止 bypass 和固定分支；未批准时不能产生有效 manifest。套餐/可见性以真实账号与当前官方文档为准。
5. 批准后发布 manifest。Release 资产不是天然签名对象；主机按固定 repo/workflow/head_sha/run_attempt 核验实际发布 job 成功，并按不可变 ops commit 逐项绑定请求内容与可信 workflow/脚本版本。
6. 只消费固定 registry/repo@digest，不接受远端 command/volumes/secret_path/Compose；首次接受前篡改、接受后同 ID 异 hash、跳过 job、过期/旧 generation 都需拒绝。

`generation` 对应固定发布 workflow 的可信 run_number；rerun 不自动变成新序号，失败后重发使用新的获批 run。更换 workflow/仓库需要受信序号迁移策略。自动回滚是当前事务补偿，不降低高水位。多架构 digest 类型不同，首版限定单架构。

## 六、Deployer：状态、持久化和恢复

```text
Validated → Prepared → CandidateReady → Switched → Committed
                  失败：清理候选         失败：RollingBack → RolledBack
                                                        └→ ManualRequired
```

组件：manifest/authorizer/contentbind/preflight/statemachine/journal/caddy/bluegreen/lock。单主机 flock 串行所有入口；先协调 pending 再接受新发布。

- 每步副作用前写意图、后写结果；同目录临时文件 + fsync + rename + dirsync。pending、active、previous、generation 和失败记录一致提交；损坏 journal 不自动重置。
- Compose 模板、secret 挂载、资源与 Caddy 配置由宿主受信策略决定。Deployer 拥有高权限宿主能力，不能声称非 root UID 即已最小权限。
- 固定 blue/green 两项目和 `127.0.0.1:18080/18081`，数据库独立。按固定 argv/env/工作目录调用工具，无 `sh -c` 拼接；限制进程树、输出和超时。
- Caddy：验证配置 → 记录切流意图 → 原子替换 → reload → 从真实代理入口检查版本/业务。磁盘文件不等于运行态；管理口只供受信本地控制面。
- 观察窗口初始 60–120 秒。切流前失败仅清理候选；切流后按故障类别恢复旧配置与应用并复验；旧版本也失败则 ManualRequired。首次部署无 previous，失败不能记作回滚成功。
- 在 reload 成功但 journal 未写、提交前、回滚中等窗口 SIGKILL，重启结合 journal、代理运行态、容器 digest/标签和真实入口恢复。SIGKILL 不等同于掉电耐久性证明。
- 重复当前成功 manifest 核对实际状态后返回 ALREADY_APPLIED，零新增部署副作用；失败版本隔离，不因 timer 无限重试。
- 保留旧镜像和资源，本地恢复不依赖 AgentKit/GitHub 在线。回退 ≤120s 从“发现故障”计到“旧业务恢复”，另记录发现延迟，不推导长期 SLA。

## 七、事件、部署档位和数据

### 事件自修

notifier 验签、delivery_id 去重、持久队列、快速应答；按 session 串行且核对 PR/head/workflow/run，忽略过期事件。运行时配置持久会话与 graph checkpoint，验证重启恢复与工具幂等。最大轮数、时间、模型费用和构建次数由受信配置限定，超限转人工待办。只对首版明确错误类型承诺自修。

独立人类告警不等待 AgentKit；值守 Agent 后置。模拟平台故障时验证已有合法授权的部署、本地回滚、人类告警，而不只检查存量服务还活着。

### 部署档位

宿主受信 `/etc/student-app/profile.conf` 管理 `tls_mode=http/internal/auto`、`cors_mode=open/allowlist`、访问方式；不允许 Agent/manifest 修改。

- http 仅无敏感数据的受控演示；internal 要求客户端显式信任内部 CA，不关闭 TLS 校验；auto 需域名和签发条件。
- CORS open 仅用于不携带跨域凭据的场景；统一 OPTIONS、允许头/方法和去除重复响应头。CORS 不是鉴权，不保证任意应用零改动。
- 档位进入 journal/告警；切换正式配置需验证浏览器与业务兼容，而非只改一行即视为成功。

### 数据

默认 `none`；扩展到 `expand` 时，迁移由受信入口使用独立 DDL 账号执行已审核的固定内容，具备 checksum、锁、超时与 N/N−1 兼容测试。新旧槽位不得自动分别迁移；应用回滚不撤销数据库写入、外部调用和不可逆操作。

使用持久数据就必须做一致性备份、异地副本、空白实例恢复。RPO≤24h/RTO≤60min 是待实测目标，不能由发布回退时间推导。

## 八、23 天排期（四人，早验证、单线收敛）

A 负责示例/验收与运行时流程，B 负责 CI/审批/初始化，C 负责 Deployer，D 负责 Broker/执行底座；A、B 分担 MCP/事件集成，避免 D 同时承担全部工作。

| 阶段 | A 应用/Agent | B CI/审批 | C Deployer | D Broker/执行底座 | 门禁 |
| --- | --- | --- | --- | --- | --- |
| D1–3 | 一个应用契约；AgentEngine/模型薄调用 | 真实 App/Environment、MCP 认证协作 | Schema/journal 冻结，授权与篡改 fixture、Caddy 原型 | Broker 只读薄链路；真实构建+隔离 PoC | F01–F05；不足则停对应路径，不开放权限 |
| D4–7 | shell→改代码→构建→PR | PR/main/digest、ops 审批最小流 | 真实授权→候选→切流；不完整恢复仅限测试环境 | 四动作最小安全实现、快照/归属/审计 | 单应用最小端到端链路 |
| D8–11 | 一种错误注入与修复 | notifier 去重/队列、权限负例 | 回滚、幂等、真实代理复验 | 资源/取消/销毁、patch 最终树验证 | F06、候选失败不影响旧服务 |
| D12–15 | 会话重启/预算；持久数据时恢复演练 | bootstrap/setup 幂等、独立告警 | journal 崩溃窗口与本地恢复 | 审计外部锚定、孤儿回收、隔离负例 | F07–F09；适用时 F10 |
| D16–18 | 全链路加固，不再首次接入 | 授权与事件攻击测试 | 故障注入、回退计时 | 资源/身份负例复测 | 未过项停止扩展 |
| D19–20 | 固定样本验收与证据包 | 同左 | 同左 | 同左 | 保存原始失败和重试 |
| D21–23 | 彩排、文档、缓冲、演示 | 同左 | 同左 | 同左 | 明确已实现/未验证/后置项 |

## 九、完成定义

- 一个应用与一个合格执行后端组合跑通真实模型/MCP/Broker/构建/PR/CI 修复/人工审批/发布链路。
- 未批准、越权、伪造/重放发布必测用例全部拒绝，留 API/网络/文件证据；有限测试不证明绝对安全。
- 固定 20 次正常发布至少 19 次成功；定义故障演练有明确结果；回退≤120s、重复 manifest 零新增部署副作用、关键崩溃窗口收敛。
- 模型上下文、沙箱/构建环境、工具输出、Trace/日志、镜像层的规定凭据样本扫描零命中；受信组件合法用凭证不算违规，不宣称进程内存零秘密。
- CI 固定故障在预算内修复，重复/过期事件不产生重复写；平台中断不阻塞独立发布恢复与人类告警。
- 如含持久数据，必须提交独立恢复证据；若不含，标注 DATA/备份项不适用，不声称已达成 RPO/RTO。
- 第二后端、其他语言、云端 Skills 完整能力尚未实现则明确后置；不使用“全部完成”覆盖未测范围。
