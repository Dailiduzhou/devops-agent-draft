# 执行底座：SandboxProvider 与 BuildExecutor 分离

> 当前决策：拓扑 B；Agent 主循环在受信运行时，shell 与仓库代码在独立沙箱，真实镜像构建由独立 BuildExecutor 承担。首版交付一个通过门禁的组合，第二后端后置；不是双轨同时交付或等强度隔离承诺。凭证与身份见 [agent-runtime.md](agent-runtime.md)。

## 一、两类接口，不把构建能力强塞进沙箱

```text
受信工具适配器 ── shell / 文件操作 ──> SandboxProvider
        │                              └─ 独立 workspace 沙箱
        └─ 冻结并登记内容快照 ───────────────┘
        │
        └─ broker_call(sandbox.build) → Broker → BuildExecutor
                                                   └─ 独立临时构建环境
```

SandboxProvider 管理编辑、命令和文件工作区；BuildExecutor 管理构建作业。两者可以使用同一云厂商，但不共享可写 workspace、凭证或跨任务可写缓存。构建消费不可变快照，不读取一个仍被 shell 修改的目录。

### SandboxProvider 契约

```text
Create(SandboxSpec) -> Instance
Destroy(Instance) -> CleanupResult
Instance.Exec(Cmd, Budget) -> ExecResult
Instance.ReadFile / WriteFile(workspace_relative_path, ...)
Instance.ExportSnapshot() -> SnapshotRef
Instance.ObservePolicy() -> PolicyObservation

SandboxSpec: 模板、专用 UID、允许挂载、网络模式、CPU/内存/PID/磁盘配额、TTL
SnapshotRef: 内容摘要、归属 session、字节数、文件清单、不可变存储引用
```

ExportSnapshot 必须冻结写入或用一致性快照机制；归档拒绝逃逸路径、特殊设备、symlink/gitlink、解压炸弹。快照存储与句柄由受信协调端登记，模型不能提供下载 URL 或宿主路径。ObservePolicy 是配置/观测信息，不是安全通过证明。

### BuildExecutor 契约

```text
Submit(BuildSpec) -> JobID
Status(JobID) -> JobState
Result(JobID) -> BuildResult
Cancel(JobID)
Reconcile() -> OrphanCleanupResults

BuildSpec: session、snapshot_id、固定 preset、幂等键、资源预算、deadline
BuildResult: 退出状态、构建/业务 smoke 结果、限长脱敏日志、产物摘要、回收状态
```

只有 Broker/受信调度方可以提交和管理作业。模型仍只看到四个 Broker 动作，不新增任意远程执行 API。BuildExecutor 控制端若持有 VM/沙箱管理身份，该身份不能进入 guest。guest 不持有模型、平台、Broker、GitHub 或生产凭据，不接触生产网和宿主 Docker socket。

沙箱构建产物只供反馈，不直接成为生产候选；正式候选由受保护 main CI 针对获批 source_sha 重新构建，并绑定 digest。

## 二、可选后端组合与首版选择

| 工具沙箱 | 构建执行器 | 决策条件 |
| --- | --- | --- |
| KCI | KCI 上独立临时构建实例 | 真实构建、权限、网络、资源与清理测试全部达标后可用 |
| KCI | 专用节点上的临时 VM | KCI 能满足工具沙箱隔离但不满足构建条件时，优先回退组合 |
| local 一次性容器/VM | 专用临时构建环境 | 平台沙箱不达标时的备选；仍需独立验证 |

D1–3 验证 KCI 工具沙箱和真实构建；依据证据选择一个合格组合。无法验证的能力标记“未验证”；不因首版预算不足而使用 privileged、宿主 Docker socket 或无限制网络凑结果。云端运行时若是比赛硬性要求，沙箱后端回退不等于放弃 AgentKit 运行时接入。

local 后端不是“天然安全的基准”：共享内核容器与 VM 的隔离假设不同。构建节点与生产主机必须分开；rootless 降低权限，不自动保证网络、进程和缓存隔离。BuildKit 的 user namespace、mount、snapshotter、PID namespace、seccomp/AppArmor 需求须在目标系统验证，不为构建随意关闭宿主防护。

Provider 的替换只覆盖其接口，不自动替换 AgentEngine、模型、MCP、Skills、会话存储和 GitHub。后端预置完成且能力满足时可通过配置切换；完整离线系统还需独立替换代码托管、registry、模型、依赖源与授权验证协议，不属于 MVP。

## 三、网络与资源策略

### 工具沙箱

默认禁止外联；不直连 Broker、模型、平台管理 API、GitHub 或生产网络。源码/结果由受信控制端通过 Provider API 交换。平台控制通道所需端点单独审查，不向 guest 暴露管理凭据。必须使用沙箱实际配置验证，不能假设模板默认安全。

### 构建环境

MVP 优先使用受信预置依赖和基础镜像、离线构建；缺少缓存时报明确错误。如果示例必须在线下载，则显式选择“独立无秘密下载阶段 + 受控代理 + 后续离线构建”，不得自动回退任意外联。

下载代理限定源、方法、路径与响应大小，覆盖解析后地址、重定向、IPv6、link-local、云元数据、内部网段和 DNS 重绑定。允许包源/DNS 不等于没有外传通道；还需评估可控包名/URL、DNS 隧道和依赖安装脚本。缓存按不可信任务和可信发布构建分域，记录依赖 lockfile、校验和与基础镜像 digest。

CPU、内存、PID、磁盘/inode、输出量与时间均设上限。取消不仅停止等待，还需终止作业进程树并核对真实运行状态。守护端重启后回收孤儿任务。原始日志、镜像、卷和缓存按策略保留/清除；预热实例不得复用上一任务 workspace 或身份。

## 四、验证体系：共享行为用例 + 后端专属证据

共享测试面向契约，不要求两个后端已同时交付，也不把通过同一套有限测试解释为隔离强度等价。

| 层级 | 必测内容 | 证据 |
| --- | --- | --- |
| 功能 | Exec、文件交换、快照一致性、真实 Dockerfile 构建、启动 HTTP smoke | 命令结果、快照摘要、镜像 digest、业务响应 |
| 网络 | 允许目标正向控制；生产网、宿主、元数据、IPv6、重定向、DNS 重绑定拒绝 | guest 结果、受控接收端记录、有效策略配置 |
| 文件/身份 | 宿主目录、socket、平台/模型/Broker/GitHub canary 不可读；跨会话快照拒绝 | 实际文件结果、认证拒绝与审计 |
| 资源/生命周期 | 超时、后台进程、输出限额、资源上限、取消、重启回收 | cgroup/平台指标、作业及清理记录 |
| 跨任务残留 | 新实例无法读取上一任务标记、无有效旧身份、无未授权缓存共享 | 两轮任务及撤销证据 |
| 后端专属 | local 宿主策略核查；云端模板/API/供应商隔离说明与可观测限制 | 配置版本、平台返回、明确不可观测部分 |

网络与文件负例分别在 shell、依赖脚本、Dockerfile RUN 和可执行 Skill 中测试。云租户不能直接证明物理宿主磁盘完全擦除；销毁验收限定为租户可观测的资源释放、身份撤销和跨任务不可达，不声称“物理零残留”。

## 五、首个 PoC 与止损

1. 固定平台模板/SDK/BuildKit/基础镜像版本与一个带真实依赖的 HTTP 应用。
2. 在目标限制下构建并启动 smoke，保存实际权限与挂载配置。
3. 同时执行隔离正负测试，不把整体断网误判成策略正确。
4. 注入超时、后台进程、资源超限，验证取消和销毁。
5. 创建下一任务检查残留；记录不可观测项与限制。

通过后再实现 Provider 兼容和性能优化。KCI 不能安全构建时拆分 BuildExecutor；KCI 工具沙箱隔离本身不达标时换工具沙箱后端。无合格后端则暂停不可信执行，不能仅标“风险接受”后继续声称安全成立。
