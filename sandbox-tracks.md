# 沙箱与运行时：平台方案与自建方案（平行双轨）

两套方案并行交付：**Track-A 使用金山云星源 AgentKit 平台能力**（KSADK/AgentEngine 运行时、KCI 沙箱、MCP、Skills），**Track-B 全自建**（本地沙箱容器 + 构建执行器 + Broker 直连）。两轨共用同一套上层组件（Broker、发布协议、Deployer、审计），差异只收在**沙箱与执行底座**一层，通过 plugin 抽象层隔离。

## 一、Sandbox Provider 抽象层（plugin 架构）

所有沙箱能力收敛为一个抽象接口（生命周期、命令执行、文件交换、网络策略），Broker / 构建调度器只依赖接口，不感知后端，也不绑定具体实现语言：

```
SandboxProvider 接口（语言无关的契约描述）

  Name() -> "agentkit" | "local"

  /* 生命周期 */
  Create(Spec) -> Instance
    // Spec: 镜像/模板ID、专用 UID、workspace 卷、资源上限、超时、
    //       网络策略（见 NetworkPolicy）、挂载白名单
  Destroy(Instance)
    // 任务结束即销毁

Instance 接口
  Exec(Cmd) -> ExecResult
    // 沙箱内执行命令（shell 工具后端）
    // ExecResult: 退出码 + stdout/stderr 截断 + 耗时
  ReadFile / WriteFile(path, ...)
    // workspace 内文件交换（patch 生成用）
  NetPolicy() -> NetworkPolicy
    // 实际生效策略自报，供验收核对

NetworkPolicy
  AllowEgress: [broker、构建执行器、受控包源]
  DenyMeta: 云元数据/loopback 代理/DNS 重绑定防护
  DNSStrategy: "stub-resolver"
```

两个实现均为独立 plugin，按需选择接入方式，不引入运行时动态加载：

```
sandbox/
  provider           # 抽象接口 + Spec/NetworkPolicy 定义 + plugin 注册
  local/             # Track-B：本地容器后端
  agentkit/          # Track-A：KCI 沙箱后端
  contract/          # Provider 契约测试：任何后端必须通过的
                     # 能力与隔离测试（SEC07、网络 DENY、超时、销毁）
```

**关键设计：契约测试套件（contract tests）与后端无关。** SEC07（读宿主 .ssh/环境变量/Docker socket/云元数据全部拒绝）、网络 DENY、资源上限、销毁后零残留——这些用例直接针对 `Provider` 接口编写，Track-A 和 Track-B 跑同一套。这既是安全验收，也是"平台能力是否达到我们隔离标准"的实测答案（契约测试跑不过就说明 KCI 粒度不够，直接用证据说话）。

### 两后端的差异点（诚实标注）

| 维度 | local（自建） | agentkit（KCI） |
| --- | --- | --- |
| 隔离实现 | 本机容器 + 专用 UID + nftables 规则，我们完全掌控粒度 | 平台托管（E2B SDK），粒度依赖 KCI 模板配置；VPC 内资源访问可控 |
| 元数据/DNS 防护 | 自配 stub resolver + 显式 DENY | 需实测模板网络选项是否覆盖；契约测试裁决 |
| 构建环境 | 自维护构建镜像（docker/compose/多语言工具链） | KCI"实例提交为镜像"能力，模板预热池加速冷启动 |
| 弹性 | 单机上限 | 平台弹性，多任务并发友好 |
| 部署依赖 | 仅需一台受信主机 | 依赖金山云账号与平台可用性 |

## 二、Track-A：平台方案（AgentKit）

**组件替换清单：**

| 自建组件 | 平台替代 | 接入点 |
| --- | --- | --- |
| Orchestrator 脚本 | KSADK（LangGraph）+ AgentEngine Serverless 运行时 | Agent 主循环用 LangGraph 写，`agentengine` CLI 部署 |
| 沙箱 + 构建执行器 | KCI 沙箱（自定义模板） | `agentkit` Provider 实现，E2B SDK |
| Broker 四动作 | Broker 不变 + **自定义 MCP Server** 包装（Streamable HTTP，代码方式部署） | 协议外壳，无新信任边界 |
| DevOps 流程知识 | **Skills 中心**：containerize-node/spring/go、fix-ci-failure、security-review-app | 流程层，只调四动作，不扩工具面 |
| Webhook 通知 | notifier（受信侧）→ AgentEngine 运行时 API 唤醒 Agent 会话 | 事件驱动替代轮询 ci.read |

**原则**：MCP 给 Agent 手（能力，四动作白名单不变），Skills 给 Agent 脑（流程）；AgentKit 凭证按官方凭证分层放控制台密文变量，绝不进 Agent 会话上下文；Skill Runtime 需自建包含运行时的专用沙箱镜像（官方文档明确提醒通用 Sandbox ≠ Skill Runtime）。

**Track-A 专属风险**：KCI 网络粒度以契约测试实测为准；平台可用性纳入故障分类（平台不可用 → 沙箱任务排队，不影响已运行生产——生产链路完全不经过 AgentKit）。

## 三、Track-B：自建方案

全部跑在受信主机/自管虚拟机上：

- Orchestrator：简单 Go 调度器（起容器、注册会话、TTL 回收），无外部依赖。
- 沙箱：`local` Provider——一次性容器，专用 UID 2000，workspace 卷，nftables 白名单出口（仅 Broker、构建执行器、包源 proxy），stub resolver 防 DNS 重绑定，DENY 169.254.169.254。
- 构建执行器：独立 rootless BuildKit 实例，固定参数、超时、资源上限。
- Broker 直连（REST，原生协议），无 MCP 层——接口语义与 Track-A 的 MCP 工具一一对应。
- Webhook：同一个 notifier，唤醒本地 Agent 进程（systemd socket 或本地队列轮询）。

## 四、与上层架构的关系（两轨不变的部分）

Broker 策略管线、proposal 校验、哈希链审计、获批 manifest 发布协议、Deployer、Environment 人工审批——**全部与沙箱后端无关**，因为 Provider 抽象保证了两轨对上层呈现完全一致的能力与隔离语义。演示叙事也顺：跑 Track-B 证明"不依赖任何云平台也成立"，切一个环境变量换 Track-A 证明"平台化接入零改造"。

## 五、排期影响（23 天）

- D 线（沙箱/Broker）改造：D3–5 先交付 `local` Provider + 契约测试（它是基准实现）；D9–11 并行加 `agentkit` Provider（E2B SDK + 模板镜像），跑同一契约测试。
- 契约测试在 D3 冻结，作为两轨共同验收门禁。
- MCP 包装层与 Skills 属 Track-A 增量，排 D12–15（与 Broker 收尾并行），不影响 Track-B 关键路径。
