# Agent 运行时：启动、能力、限制与权限分工

## 一、Agent 的启动与生命周期

Agent = 跑在无凭据沙箱容器里的 LLM 会话，任务驱动、短生命周期。

1. 人类操作员向 orchestrator（受信调度脚本）发起任务。
2. orchestrator 创建一次性沙箱容器：专用 UID、独立 /workspace；不挂载宿主 HOME/.ssh/Docker socket/SSH_AUTH_SOCK；环境变量白名单；网络默认 deny，仅放行 Broker 与构建执行器及受控包源。
3. orchestrator 向 Broker 注册会话，Broker 校验沙箱实例登记，签发短时 session_token（绑定容器 ID，TTL=任务时长）。
4. Agent 进程持有且仅持有 session_token + 任务描述 + 工具清单。
5. 任务完成或 TTL 到期 → 容器销毁，token 自动作废。每次任务全新身份，零残留。

Agent 的身份就是 session_token；Broker 每请求校验 token 与容器实例匹配，防复用。orchestrator 只决定"开不开沙箱"，不参与代码与发布决策。

## 二、工具面 = 能力面

### shell（沙箱本地）
沙箱内 exec，UID 2000，只写 /workspace。读代码、编译、测试、生成 patch。构建经 sandbox.build 代理。

### broker_call（唯一出沙箱通道）
| 动作 | 用途 | 典型链路 |
| --- | --- | --- |
| repo.read | 了解项目：文件树/源码/依赖，白名单路径 | 任务开始 → 判断技术栈 |
| proposal.submit | 交付：patch → Broker 建 agent/ 分支 + PR | 改完+构建通过 → 提交 |
| ci.read | 迭代：读筛选后 CI 日志 → 修 → 再提 | PR run_id → 分析失败 → 修复循环 |
| sandbox.build | 真实构建验证 | 提 PR 前完整 build + smoke |

不存在的工具：git push、任意 HTTP、SSH、沙箱外写、secret 读取、workflow 创建、merge、审批评论。

## 三、限制四层次（纵深防御）

1. **网络层**：仅 Broker、构建执行器、受控包管理源可达；deny 云元数据/宿主内网/loopback 代理；DNS 由本地 stub resolver 固定。Agent 被完全攻破也发不出 GitHub/生产/任意外网包。
2. **身份层**：零凭据——沙箱内不存在任何 GitHub token、SSH 密钥、云凭据；session_token 对 GitHub 无意义。物理上排除"Agent 泄露凭据"。
3. **Broker 策略层**：统一管线——认证（token+容器匹配）→ 限流 → 严格 Schema → 动作策略 → 以 App token 执行（token 永不回传）→ 结果过滤 → 哈希链审计。
   - repo.read：repo/ref 白名单，路径规范化 + 允许树检查，控制路径拒绝，响应截断。
   - proposal.submit：逐 hunk 路径检查（拒绝 ../、绝对路径、.git/）；符号链接逃逸检查；控制文件（.github/workflows、CODEOWNERS、策略文件、requests/、.gitleaksignore）只能以 docs/proposals 文本建议；base_sha 必须为 main HEAD；Git 大小写/同形字符专项测试；分支名与 PR 由 Broker 生成。
   - sandbox.build：转发独立无秘密构建执行器（rootless BuildKit/临时容器），固定参数、超时、资源限制，只返回结论+日志摘要。允许改 Dockerfile = 允许写任意构建代码，故用无秘密可丢弃环境兜底。
   - ci.read：run_id 必须关联本会话 PR；白名单正则提取失败步骤/编译错误/测试名/结论，不回原始日志。
4. **平台层**：App 安装范围仅 app 仓库、contents:write+pull_requests:write；main 强制 PR+独立批准+状态检查，无 bypass；运维仓库未安装 App——生产授权通道物理不存在。

## 四、权限分工

**Agent 有劳动能力，Broker 有代理能力，人类有决定能力。**

| 能力 | Agent | Broker | 人类维护者 |
| --- | --- | --- | --- |
| 读应用源码 | repo.read（白名单） | App 代执行 | 直接 |
| 代码提案 | patch → proposal.submit | 校验+写分支+开 PR | 审查合并 |
| 读 CI 结果 | ci.read（过滤版） | 持映射+过滤原始日志 | 直接 |
| 触发构建 | 请求 sandbox.build | 调度隔离执行器 | 直接 |
| 合并/发布触发 | 无通道 | 无权限 | ops 发布请求 PR |
| 批准生产 | 无通道 | 无权限 | Environment 审批（唯一入口） |
| 生产凭据 | 不存在 | 不存在 | 宿主 secret / Env secrets |

## 五、完整任务时序示例

Node 项目容器化：repo.read 判断技术栈 → shell 写 Dockerfile+健康端点 → sandbox.build 失败（npm 超时）→ 修复 → 构建成功 → proposal.submit → PR #42 → ci.read 发现 lint 失败 → 修正重提 → 全绿 → 沙箱销毁 → 维护者审查合并 → 发布协议（Agent 不再参与）。全程 broker_call 入哈希链审计；提示注入试图读宿主文件外传时，shell 只见沙箱文件、网络层 DENY 无路可走（SEC05/SEC07 实测）。
