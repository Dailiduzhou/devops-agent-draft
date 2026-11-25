# Agent 运行时：拓扑 B、身份与能力边界

> 当前决策：采用拓扑 B。本文定义运行位置、凭证归属与工具执行边界；实现范围和排期见 [implementation.md](implementation.md)。以下为设计与验收要求，不代表已经实现或测试通过。

## 一、拓扑 B：主循环与不可信执行环境分离

```text
维护者 → 受信调度器 / AgentEngine 运行时（KSADK + LangGraph）
                        ├─ 模型客户端：使用模型凭证
                        ├─ 固定工具适配器：校验参数、附加服务身份
                        │    ├─ broker_call → MCP → Proposal Broker
                        │    └─ shell → SandboxProvider → 独立执行沙箱
                        └─ 会话状态 / 预算 / 工具调用记录

Proposal Broker ── sandbox.build ──> BuildExecutor 控制端
                                      └─ 独立临时构建环境
Proposal Broker ── GitHub App ──> 应用仓库提案 PR

人工审查与独立发布审批 → 获批 manifest → 生产 Deployer
```

区分四个主体：模型产生不可信文本和工具参数；受信运行时维护状态、调用模型；固定工具适配器执行受控分派；沙箱执行 shell、仓库代码和可执行 Skill。**持有平台凭证的运行时不得执行模型生成或仓库提供的任意代码。**

受信运行时仅运行经审查、固定版本的编排与适配代码；禁用本地任意 shell/Python、`eval/exec`、仓库插件动态加载和不可信 `local_process` Skill。模型、源码、日志和 Skill 输出一律按不可信输入处理，不能改写工具注册表、身份、目标仓库或安全策略。

## 二、安全目标与不保证的事项

目标：未经批准的 Agent 会话不能直接读取生产秘密、访问生产网络或授权发布。模型只能请求有限工具能力；权限由适配器、Broker、OS、挂载、网络及 GitHub 门禁共同约束，不以提示词或工具名称作为边界。

限制：Agent 生成的代码若获批上线，可能使用应用合法持有的数据库/API 权限。人工审批是授权而非代码安全证明；只读 secret 挂载也不阻止应用读取和泄露。通过独立审查、受保护验收测试、最小业务账号、生产出站控制和凭据轮换降低风险，不宣称物理消除所有凭据泄露。

可信计算基包括受信运行时及适配器、调度器、Broker、BuildExecutor 控制端、平台与隔离基础设施、GitHub/管理员/审批人、发布 workflow 和 Deployer。上述主体被攻陷不属于“提示注入无法越权”的无条件保证。

## 三、凭证归属与会话认证

| 凭证/身份 | 使用主体 | 不可信执行区是否可读 |
| --- | --- | --- |
| 模型 API Key | 受信运行时模型客户端 | 否 |
| AgentKit 沙箱管理/API 凭证 | 受信调度器或 Provider 适配器 | 否 |
| IAM AK/SK（部署控制面） | 部署工具或专用控制进程；不无差别注入运行时 | 否 |
| Broker 调用方身份与短期 session_token | 受信工具适配器 | 否 |
| GitHub App 私钥/installation token | Broker | 否 |
| 构建基础设施管理身份 | BuildExecutor 控制端 | 否 |
| 生产业务 secret | 对应生产应用，按需授予 | 不进入 Agent 沙箱或构建环境 |
| 发布、镜像拉取、迁移、备份身份 | 对应生产受信主体，彼此分离 | 否 |

密文变量解决存储问题，使用凭证的受信进程仍可能接触明文。准确承诺是“模型上下文不包含认证材料，不可信沙箱/构建环境不持有模型、平台、GitHub、Broker 或生产凭据”，而不是“运行时进程零凭据”。Trace、日志和工具结果不记录认证头、完整环境或密钥。

MVP 由受信适配器发起所有 Broker 请求，沙箱不直连 Broker、模型或平台管理 API。适配器经 TLS 和独立服务身份（MVP 采用 mTLS）认证；Broker 的会话记录关联调用方、维护者授权的 app 仓库、基线、PR、workspace/沙箱、预算及有效期。session_token 留在适配器，必须与服务身份一起校验；模型参数不能选择或替换调用方身份。

沙箱 ID 只作为资源归属关联，不能单凭请求自报 ID 或 token 中的 instance_id 证明请求来源。mTLS 校验服务身份，不证明任意 guest 的完整性；受信适配器被攻陷仍可能滥用自身权限。共享 MCP 网关必须能安全传递或终止并重建受信认证链，且隔离会话；早期 PoC 验证，不允许静默降级成匿名 Broker。

## 四、启动、事件与销毁

1. 维护者向受信调度器提交任务；调度器固定 app 仓库、基线和预算。
2. 调度器通过 SandboxProvider 创建一次性 workspace；无宿主 HOME/.ssh、Docker socket、SSH_AUTH_SOCK、平台环境变量和生产网络。
3. 调度器经管理入口在 Broker 登记会话与资源；该入口不暴露给模型。短期会话认证材料只交给工具适配器。
4. 运行时加载经审查的只读流程 Skill。`repo.read` 返回受限源码后，经 Provider 文件接口写入 workspace；shell 仅在该沙箱执行。
5. 构建前由受信协调端冻结/导出一致的 workspace 快照，校验路径、文件类型和大小，登记内容摘要与归属。适配器附加服务端认可的 snapshot_id；模型不能提交宿主路径、任意 URL 或其他会话的快照。
6. Broker 调度独立 BuildExecutor。沙箱构建仅供反馈，生产镜像由受保护 main CI 重新构建。
7. CI Webhook 经 notifier 验签、持久化、去重，关联当前 PR head/run；按会话串行触发同一 session 的继续执行。恢复依赖持久会话与 graph checkpoint，不仅是历史文本。
8. 完成、取消、超时或预算耗尽：撤销会话授权，取消并回收构建任务，销毁沙箱/workspace，记录结果。进程重启后回收孤儿资源。人工要求修改时开启新授权会话并重新核对已有 PR 归属，不复活旧 token。

任务预算必须包含修复轮数、墙钟时间、模型费用、构建次数、资源与日志大小；到限后输出诊断和人工待办。CI 等待期间可续期/重建沙箱，但不能由模型自助延长授权。

## 五、工具面与 Broker 策略

### shell

固定适配器通过 Provider 在独立沙箱执行命令，只能写获准 workspace/临时目录。测试脚本、编译器插件、依赖安装脚本均视为任意代码；安全依赖隔离而非命令名称。工具适配器本身不在 AgentEngine 本地执行这些命令。

### broker_call：四动作不变

| 动作 | 输入及策略 | 输出 |
| --- | --- | --- |
| repo.read | 服务端固定 repo/ref；相对路径允许树、规范化、大小限制 | 受限源码/文件树 |
| proposal.submit | base_sha、预期受管 PR head、文本 patch；校验 Git tree/模式/父路径/最终 diff | Broker 生成分支、提交与 PR；重试更新已有受管 PR |
| ci.read | run 必须关联当前会话提案、允许 workflow 和提交 | 筛选、截断、脱敏的结构化失败信息 |
| sandbox.build | 固定 preset、服务端登记快照、预算；作业幂等与归属校验 | 构建/smoke 结论与限长摘要 |

统一管线：调用方与会话认证 → 配额 → 严格 Schema → 动作授权 → 外部操作 → 结果过滤 → 审计。关键写操作先持久记录意图；失败/超时需核对真实结果再重试。

- 拒绝任意 Git URL、header、Broker shell、宿主路径和通用 HTTP 代理。
- 首版 patch 仅允许普通 UTF-8 文件新增/修改/删除；拒绝 symlink、submodule、二进制、文件模式变更及未支持的 rename/copy。精确基线应用后复验最终树；不在 Broker 工作目录执行仓库 hook/filter。
- `.github`、CODEOWNERS、发布与安全策略等控制文件仅允许文本建议，不自动应用。不能为修复 CI 放开控制路径。
- 基线固定于会话；更新 PR 使用预期 head 防止覆盖并发提交。main 漂移时按策略要求重新核对/重建，不把每次等于瞬时 main HEAD 当作唯一策略。
- GitHub App 安装仅 app 仓库，按需授予 Contents/PR 写、Actions 读；无规则 bypass。token 不天然限 `agent/` 分支，也不天然无 merge 能力，需结合 Broker 与真实分支更新规则验证。Broker 无 merge 工具，Agent 无 App 身份。
- 禁止通过删除测试或放宽断言让 CI 假绿；受保护的独立验收测试不由 Agent 修改。

## 六、验收边界

- 在 shell、仓库安装脚本、Dockerfile RUN 和可执行 Skill 中分别尝试读取 canary 凭证/受禁文件和访问受禁网络；不能只测试模型是否拒绝。
- 检查模型上下文、工具参数/结果、沙箱环境、工作区、Trace、日志和镜像层；受信进程的合法凭证使用与不可信区泄露分别验收。
- 测试跨会话资源访问、伪造实例 ID、错误服务身份、过期/撤销 token 和重复调用。
- 隔离测试须有允许目标的正向控制组；拒绝和超时原因可解释。有限负例通过不证明不存在所有逃逸或外传路径。
- 平台故障影响 Agent 任务、自修与诊断；独立发布、已接受事务的本地恢复和人类告警不得等待 AgentKit。
