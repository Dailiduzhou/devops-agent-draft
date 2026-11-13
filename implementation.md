# 实现规划：23 天黑客松完整版

完整实现 idea.md 架构（Broker 服务、Deployer 状态机、获批发布协议、Agent 沙箱），应用语言不限（Go / Node / Spring Boot…），反向代理用 Caddy，无域名/备案/临时 CORS 等现实约束用**部署档位机制**显式管理。

## 一、语言与组件选型

| 组件 | 语言 | 说明 |
| --- | --- | --- |
| Proposal Broker | Go（完整服务） | 四动作白名单 API、策略引擎、哈希链审计、GitHub App token 管理 |
| Deployer | Go（完整状态机） | manifest 全量校验、authorizer、蓝绿双槽位、journal、自动回滚、崩溃恢复 |
| 沙箱构建执行器 | Go | 独立进程，Broker 调度，隔离网络与资源 |
| Bootstrap | Bash | 主机初始化 + 环境检测 + registry 连通性 |
| setup_github | Python（gh CLI/API 幂等配置） | 分支保护、rulesets、CODEOWNERS、Environment |
| 攻击/验收测试 | Python pytest | SEC/DEP/DATA/OPS 全用例 + 证据包 |
| 应用本身 | 任意语言 | 流水线对应用语言零假设，只依赖 HTTP 契约 + 容器契约 |
| 反向代理 | Caddy | 自动 HTTPS / tls internal 自签，切流即改 upstream + reload |

Go 依赖（固定版本）：google/go-github、santhosh-tekuri/jsonschema、log/slog、chi、testify。不引 Docker SDK（exec 调 compose CLI）、不引 K8s。

## 二、应用契约（语言无关）

| 契约 | 验收（黑盒） |
| --- | --- |
| GET /livez | 进程存活，不依赖 DB |
| GET /readyz | 依赖就绪检查，503/200 语义正确 |
| GET /version | 返回 source_sha、build_id，无环境变量泄露 |
| SIGTERM | 先置未就绪 → 处理在途请求 → 退出；STOPSIGNAL + stop_grace_period 25s |
| HTTP 超时 | 读头/请求/空闲超时，外部调用带 context/timeout |

提供 Go / Node(express) / Spring Boot(actuator) 三份契约实现模板 + Dockerfile 模板；Agent 探测技术栈后选模板改造应用。演示用两个不同语言应用走同一条流水线。

## 三、Proposal Broker（完整实现）

```
broker/
  cmd/broker/
  internal/api/        # repo.read / proposal.submit / ci.read / sandbox.build 四个 handler
  internal/auth/       # 沙箱会话 token：绑定沙箱实例、短时、限速、总量
  internal/schema/     # 严格 JSON Schema：additionalProperties=false、长度/枚举限制
  internal/policy/     # 路径规范化：拒绝 ../绝对路径/符号链接逃逸/.git/.github/CODEOWNERS/策略文件；内容允许列表
  internal/gitops/     # go-github：agent/ 分支创建、提交、开 PR；不 bypass ruleset
  internal/cifilter/   # CI 日志过滤：去 token 模式/环境变量 dump，白名单正则
  internal/buildsched/ # 调度沙箱构建执行器：固定参数、超时、资源、网络禁令
  internal/audit/      # 哈希链审计：hash=SHA256(prev‖canonical_json)；链头外发独立日志端
  internal/token/      # GitHub App 短期 installation token 签发/缓存/撤销
```

安全语义要点：
- Agent 不持有任何 GitHub token；App 私钥只在 Broker 侧。
- proposal.submit 只接受 base_sha + patch；分支名由 Broker 生成。
- sandbox.build 在独立无秘密执行器上跑，绝不碰宿主 Docker socket。
- 每条审计含 request_id、actor、参数摘要、策略版本、允许/拒绝原因。

## 四、Deployer（完整状态机）

```
deployer/
  cmd/deployer/
  internal/manifest/    # JSON Schema 校验 + generation/expiry/digest 格式
  internal/authorizer/  # GitHub API 核验：repo ID、workflow ID、head_sha、run conclusion、run_attempt、指定 release job 实际成功
  internal/contentbind/ # 首次接受时按 run 的不可变 head_sha 读取 requests/prod.json 与 manifest 逐项比对
  internal/preflight/   # 磁盘/inode/内存/端口/Docker/时间同步/证书
  internal/statemachine/# Validated→Prepared→CandidateReady→Switched→Committed + Failed/RollingBack/RolledBack/ManualRequired
  internal/journal/     # 意图-结果持久写：tmp+fsync+rename+dirsync；启动 reconciliation
  internal/caddy/       # 固定 Caddyfile 模板渲染 upstream → caddy validate → 原子替换 → reload；失败恢复磁盘旧文件
  internal/bluegreen/   # student-app-blue/green 双槽位，127.0.0.1:18080/18081，幂等（release_id+manifest_hash 重复→ALREADY_APPLIED）
  internal/lock/        # 主机级 flock；手工 rollback 走同一入口
```

全量核验保留：release_id 命名、generation 与可信 run_number 一致、manifest 哈希首次接受后锁定、同 release_id 内容变化拒绝、失败版本隔离不无限重试。

## 五、发布协议（完整）

1. app main 构建成功 → 镜像按 digest 推 GHCR → candidate.json（source_sha、build_run_id、artifact_id、image_digest、SBOM、扫描引用）。
2. 维护者在 ops 提交发布请求 PR（候选 run_id、source_sha、digest、迁移策略）。
3. release workflow 经 API 验证候选来源（仓库/workflow/分支/精确提交/digest 实际推送）。
4. production Environment 人工审批（展示 diff、digest、迁移影响、回退目标）。
5. manifest 以 GitHub Release JSON 资产发布，绑定 ops commit、run_id、run_attempt。
6. 主机 deployer 仅出站读取，完成全部核验后进入状态机。

## 六、部署档位（临时不完善行为的显式管理）

档位只存在于**宿主受信配置**（/etc/student-app/profile.conf），不进 manifest、不进镜像、Agent 改不了：

| 档位 | 临时值 | 升级路径 |
| --- | --- | --- |
| tls_mode | `http`（裸 IP 演示）/ `internal`（Caddy 自签） | 有域名后改 `auto`（Let's Encrypt 自动签发） |
| cors_mode | `open`（Caddy 层注入 ACAO:*，应用零改动） | 改 origin 白名单，纯配置切换 |
| access | `ip_port` | 域名解析 + tls auto |

- CORS open 在 Caddy 层注入 header，不写死进应用代码，升级时应用不动。
- journal 每次部署记录当前档位；tls_mode=http 或 cors_mode=open 时外部探活输出醒目警告。
- 交付"临时态清单 + 升级开关位置"一页，作为工程判断力展示。

## 七、数据库迁移

- MVP 支持 `none` 与 `expand`（只增可空字段/新表），迁移锁 + checksum + 语句超时。
- 迁移由受信入口以独立 DDL 身份执行；候选/旧槽位不得各自启动迁移。
- 备份：每日一致性备份 + 异地一份；验收含一次空白实例恢复实测（RPO≤24h，RTO≤60min）。

## 八、23 天排期（4 人并行）

| 天 | 应用线 A | CI/审批线 B | Deployer 线 C | 沙箱/Broker 线 D |
| --- | --- | --- | --- | --- |
| D1–2 | 契约模板×3 语言、compose 栈 | bootstrap.sh + setup_github、双仓库/保护规则 | 接口冻结（manifest schema、journal 格式） | 威胁模型、Broker API 设计、策略清单 |
| D3–5 | 2 个语言应用改造 + 本地 smoke | PR checks + main 构建 + GHCR digest 绑定 | manifest 校验 + authorizer + timer 轮询 | 沙箱容器化（无凭据、网络禁令） |
| D6–8 | 迁移文件 + 测试 | candidate.json + SBOM + 扫描 | preflight + journal + 原子写 | repo.read + ci.read + 日志过滤 |
| D9–11 | 端到端本地联调 | ops release workflow + Environment 审批 + Release 资产 | Caddy 切流 + 双槽位 + 幂等 | proposal.submit + 路径策略 |
| D12–15 | 备份脚本 + 恢复演练 | 发布请求工具化 | 状态机全量 + 自动回滚 + 崩溃恢复 | sandbox.build + 构建执行器 |
| D16–18 | 攻击测试支持（canary、接收端） | 审批演示素材 | DEP 故障注入自测 | 哈希链审计 + Agent 全链路接入 |
| D19–20 | 全量 SEC/DEP/DATA/OPS 用例跑通 + 证据包 | 同左 | 同左 | 同左 |
| D21 | 端到端彩排 ×2：陌生语言应用 fork→上线计时 | | | |
| D22 | 文档、临时态清单、演示脚本 | | | |
| D23 | 缓冲 + 演示 | | | |

关键顺序约束：D1–2 冻结 manifest schema 与审计格式（B/C/D 三线的契约）；Agent 第 16 天接入时控制面（CI+审批+deployer）已可用。

## 九、验收最小集（黑客松展示）

- 权限越界：SEC01（push main 被拒）、SEC07（沙箱读宿主/元数据被拒）、SEC09（伪造 manifest 被拒）、SEC10（重放旧 generation 被拒）
- 故障恢复：DEP01（候选失败旧服务持续可用）、DEP02（切流后失败自动回滚）、DEP06（中途杀 deployer journal 收敛）
- 全流程：两个不同语言应用、Agent 零 token 走完"改代码→构建→PR→CI 自修→审批→蓝绿上线"
- 临时态：裸 IP + 自签证书 + CORS open 环境下完整演示，并展示一行配置升级到正式域名
