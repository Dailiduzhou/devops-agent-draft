# 星源 AgentKit 接入方案：拓扑 B

> 当前决策：AgentEngine 承载受信编排与固定工具适配，KCI 仅承载独立不可信执行工作区；BuildExecutor 单独选型。平台能力有文档依据，但本项目尚未完成真实账号/模板验证。完整边界见 [agent-runtime.md](agent-runtime.md)，执行底座见 [sandbox-tracks.md](sandbox-tracks.md)。

## 一、组件映射与信任边界

| 本项目组件 | AgentKit 接入方式 | 安全约束 |
| --- | --- | --- |
| Agent 主循环、模型客户端 | KSADK + LangGraph / AgentEngine | 仅运行固定受信代码；模型输出不成为运行时本地代码 |
| shell、源码工作区 | KCI SandboxProvider（E2B SDK） | 无模型/平台/GitHub/Broker/生产凭据；无生产网络 |
| 镜像构建 | 独立 BuildExecutor | KCI 构建实例须实测；不满足则临时 VM，不开放宿主 socket |
| Broker 四动作 | 自定义 MCP Server（Streamable HTTP） | 受信适配器附加身份，Broker 完整授权；不是通用 HTTP 代理 |
| 流程知识 | 经审查的只读 Skills | 首版一个应用流程；可执行脚本只进入沙箱/构建环境 |
| CI 事件 | notifier → 持久队列 → 运行时 API | 签名、去重、关联当前提交、串行执行和预算 |
| 可观测 | Trace + 自建审计 | 认证材料不进 Trace；哈希链外部锚定用于篡改检测 |

平台参与受信执行基础设施，但不承载生产发布决定权。AgentKit 故障会影响模型任务、CI 自修与诊断；独立部署、已接受事务的本地恢复与人类告警必须不依赖 AgentKit。

## 二、MCP：只暴露固定能力

```text
模型输出工具参数
  → 受信工具适配器（固定工具注册表、会话映射、配额）
  → MCP Server devops-broker
       ├─ repo.read
       ├─ proposal.submit
       ├─ ci.read
       └─ sandbox.build → 独立 BuildExecutor
  → Proposal Broker（服务身份 + session + 动作授权 + 审计）
```

模型不看到认证头、session_token、GitHub App 私钥或平台管理凭证。会话身份仅由受信代码附加，不能使用模型传入的 session_id 选择其他任务身份。沙箱不直接调用 MCP/Broker。

MVP 优先直连自建 MCP/Broker 端点，以 mTLS 认证受信适配器并附加短期会话材料。若经平台 MCP 网关路由，必须 PoC 验证服务身份、会话隔离、认证材料保护和撤销链路；网关终止 TLS 后的受信转发也纳入可信计算基。不能把网关 API-Key 当作 app 仓库授权，也不能把 MCP 包装层描述为无需鉴权的透明外壳。

Broker 的 repo、基线、PR、workspace/snapshot 归属与预算均为服务端授权状态；MCP schema 只做第一层输入约束，不能替代 Broker 策略。

## 三、Skills：知识与执行分开

首版选一个支持的应用（如 Node HTTP），提供容器化流程与一种 CI 错误修复流程；其余语言和安全审查 Skill 后置。

- 只读流程 Skill 可以指导模型调用工具，但不能增加工具或改变权限。
- 可执行 Skill 进入独立无秘密沙箱，通过固定执行入口运行；不得使用持有凭证的 AgentEngine `local_process` 执行不可信脚本。
- 仓库提供的 Skill、脚本和配置按不可信代码处理，不能自动注册到运行时。
- 通用 Sandbox 成功不代表专用 Skill Runtime 可用。云端 Skill Space、专用模板、账号权限和挂载必须独立验证；如比赛要求云端 Skills，先交付一个有真实调用证据的 Skill。
- KCI 实例提交为镜像之前排除 workspace、令牌、日志与任务残留；预热模板不能保存任务身份。

## 四、事件驱动的有界自修

```text
GitHub CI 完成事件
  → notifier：验签、delivery_id 去重、持久入队、快速应答
  → 核对 session / PR / head_sha / workflow / run
  → 同一 session 串行消费；忽略旧提交失败、重复或已取消任务
  → AgentEngine /v1/responses（session_id；挂起任务按需 ksadk_resume）
  → ci.read → 沙箱修复 → 真实构建 → 更新受管 PR
```

官方运行时文档描述了同 `session_id` 恢复历史和通用 interrupt 的 `ksadk_resume`；MCP approval 的恢复事件与 CI 通知不是同一语义，不能拿 CI 成功伪造人工批准。

配置持久会话存储、LangGraph checkpoint 和工具调用幂等；实测 Pod 重启、重复事件与中断恢复。不把“能恢复聊天记录”当作“能安全恢复有副作用任务”。

首版只自动处理一种可重复注入的应用测试/构建错误。任务有最大轮数、时间、费用和构建次数；workflow 权限、发布审批、平台/网络故障输出人工待办。独立受保护测试阻止删除测试来取得假绿。

## 五、告警与平台故障

Deployer → 独立 notifier → 人类渠道（附 runbook）。人类通知不等待 AgentKit、模型或 Agent 队列。

值守 Agent 诊断 PR 后置；接入时只接收经授权、脱敏的运维摘要，不新增生产工具或运维仓库通道。平台故障时 Agent 分支降级/排队，独立告警出口仍可工作。

GitHub/registry 不可用可能阻止新部署，这是另一种外部依赖；本地回滚应基于已持久接受的授权和保留镜像，不要求平台或 GitHub 在线。

## 六、凭证与配置

| 凭证 | 使用处 | 约束 |
| --- | --- | --- |
| IAM AK/SK | 部署控制面/专用管理进程 | 按需使用，不默认灌入运行时和 guest |
| AgentKit/Sandbox API-Key | 受信 Provider/调度适配器 | 只用于获准平台操作，不进入 shell |
| 模型 API Key | 受信模型客户端 | 不进入 prompt、工具结果、Trace 或沙箱 |
| MCP 调用身份、Broker session_token | 受信工具适配器 | 服务身份 + 短期会话共同校验，可撤销 |
| GitHub App 私钥 | Broker | 不交给 AgentKit 或构建环境 |
| 生产 secret | 对应生产受信主体/业务应用 | 不进入上述开发执行链 |

控制台密文变量是存储保护，不是进程隔离；运行时使用时仍可能有明文。扫描分别覆盖不可信执行区与上下文/Trace，不能要求合法持凭证进程“内存零凭据”。

## 七、早期门禁与后置项

D1–3 验证模型 → MCP → Broker 薄链路、会话认证与撤销、KCI 工具沙箱隔离、BuildExecutor 真实构建、GitHub 审批可用性；D4–7 贯通一个应用的最小安全链路。生产副作用仅允许在已验证的授权与隔离边界内发生。

第二后端、五枚 Skill、值守 Agent、任意框架支持和性能预热优化均不进入首版必达。平台接入能减少部分基础设施工作，但不能预设工作量减半或“只改一个环境变量即可完全脱离平台”。

## 官方依据（实施时固定 SDK/模板版本并复核）

- [快速入门](https://docs.ksyun.com/documents/45702)
- [运行时接口说明](https://docs.ksyun.com/documents/45551)
- [进阶入门：智能体开发](https://docs.ksyun.com/documents/45712)
- [沙箱概述](https://docs.ksyun.com/documents/45552)
- [创建沙箱模板](https://docs.ksyun.com/documents/45554?type=3)
