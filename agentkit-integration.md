# 星源 AgentKit 接入方案（Track-A）

基于金山云星源 AgentKit 官方文档（docs.ksyun.com，产品概述/快速入门/进阶入门/MCP/自定义 MCP/沙箱概述）的调研结论。本文是 Track-A 的总体方案；沙箱后端的 plugin 化与双轨对照见 `sandbox-tracks.md`。

## 一、平台能力速览（调研结论）

AgentKit = 开发套件（KSADK + AgentEngine CLI，兼容 LangGraph/LangChain/ADK/DeepAgents）+ Agent 开发运行平台（构建/部署/在线调试/Serverless 运行时）+ 能力组件（知识库、记忆库、MCP、沙箱 KCI、Skills 中心、AI 搜索）+ 可观测评测（Session/Trace/指标/评测）。

与我们架构的对应关系：

| 我们的组件 | AgentKit 对应物 |
| --- | --- |
| Orchestrator（调度脚本） | AgentEngine 运行时 + KSADK |
| 沙箱容器 + 构建执行器 | KCI 沙箱（E2B SDK，自定义镜像模板、预热池、VPC 资源访问、实例提交为镜像） |
| Proposal Broker 四动作 | 自定义 MCP Server（代码方式部署，Streamable HTTP；或 HTTP 转 MCP 包装） |
| DevOps 任务知识（模板、修 CI 套路） | Skills 中心 |
| 调用链观测 | AgentKit Trace（与我们哈希链审计并存，见第五节） |

## 二、MCP 还是 Skills？——两者都用，分层接入

### Broker 四动作 → MCP（能力层）

MCP 的"Agent 调用外部受控服务"语义与 Broker 白名单动作代理一一对应：

```
MCP Server "devops-broker"（部署在受信网络）
  ├── tool: repo.read(path, ref)        → Broker repo.read
  ├── tool: proposal.submit(patch, ...) → Broker proposal.submit
  ├── tool: ci.read(run_id)             → Broker ci.read
  └── tool: sandbox.build(spec)         → Broker → KCI 沙箱执行
```

要点：

- Broker 保持自研不变，只加 MCP 协议适配层（或用平台"HTTP 转 MCP"直接包装 Broker REST API）。
- **鉴权不变**：MCP Server 校验我们的 session_token；AgentKit 平台 API-Key 只用于平台路由，不接触 GitHub App 私钥。适配层无状态、无秘密。
- **安全边界不变**：Agent 经 MCP 拿到的仍只有过滤后结果；路径校验、限流、审计全在 Broker 内执行。MCP 只是协议外壳，不是新信任边界。

### DevOps 流程 → Skills（流程层）

| Skill | 内容 |
| --- | --- |
| containerize-node | 探测 express/nest → 选 Dockerfile 模板 → 加 livez/readyz → 调 sandbox.build 验证 |
| containerize-spring | actuator 映射健康端点 → 多阶段分层 Dockerfile → 构建验证 |
| containerize-go | Go 模板 + 静态编译检查 |
| fix-ci-failure | ci.read 结果 → 按错误类别（lint/测试/构建）匹配修复策略 → 重提 patch |
| security-review-app | 应用层检查：SQL 拼接、CORS 硬编码、明文密钥（呼应安全教育主线） |

注意（官方文档明示的坑）：通用 Sandbox ≠ Skill 专用 Runtime；需用 KCI"实例提交为镜像"自建包含 Skill 运行时的专用模板。

**一句话分工：MCP 给 Agent 手（能力），Skills 给 Agent 脑（流程）。** Agent 工具面仍是四动作白名单，Skills 只决定调用顺序与内容——与"工具面 = 能力面"的安全模型完全兼容。

## 三、Webhook 通知链路（双向）

### 方向 1：CI/CD → Agent（事件驱动替代轮询）

```
GitHub Actions（check_run completed / workflow_job 失败）
  → GitHub Webhook → notifier（自建、受信侧，几十行 Go）
     ├→ 签名校验 + 过滤（只推本会话关联的 PR/run）
     ├→ 调 AgentEngine 运行时 API 唤醒挂起的 Agent 会话，
     │   注入 {"pr": 42, "run": 123, "conclusion": "failure"}
     └→ 回退：写任务队列，Agent 轮询 ci.read
```

CI 挂了 Agent 立即被唤醒修复，修完提交再等下一个 webhook——天然的事件循环演示。边界：webhook secret 只在 notifier；Agent 收到的是它本可用 ci.read 拿到的信息，webhook 未扩大 Agent 任何权限。

### 方向 2：Deployer 警告 → 人 + Agent 值守

```
Deployer journal（发布失败/回滚/ManualRequired/探活异常）
  → notifier → ① 人类渠道（飞书/钉钉 webhook + runbook 链接）
             ② 唤醒值守 Agent：读 journal 摘要 + 公开日志 →
               生成诊断初稿 → 提 docs/proposals 诊断报告 PR
```

边界：值守 Agent 只读摘要与公开日志，诊断走 PR；重试/回滚决策仍是 deployer 单调 generation + 人工入口的事。

## 四、凭证管理（按官方凭证分层）

| 凭证 | 存放 |
| --- | --- |
| IAM AK/SK（控制面、KS3） | 本地 .env 或控制台密文变量 |
| AgentKit API-Key（MCP/沙箱/AI 搜索） | 控制台密文变量 |
| 模型 API Key | 控制台密文变量 |
| GitHub App 私钥 | 仅 Broker 侧（与平台无关） |
| Agent session_token | 仅沙箱内，短时绑定实例 |

原则：平台凭证绝不进入 Agent 会话上下文或沙箱环境。

## 五、收益、保留项与风险

**收益**：沙箱/执行器/运行时托管三块自建基础设施换成平台能力，23 天 D 线工作量减半；Trace 满足可观测演示；Serverless 省自管 Agent 进程。

**保留不变（安全叙事核心）**：Proposal Broker 及全部策略、哈希链审计、获批 manifest 发布协议、GitHub App 身份、人工审批。AgentKit 提供执行底座，不进入信任决策链。Trace 与哈希链并存：Trace 看调用链（平台侧），哈希链是不可篡改审计证据（我们自己的主张，不外包）。

**风险与实测项**：
- KCI 网络隔离粒度（云元数据 DENY、DNS 重绑定）以 Provider 契约测试实测为准（sandbox-tracks.md）。
- Skill Runtime 镜像需自建。
- 平台可用性纳入故障分类：AgentKit 不可用只影响沙箱任务排队，生产链路完全不经过平台。
