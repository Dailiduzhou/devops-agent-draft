# 项目核心对齐文档（v2）

供项目内部对齐使用。v2 修订：将**金山云星源 AgentKit 平台适配确立为主线**（Agent 运行时、沙箱、MCP、Skills 均基于平台能力构建），同时通过 **Sandbox Provider 抽象层**保留沙箱等执行组件的可替换性（自建后端随时可切换）。安全决策链（Broker、发布协议、审批、审计）不依赖任何平台。

## 一、背景与要解决的问题

学生项目普遍缺少两块能力：

1. **安全意识**：对 SQL 注入、跨域、重放攻击、凭据管理等 Web 安全问题认识不足；引入 AI Agent 后，凭据和生产环境更是直接暴露在 Agent 可触达范围内。
2. **部署与运维能力**：本地能跑、上线即崩；部署靠手工 SSH，没有版本化、审批、健康检查和回滚；对生产环境的安全法则（权限隔离、最小权限、审计）了解不深。

本项目要做的，就是**代替学生跑通从"一个学生 Web 项目"到"完整、安全、可持续运维的线上服务"的全流程**，并把这条流程本身做成可复用、可演示、可教学的产品——以金山云星源 AgentKit 作为 Agent 侧的运行底座。

## 二、核心定位

- 输入：一个已有的学生 Web 项目（语言不限：Go / Node / Spring Boot…，流水线对应用语言零假设）。
- 输出：一套端到端的工程系统——改造后的应用、基于 AgentKit 的 Agent 工程化链路、CI/CD 流水线、权限边界体系、故障恢复机制、以及完整的验证证据。
- 差异化：不只是"跑通 DevOps"，而是让 AI Agent **托管在云平台 AgentKit 上**安全地参与真实工程——Agent 干活，但生产凭据、生产网络、发布决定权始终由受控边界和人来掌握。平台提供执行底座，不进入信任决策链。

## 三、核心目标

1. **可复现交付**：新成员凭 README 在干净机器上一键本地启动与测试。
2. **一次构建，处处部署**：同一镜像 digest 贯穿验证与生产发布，生产主机只消费不可变镜像。
3. **全程可审计**：每次生产变更有来源提交、检查结果、人工审批记录、获批 manifest 和实际部署结果；Agent 侧有 AgentKit Trace + 自建哈希链审计双通道。
4. **故障自愈**：健康验证失败自动恢复旧应用，保留故障现场；数据库变更独立管理。
5. **生产级运维**：TLS、优雅退出、资源约束、日志轮转、监控告警、备份与恢复演练。

## 四、服务对象

- **直接对象**：学生工程团队（23 天黑客松节奏，详见 implementation.md）。
- **最终受益者**：所有想把课程/比赛项目安全上线的开发者——他们获得的是一条现成的安全部署路径和一套可以照做的安全工程范式。
- **角色分工**：项目负责人（架构与门禁）、应用与容器负责人（应用契约与镜像）、平台与发布负责人（CI/ops/deployer）、Agent 与安全评审负责人（AgentKit 接入、Broker 策略、负面用例与凭据审计）。

## 五、主线架构：基于 AgentKit 的 Agent-in-the-Loop

### 5.1 组件映射

| 能力 | 承载组件 | 说明 |
| --- | --- | --- |
| Agent 主循环与运行时 | KSADK（LangGraph）+ AgentEngine Serverless 运行时 | 兼容主流框架，云端托管，在线调试 |
| Agent 沙箱与构建执行 | KCI 沙箱（E2B SDK，自定义模板、预热池） | 隔离执行代码/命令/构建，经 Provider 抽象层接入（见 §6） |
| Broker 四动作对外暴露 | 自定义 MCP Server（Streamable HTTP） | repo.read / proposal.submit / ci.read / sandbox.build 四个 MCP tool，协议外壳，无新信任边界 |
| DevOps 流程知识 | Skills 中心 | containerize-node/spring/go、fix-ci-failure、security-review-app |
| CI/部署事件驱动 | GitHub Webhook + notifier → AgentEngine 运行时 API 唤醒 | CI 失败即刻唤醒 Agent 修复；Deployer 告警触发值守 Agent 出诊断 PR |
| 可观测 | AgentKit Trace + 自建哈希链审计 | Trace 看调用链；哈希链是不可篡改证据，不外包 |

**分层原则：MCP 给 Agent 手（能力，四动作白名单），Skills 给 Agent 脑（流程）。** Agent 工具面仍是四动作白名单，Skills 只决定调用顺序——"工具面 = 能力面"的安全模型不变。

### 5.2 核心链路

```
AgentEngine 运行时（AgentKit 托管）
    │ 加载 Skills（容器化/修CI/安全审查）
    ▼
KCI 沙箱（无凭据，经 Provider 抽象层）
    │ broker_call（MCP）
    ▼
Proposal Broker（自研，持有 GitHub App 身份，四动作白名单 + 策略 + 哈希链审计）
    ▼
应用仓库 PR → 人工审查 → 合并 main → 可信构建 → 镜像推送（digest 绑定 source_sha）
    ▼                          ▲
    │ Webhook（CI 失败）→ notifier 唤醒 Agent 自修（事件驱动闭环）
    ▼
运维仓库（Agent 无权访问）：发布请求 PR → release workflow 校验候选来源
    ▼
production Environment 人工审批 → 生成获批 manifest（GitHub Release 资产）
    ▼
生产主机 deployer（systemd timer，仅出站拉取）：校验 manifest → 蓝绿双槽位
    → 候选验证 → Caddy 切流 → 观察窗口 → 原子提交；失败自动回滚
    → 告警经 notifier → 人类渠道 + 值守 Agent 诊断 PR
```

三条关键设计原则：

1. **提案与发布双边界**：Agent 的产出只以 PR 进入；生产发布需要独立仓库、独立身份、独立人工审批。
2. **Pull 部署**：服务器只出站拉取获批 manifest 和不可变镜像，授权来自受保护 workflow 生成的批准记录——**生产链路完全不经过 AgentKit**。
3. **部署状态机**：预检 → 拉取 → 候选启动 → 验证 → 切流 → 观察 → 原子提交，每步有 journal、幂等和补偿路径。

## 六、可扩展性：Sandbox Provider 抽象层

沙箱不是写死绑定 KCI。所有沙箱能力收敛为一个抽象接口（Provider / Instance / NetworkPolicy），后端以 plugin 形式接入，可替换：

| 后端 | 实现 | 适用 |
| --- | --- | --- |
| `agentkit`（主线） | KCI 沙箱，E2B SDK，模板预热加速 | 黑客松主线：省自建基础设施，弹性并发 |
| `local`（备选） | 本地一次性容器 + 专用 UID + nftables 白名单 + rootless BuildKit | 零云平台依赖部署；隔离粒度完全自控的基准实现 |

**契约测试套件与后端无关**：SEC07（宿主文件/元数据/Docker socket 访问拒绝）、网络 DENY、资源上限、销毁零残留等用例直接针对 Provider 接口编写，两个后端跑同一套。KCI 隔离粒度是否达标由契约测试实测裁决，而非假设。切换后端 = 换一个环境变量，上层（Broker、发布协议、deployer）零改动。

## 七、学生能力差距与我们的答案

学生的起点是"可能没有 Docker、SSH、Linux 基础，看不懂 workflow 报错，不懂 Repo Environment、known_hosts、镜像仓库配置"。答案不是补课，而是**把每一个能力缺口变成项目的一个交付物**：

| 学生缺的能力 | 我们的交付 | 学生实际要做的 |
| --- | --- | --- |
| 装 Docker / Compose / Caddy | ops 仓库 bootstrap 幂等初始化脚本（含环境检测，报具体失败原因） | 干净 Ubuntu LTS 主机执行一条命令 |
| 配置 GitHub Environment / rulesets / CODEOWNERS | 幂等 setup 脚本（gh CLI / API） | 复制粘贴一条命令并确认输出 |
| 服务器 SSH、known_hosts、密钥 | 全程 Pull 部署，发布流程零 SSH 依赖 | 无 |
| 镜像仓库配置 | GHCR 推送用 GITHUB_TOKEN；主机 bootstrap 实测连通性，不足时镜像复制兜底 | 无 |
| 看不懂 CI/workflow 报错 | CI Webhook 唤醒 AgentKit 上的 Agent，经 ci.read 读筛选日志自行修正重提 | 看 Agent 汇报的结论 |
| 安全配置判断 | 分支保护、审批门禁、扫描全部由 setup 脚本和模板预置 | 审批界面核对 diff 并点批准 |

**分工原则**：Agent（AgentKit 托管）执行全部代码改造、构建验证和报错诊断；学生保留人工审批——这既是教学价值所在，也是架构中不可绕过的人工门禁。学生能力要求从"会操作"降为"能看懂提示并做出批准决定"。

## 八、能力地图（我们交付什么）

| 板块 | 内容 |
| --- | --- |
| Agent 平台接入 | AgentEngine 运行时部署、KCI 沙箱模板（含 Skill Runtime 镜像）、devops-broker MCP Server、五枚 Skills、Webhook 事件驱动闭环 |
| 沙箱抽象层 | Sandbox Provider interface + agentkit/local 双后端 + 后端无关契约测试 |
| 应用改造 | 语言无关契约（livez/readyz/version、优雅退出、超时）、Go/Node/Spring Boot 三份模板、业务 smoke、多阶段非 root 镜像 |
| 本地闭环 | compose 开发栈、make dev-up/smoke/dev-down、示例配置 |
| CI | 按信任等级拆分 job：PR checks → main 可信构建 → 隔离镜像推送 → candidate 元数据（SBOM、digest、来源绑定） |
| 发布协议 | ops 仓库发布请求、Environment 审批、严格 JSON Schema 的 manifest、generation 单调序号、有效期 |
| 一键初始化 | 主机 bootstrap（Docker/Compose/Caddy/证书/目录/registry 连通性）、GitHub setup 脚本、部署档位配置（tls_mode/cors_mode/access，宿主受信配置，Agent 改不了） |
| 生产部署 | Go deployer、flock、崩溃恢复 journal、蓝绿双槽位、Caddy 模板渲染切流 |
| Agent 安全 | 无凭据沙箱、Broker 四动作白名单 + 策略管线、哈希链审计 |
| 秘密管理 | 宿主受限目录只读 secret、AgentKit 凭证分层（平台凭证绝不进 Agent 会话上下文）、轮换与吊销 |
| 数据安全 | expand-contract 迁移、迁移锁、每日备份 + 异地副本 + 空白实例恢复演练 |
| 运维值守 | 外部探活、结构化日志、告警 runbook、故障分类；notifier 双出口（人 + 值守 Agent） |
| 验证体系 | 20+ 自动化安全与故障用例，含 Provider 契约测试与 AgentKit 平台故障分类 |

## 九、排期与验收（要点）

排期按 23 天黑客松节奏（四线并行，详见 implementation.md），主线变化：Agent/沙箱线基于 AgentKit 组件搭建（D3–5 local Provider + 契约测试冻结，D9–15 agentkit Provider + MCP + Skills，D16–18 Webhook 事件闭环与 Agent 全链路接入）。

验收在原口径上增加：

- **双后端契约测试**：agentkit 与 local 跑同一套隔离/能力用例，全部通过。
- **事件驱动闭环**：CI 注入失败 → Webhook 唤醒 Agent → 自动修复重提 → CI 转绿，全程无人工干预，Trace 可回放。
- **平台解耦证明**：模拟 AgentKit 不可用，生产部署链路（审批 → manifest → deployer → 回滚）不受影响。
- 原有口径保留：发布成功率 ≥19/20、故障演练全恢复、回退 ≤120s、权限越界用例 100% 拒绝、幂等、RPO≤24h/RTO≤60min、凭据零暴露。

## 十、一句话

我们交付一条**可审计、可回滚、Agent 权限被结构性约束**的端到端部署流水线：学生项目进来，安全稳定的线上服务出去——Agent 跑在金山云星源 AgentKit 上干活，执行底座可插拔，信任决策链永远在我们自己的边界里，并且每一步都有证据。
