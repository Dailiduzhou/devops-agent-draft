# 项目核心亮点：Agent-in-the-Loop

## 一句话

Agent 真正动手做 DevOps——读代码、写配置、跑构建、改 CI 报错、提 PR——但它从结构上碰不到任何生产秘密和发布决定权。秘密不是"藏起来"，而是**从未进入 Agent 的世界**。

## 亮点 1：Agent 做真活，不是聊天助手

大多数"AI + DevOps"停在生成配置文件让人抄。本项目里 Agent 是流水线上的一个正式工位：

| Agent 动作 | 通道 | 为什么是"真" DevOps |
| --- | --- | --- |
| 读仓库代码、理解项目 | Broker `repo.read`（固定 repo/ref/路径） | 基于真实代码做容器化改造，不是对着片段空谈 |
| 本地改 Dockerfile、compose、健康端点 | 沙箱工作目录 | 沙箱内真实执行编译、测试、构建镜像 |
| 沙箱构建验证 | Broker `sandbox.build`（独立临时构建执行器） | 改完立刻真跑构建，失败自己修 |
| 提交变更 | Broker `proposal.submit` → 自动生成 `agent/` 分支 + PR | 变更走完整的 PR 审查流程 |
| 读 CI 结果、自我修正 | Broker `ci.read`（筛选后的日志） | workflow 报错由 Agent 消化重试，学生不用看懂报错 |

闭环：**改代码 → 沙箱构建 → 提 PR → CI 跑 → 读报错 → 再改 → 再提**。Agent 在环中迭代，人在门禁处批准。

## 亮点 2：敏感信息靠结构隔离，不靠自觉

核心设计原则：**不是"禁止 Agent 读秘密"，而是"秘密在 Agent 的宇宙里不存在"**。四层结构性隔离：

1. **无凭据沙箱**：独立临时用户与工作目录，无宿主 HOME/.ssh、无 Docker socket、无 SSH_AUTH_SOCK、不继承环境变量、无生产网络路由（含 loopback/云元数据/DNS 重绑定防护）。秘密对 Agent 而言物理上不可寻址。
2. **Agent 不持有任何 GitHub token**：所有 GitHub 操作必须经 Proposal Broker。Agent 想越权，没有可越权的凭证。
3. **Broker 白名单动作**：只有 4 个动作（repo.read / proposal.submit / ci.read / sandbox.build），固定参数、严格校验。没有通用 HTTP 代理、没有 Shell、没有 SSH——**没有能力，就没有泄露**。路径穿越、符号链接、控制文件（.github、CODEOWNERS、策略文件）在提交前被拒绝。
4. **生产侧零暴露**：生产凭据只存在于宿主受限目录和 Environment secrets；CI 按信任等级拆分 job，PR 代码永远不会被放进持有发布 token 或生产秘密的环境里执行。

关键洞察（项目独创表述）：**区分"秘密内容不可见"和"生产能力不可用"**。拿不到私钥但能用 Broker 执行任意 SSH，照样是越权——所以本项目约束的是能力本身，而不仅仅是字节。

## 亮点 3：审批权在结构上不属于 Agent

- Agent 的产出只能以 PR 进入，写 `agent/` 提案分支；main 合并、运维仓库、生产 Environment 审批三个环节 Agent 均无身份可达。
- 发布授权是 ops 仓库受保护 workflow 生成的**获批 manifest**（digest 绑定 source_sha、generation 单调、短期有效），主机 deployer 只消费这一条通道——Agent 既造不出也不能伪造这张"通行证"。
- 结果：Agent 越努力干活，安全边界越清晰。**信任来自结构，不来自提示词。**

## 亮点 4：每个安全声明都有测试证据

权限越界不是"应该被拦"，而是被测出来的：SEC01–SEC10 十类攻击用例（push main、伪造 manifest、重放 generation、路径穿越、宿主文件读取……）100% 拒绝，以 API/网络/文件的实际结果为证据，附哈希链审计日志。

## 可对外的一句话总结

> 这个项目让 AI Agent 真正参与容器化、CI/CD 和生产部署的全流程，同时用无凭据沙箱、白名单 Broker 和结构性审批门禁保证：Agent 能修好你的 Dockerfile，但永远读不到你的数据库密码、也永远代替你按下发布按钮。
