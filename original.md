项目书：学生项目端到端容器化部署流水线（Agent 最小权限在环）

版本：v1.0（课题立项稿）
周期建议：8–10 周
项目定位：不是单纯“跑通 DevOps”，而是构建一条 Agent 可参与工程化、但在结构上无法触达生产秘密 的容器化部署流水线。

---

1. 项目背景与问题

学生项目通常存在三类问题：

1. 交付形态不可复现：本地能跑，服务器上因环境差异失败；
2. 部署靠手工 SSH：无版本化、无回滚、无健康检查，服务不稳定；
3. 引入 AI Agent 后权限失控：Agent 能改 workflow、能读 .env、能直接 SSH，生产凭据暴露在 Agent 可触达范围内。

本项目要解决的核心矛盾是：Agent 要帮学生完成容器化与部署配置，就必须能读写仓库和执行部分命令；但传统 .env + Repo Secrets 模式会把凭据暴露给 Agent。 因此课题的实质是设计一条“两区一界”的流水线：Agent 只在受限区活动，生产秘密被隔离在 Broker 之后。

---

2. 项目目标

2.1 总目标

将一个学生项目完成 容器化 → 反向代理 → GitHub Actions CI/CD → 云服务器稳定运行 的全流程闭环，并确保：

· Agent 只能提交变更建议，不能直接写 main、不能触发生产部署、不能读取真实凭据；
· 部署具备 预检、幂等、健康检查、自动回滚；
· 中间障碍项在每阶段开始前被显式列出并有对应门禁。

2.2 可量化目标

· 部署成功率 ≥ 95%（连续 20 次部署统计）；
· 健康检查失败后自动回滚成功率 100%；
· Agent 权限越界测试全部被拦截；
· 部署脚本重复执行结果一致（幂等验证通过）；
· 生产凭据在 Agent 进程/上下文/日志中零出现（扫描验证）。

2.3 非目标

· 不训练或微调 Agent 模型；
· 不替代云平台自身安全能力；
· 不覆盖所有 PaaS/异构服务器，仅选 1–2 类典型环境验证。

---

3. 总体架构：两区一界

区域 内容 Agent 权限
A. Agent 可触达区 学生仓库、Dockerfile、compose、Nginx/Caddy 配置、workflow 建议版本、CI 构建环境 可读代码、可开 PR、可看 CI 结果；不可写 main、不可触发部署 job、不可读 secrets
B. 生产秘密区 云服务器、SSH 私钥、DB 密码、API Key、TLS 私钥、Environment secrets/Vault 完全不可触达
C. 边界：Credential Broker / 注入代理 占位符 token、出站替换、SSH agent forwarding、审计日志 Agent 只持有占位符；真实凭据在出站瞬间注入

关键原则：Agent 的产出只以 PR 进入；生产部署只由受保护分支触发，并挂在带人工审批的 Environment 下。

---

4. 实施阶段与里程碑

阶段一：本地闭环（第 1–2 周）

任务：学生项目容器化 → docker compose 跑通 → Nginx/Caddy 反向代理 → 独立 /health 端点。
产出：本地可复现的 compose 栈、代理配置、健康检查端点。
门禁：本地 smoke test 通过；.dockerignore 排除 .env、\*.pem。

阶段二：CI 但无 CD（第 3–4 周）

任务：GitHub Actions 只做 build + test + secrets 扫描（gitleaks/trufflehog）+ 镜像推送（OIDC/短期 token）。Agent 只能开 PR 修改 Dockerfile/workflow，部署 job 未启用。
门禁：PR 必须通过 CI；main 分支受保护；CODEOWNERS 覆盖 workflow 文件。

阶段三：CD 加最小权限（第 5–7 周）

任务：启用部署 job，挂 Environment + required reviewers + concurrency 锁；优先 Pull 模式（服务器侧 Watchtower/自写 poller 拉镜像），或 SSH 凭据放 Environment secrets；部署后 smoke test，失败自动回滚到上一版本 tag。
门禁：生产部署需人工审批；镜像 tag 版本化（git SHA/语义版本），禁用 latest；部署脚本幂等。

阶段四：Agent 在环安全验证（第 8–10 周）

任务：攻击自己的流水线：构造含 echo $SECRET 的 PR，验证扫描门禁拦截；构造健康检查失败，验证回滚；验证 Agent token 无法写 main、无法触发部署。
产出：安全验证报告、审计日志样例、最终演示。

---

5. 障碍项与对策清单（提前暴露）

障碍 风险 对策
Agent 改 workflow 读取 secrets 致命 workflow 修改需 CODEOWNERS/人工审批；部署 secrets 只放 Environment secrets；部署 job 仅受保护分支触发
SSH 私钥放 Repo secrets 高 优先 Pull 模式（Actions 不持有服务器 SSH 凭据）；或 OIDC 短期凭证；SSH key 放 Environment secrets
Dockerfile 硬编码凭据/COPY .env 高 CI secrets 扫描门禁；多阶段构建；.dockerignore 排除敏感文件
代理配置错误/健康检查误判 中 nginx -t 预检；独立 /health 端点检查 DB 连通性；失败自动回滚
服务器环境漂移（磁盘满/端口占用/旧容器） 中 部署前 preflight 检查磁盘/端口/版本；docker compose up -d --remove-orphans 幂等收敛
Agent 自动变更不可审计 中 Agent 只开 PR；token 白名单；Broker 操作级约束；审计日志记录 action/参数/结果
并发部署踩踏 中 concurrency 组串行化，CD 设 cancel-in-progress: false
DB 迁移不可逆 高 expand-contract 向前兼容；迁移前备份；失败回滚应用版本而非回滚 DB

---

6. Agent 最小权限矩阵

权限 授予 理由
contents: read ✅ 读代码生成配置
pull_requests: write ✅ 提交变更建议
actions: read ✅ 查看 CI 结果
contents: write（main） ❌ 防止绕过 PR 门禁
actions: write ❌ 防止自行触发部署
读取 secrets ❌ 结构上不可达

Broker 层约束：仅允许 github:create_pull_request、docker:build 等列举动作；禁止 github:update_secret、ssh:exec 到生产主机；每次调用记录审计日志。

---

7. 验收标准与安全验证

1. 全流程演示：学生项目从提交 PR 到云服务器稳定运行，全程可复现。
1. 攻击测试：
   · Agent 尝试在 workflow 中输出 secret → CI 扫描/审批门禁拦截；
   · Agent 尝试直接 push main → 分支保护拒绝；
   · Agent 尝试触发生产部署 → token 无 actions: write，拒绝；
   · 健康检查失败 → 自动回滚成功。
1. 幂等验证：连续执行部署脚本两次，服务状态与版本一致。
1. 审计验证：审计日志完整记录 Agent 每次调用及结果。
1. 凭据零暴露：日志、镜像、构建产物中扫描不到真实凭据。

---

8. 交付物

· 容器化后的学生项目（Dockerfile、compose、反向代理配置、健康检查端点）；
· GitHub Actions CI/CD workflow（含 Environment、审批、concurrency、回滚）；
· Agent 权限配置与 Broker 配置（占位符/注入/审计）；
· 障碍项清单与门禁设计文档；
· 安全验证报告（攻击测试结果）；
· 部署与运维手册（预检、回滚、排障）。

---

9. 风险与缓解

风险 缓解
云服务器网络不可达 优先 Pull 模式；self-hosted runner 放同 VPC；部署脚本重试+退避
Agent 生成配置质量不稳定 CI 强制 lint/测试；人工审批门禁；预检脚本
时间不足 阶段二可先只做 CI；CD 采用最小可行 Pull 模式
学生环境异构 限定 1–2 类目标环境；用 Ansible/容器抹平差异

---

10. 团队与时间建议

· 角色：1 名项目负责人（架构与门禁设计）、1–2 名学生开发（容器化与代理）、1 名安全评审（攻击测试与权限审计）。
· 周节奏：每周一个可演示增量；阶段末做门禁评审。
· 总周期：8–10 周，阶段四预留 2 周用于安全验证与报告。

---

一句话总结：本项目交付的不只是一个“能跑的服务”，而是一套 可审计、可回滚、Agent 权限被结构性限制的端到端部署流水线；障碍项在每阶段开始前显式列出，并通过门禁设计提前拦截。
