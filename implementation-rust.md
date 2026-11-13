# Rust 实现方案：技术选型与工程细节

> 状态：待原型验证的实施设计，不代表已有实现或已通过安全验收。
>
> 本文是 [implementation.md](implementation.md) 的 Rust 备选方案，不替换原 Go 方案。安全目标、发布授权和验收口径继承 [idea.md](idea.md)；组件范围、Caddy、部署档位和 23 天交付目标沿用 implementation.md。应用仍然语言无关。

## 一、结论与适用条件

**Rust 能覆盖 Proposal Broker、Deployer 和沙箱构建执行器控制端的全部核心能力，没有已知的必需生态缺口。最适合发挥 Rust 优势的是部署状态建模、严格输入边界和错误分类，而不是本项目并不紧缺的吞吐量。**

采用条件：

1. Broker 和 Deployer 各有能独立交付 Rust 工程的负责人，已掌握 Tokio、Serde、错误处理和基本 Linux 进程模型。
2. 首版限定单机、单应用、单关系数据库、一个受信镜像仓库和明确的 Ubuntu LTS 版本；先交付 linux/amd64，其他架构另行验证。
3. 不自研容器运行时、沙箱内核、Git 实现、分布式工作流或通用 Agent 框架。
4. D1–2 冻结协议，早期验证真实 GitHub API、Caddy 和崩溃恢复，不把外部集成留到演示前。
5. 外围脚本和黑盒测试保持原有语言，不以“全部 Rust 化”为目标。

若团队主要熟悉 Go、Rust 尚处于学习阶段，23 天期限内应优先采用原 Go 方案。Rust 的编译期约束不能替代授权协议、权限隔离和故障演练；Go 也能正确实现这些能力。

## 二、组件边界与不变的安全架构

| 组件 | 选型 | 职责与边界 |
| --- | --- | --- |
| Proposal Broker | Rust 常驻服务 | 四动作 API、会话鉴权、策略、GitHub App、审计；不执行不可信构建 |
| Deployer | Rust 单次执行 CLI + systemd timer | 拉取授权、校验、蓝绿部署、journal、恢复；无远程通用执行 API |
| Build Executor | Rust 控制端 + 独立临时构建环境 | 固定构建预设、资源限制、作业回收；不持有 GitHub 或生产凭据 |
| Agent | 已有 Agent 工具与无凭据沙箱 | 通过 Broker 调用有限动作，不要求用 Rust 重写模型调用层 |
| Bootstrap | Bash | 安装 Docker/Compose/Caddy、目录权限、systemd、环境检测 |
| GitHub setup | Python + gh/API | 仓库、rulesets、Environment、CODEOWNERS 幂等配置 |
| 验收测试 | Python pytest + Rust tests | Python 验证外部边界，Rust 验证内部规则 |
| 业务应用 | 任意语言 | 只要求 HTTP、容器、优雅退出和迁移兼容契约 |
| 基础设施 | Docker Compose、Caddy、GitHub Actions/GHCR | 继续使用，不因换语言改变发布信任链 |

```text
Agent 沙箱（不持有 GitHub/生产凭据，仅有受限 Broker 会话凭证）
    │ repo.read / proposal.submit / ci.read / sandbox.build
    ▼
Rust Broker ──受限作业协议──> 独立 Build Executor / 临时构建环境
    │ GitHub App：仅目标 app 仓库、最小权限
    ▼
app PR → 人工审查 → main 可信构建 → digest + candidate
    ▼
ops 发布请求 → 受保护 workflow → production 人工审批 → manifest
    ▼
Rust Deployer（仅出站）→ Compose 双槽位 → Caddy → 观察/提交或回滚
```

“Agent 无 token”准确指无 GitHub token 和生产凭据，不是完全没有调用 Broker 的认证材料。会话凭证必须短期、可撤销、绑定沙箱与固定仓库，且不能换取更大权限。

### 明确不做

- 不改写业务应用，不将应用框架绑定到 Rust。
- 不给 Broker 增加任意 Shell、HTTP 代理、SSH 或 Docker API。
- 不将 Broker 和生产 Deployer 合并进同一高权限进程。
- 不将 `approved: true`、Release 作者、镜像 label 或 tag 当作授权证据。
- 不把 Rust 内存安全、rootless 或哈希链描述为完整安全保证。

## 三、Rust 技术栈

### 3.1 依赖选择

| 能力 | 首选 | 用法与约束 |
| --- | --- | --- |
| 异步运行时 | `tokio` | Broker 并发 I/O、GitHub 请求、健康探测、子进程管理 |
| HTTP 服务 | `axum` + `tower` / `tower-http` | 四动作路由、鉴权、body 大小、超时和并发限制；会话配额单独实现 |
| HTTP 客户端 | `reqwest` + Rustls | 固定 API 和健康地址；显式超时、重定向、代理、响应大小策略 |
| GitHub 客户端 | `octocrab` | App/installation 认证和常用 API；缺失端点通过受控适配层补齐 |
| JSON/数据类型 | `serde`、`serde_json` | 严格结构体；所有不可信嵌套对象拒绝未知字段 |
| JSON Schema | `jsonschema` | 固定草案、固定本地 Schema；关闭或拒绝外部网络/文件引用 |
| CLI/配置 | `clap`、`toml` | CLI 子命令和宿主配置；拒绝远端配置覆盖安全策略 |
| 时间 | `time` | RFC3339、有效期；运行超时使用单调时钟 |
| 错误 | `thiserror`；入口可用 `anyhow` | 领域层结构化错误，二进制入口加上下文；不输出秘密 |
| 可观测性 | `tracing`、`tracing-subscriber` | JSON 运行日志；审计事件使用独立格式和写入通道 |
| 哈希/秘密包装 | `sha2`、`secrecy`，必要时 `zeroize` | 哈希链、减少误打印；不能保证秘密从未复制或绝对擦除 |
| Linux 能力 | 标准库 + `rustix` | 文件锁、进程信号及目录操作；依赖冻结时核对具体 API |
| 测试 | `proptest`、`wiremock`、`tempfile` | 属性测试、GitHub mock、临时状态目录；不能替代实机测试 |
| 依赖审查 | `cargo-audit`、`cargo-deny` | 已知漏洞、许可证、来源；安全例外须有原因与到期日 |

依赖名称是选型，不是已验证的版本组合。D1–2 选定相互兼容的具体版本，提交 `Cargo.lock`、精确工具链版本的 `rust-toolchain.toml`，在干净 CI 中以 `--locked` 验证。禁止未经审查的 Git 分支依赖和不必要的默认 feature；保留实际 feature 清单及依赖树证据。

### 3.2 异步与阻塞边界

- Broker 使用 Tokio；GitHub 网络请求和执行器作业等待均设超时、并发上限和取消策略。
- Deployer 每次只处理一个发布事务，不为各状态步骤额外建立并发任务网络。
- journal 的 `fsync`、目录同步等阻塞操作放入受控阻塞边界；异步任务收到完成通知后才能继续副作用。
- 不持有跨 `.await` 的全局互斥锁；审计单写者、token 缓存和作业表分别管理。
- 不将 `spawn_blocking` 的 future 被取消当成底层工作已停止；写 journal 的最终结果必须可核实。

### 3.3 构建与交付

首版采用明确目标环境上的 GNU/Linux 构建，优先减少原生依赖，但不预先承诺全静态链接。若选择 musl，需另测 DNS、TLS、CA 证书、系统调用及目标主机兼容性。

Broker 交付非 root 容器；Deployer 交付主机二进制、systemd 单元和固定模板；执行器控制端部署在独立构建基础设施。所有二进制输出版本、源码提交和构建标识，禁止在 `/version` 中泄露环境变量。

## 四、代码组织与信任边界

建议在受保护的控制面/ops 仓库内维护一个 Cargo workspace。被 Agent 修改的 app 仓库不放 Broker、Deployer、workflow 策略和主机模板的权威副本。

```text
control-plane/
  Cargo.toml
  Cargo.lock
  rust-toolchain.toml
  crates/
    protocol/                 # DTO、协议版本、领域值类型
    broker/
      src/{api,auth,policy,gitops,cifilter,buildsched,token,audit}/
    deployer/
      src/{manifest,authorizer,contentbind,preflight,journal}/
      src/{statemachine,bluegreen,caddy,lock,reconcile}/
    executor/
      src/{jobs,presets,runtime,limits,cleanup}/
    infra/                    # 小范围共享：受控命令、原子文件写、日志
  schemas/                    # manifest、Broker 请求、journal、审计
  fixtures/                   # 合法/非法样本、跨语言 canonical JSON 向量
  templates/{compose,caddy,systemd}/
  scripts/{bootstrap.sh,setup_github.py}/
  tests/{security,deployment,data,operations}/
  docs/runbooks/
```

共享代码不等于共享权限。`protocol` 不应依赖 GitHub token 或主机执行能力；`infra` 不应演变成可被远端调用的通用命令接口。构建产物分别打包，不把 Broker 私钥和生产读凭据同时装进一个镜像。

核心概念：

- `RawManifest`：尚未授权，只能进入校验流程。
- `ValidatedManifest`：结构与本地策略通过，尚不代表人工审批有效。
- `AuthorizedRelease`：授权核验模块唯一构造，包含接受时证据和规范化哈希。
- `RecoveredTransaction`：仅由本地恢复模块从可信 journal 和现实状态重建，不通过反序列化直接获得“已授权”标记。
- `ImageDigest`、`Generation`、`Slot`、`ConfigRevision`：使用独立类型，避免参数混淆。

不要为 `AuthorizedRelease` 暴露公共字段或直接实现不受约束的 `Deserialize`。类型约束减少误调用，但程序漏洞和错误的授权实现仍可破坏保证。

## 五、Proposal Broker 实现细节

### 5.1 API 与入口治理

建议固定四个 POST 路由，与协议动作一一对应：

| 路由 | 输入概要 | 输出概要 |
| --- | --- | --- |
| `/v1/repo/read` | 允许的 ref、相对路径 | 限长文本/目录、blob 标识 |
| `/v1/proposals/submit` | base_sha、patch、标题、说明、幂等键 | 提案分支、commit、PR |
| `/v1/ci/read` | 当前提案关联 run_id、分页位置 | 脱敏结论和限长日志 |
| `/v1/sandbox/build` | 绑定会话的 workspace_id、固定 preset、幂等键 | 作业结果或受限作业句柄 |

作业查询如需独立路由，只能读取该会话已创建的作业，不增加新的可执行能力。`repo_id` 由会话绑定；即使协议携带该字段，也必须与绑定值一致。

处理顺序：请求上限 → 会话鉴权 → Schema/类型校验 → 对象归属和策略 → 审计意图 → 外部操作 → 审计结果。错误响应只给稳定错误码、request_id 和脱敏说明，不返回底层认证 header 或内部文件路径。

可用初始限制：JSON 请求体 1 MiB、单文件读取 256 KiB、最多 100 个补丁文件、返回日志 1 MiB；构建并发每会话 1 个、全局按主机容量限制。这些是待压测的起始配置，不是验收结论。HTTP body、解压后日志、Git blob 和输出缓冲都要分别限额。

会话凭证使用密码学安全随机值，服务端仅保存其哈希及会话元数据；绑定沙箱、仓库、到期时间、动作、速率和总量。会话请求不允许替换 workspace 的实际目录或执行器地址。

### 5.2 GitHub App 与权限

- App 私钥只在 Broker 的受限 secret 文件中；安装范围仅 app 仓库，不包括 ops。
- 按需要授予 Contents/PR 写入、Actions 读取等最小权限；不添加管理、Environment 管理、workflow 写入或 ruleset bypass 能力。
- GitHub token 的权限并不天然限定到 `agent/` 分支；必须同时使用 Broker 策略和实际 ruleset 阻止 main 写入及越权合并。
- installation token 在 Broker 内缓存，提前续期；并发刷新采用 single-flight，避免同时签发大量 token。
- 不向 Agent 返回原始 GitHub 客户端错误、认证对象或完整 CI archive。
- SDK 没有覆盖的 run attempt、job、Release 等端点，由单一 GitHub 适配层补齐，统一处理分页、API 版本、限流和日志脱敏。
- 401/403 不进行无限重试；429、临时 5xx 和网络错误有限退避，遵守 `Retry-After`。

验收必须直接以 App 身份尝试 main 写入、合并和禁止动作，不能仅测试 Broker 会拒绝。

### 5.3 Patch 的安全应用

推荐首版只支持 UTF-8 文本文件的新增、修改和删除；拒绝二进制补丁、submodule、符号链接、文件模式变更，以及未明确支持的复杂 rename/copy。需要移动文件时显式提交删除与新增，仍检查两个路径。

处理流程：

1. 校验 base_sha 是该会话允许基线；从固定仓库读取该提交树，不接受任意 Git URL。
2. 解析 patch 的所有路径、hunk 和大小；拒绝绝对路径、`..`、NUL、控制字符、非规范形式。
3. 对旧路径、新路径、父目录和最终结果统一执行策略；拒绝 `.git`、`.github`、所有受保护位置的 CODEOWNERS、策略及扫描忽略文件。
4. 首版只允许目标范围内的普通文件；从 Git tree 检查模式和父级，避免跟随 symlink 或 gitlink。
5. 在无凭据、限资源的 patch 处理区对精确 base 应用，禁用 hook、外部 diff/filter、submodule 等扩展行为；不 checkout 到 Broker 的宿主工作目录执行仓库脚本。
6. 再次验证最终树和完整 diff，不仅检查输入 patch 的表面头部。
7. Broker 生成 `agent/<session>/<proposal-id>`，通过 GitHub API 写 blob/tree/commit/ref 并开 PR；不 force push 未确认归属的分支。

Broker 的写 API 超时后先查询既有分支/commit/PR，再决定是否补做；幂等键绑定请求摘要，不允许同键不同内容。与其他语言交换路径时明确 UTF-8 和大小写规则，测试 Git 特殊路径，不依赖宿主平台碰巧如何解释路径。

Agent 对 workflow 等受保护文件的修正只能作为文本建议/patch 附件交给维护者，不能为了“CI 自动修复”绕过保护。

### 5.4 CI 结果与日志过滤

`ci.read` 必须核对 run 对应当前提案、提交和允许的 workflow。对 GitHub 日志下载重定向采用独立允许策略，禁止把 Authorization 转发到不受信目标；限定压缩包条目数、解压大小和处理时间。

优先返回结构化检查结论与允许的错误片段，再返回脱敏日志。过滤已知 token 模式、认证头、环境变量 dump、ANSI 转义和日志注入内容。扫描不是秘密隔离的替代：持有生产秘密的 job 不应执行 PR 代码，其日志也不应进入 Agent 通道。

### 5.5 审计

每条事件包含序号、request_id、actor/session、动作、参数摘要、策略版本、允许/拒绝原因、外部对象 ID、结果、时间和前序哈希。

采用明确的跨语言规范化规则，例如 RFC 8785 JCS；选择兼容库前必须通过 Rust/Python 测试向量。不要用普通 `serde_json::to_string` 或“仅排序顶层字段”冒充规范化 JSON。JSON 入口先拒绝重复键，再进行 Schema 验证和规范化；不能先解析为已丢失重复键信息的 `Value` 再检查。

建议链格式固定为：

```text
hash = SHA256(domain_separator || previous_hash_bytes || canonical_event_without_hash)
```

使用单写者串行分配序号；关键写动作在本地审计意图持久化后才能执行。远端日志暂时不可达时写持久 outbox；本地审计失败或积压达到上限时拒绝新的写动作并报警。重启后恢复发送，链头周期性锚定到独立日志端。

本地 hash chain 不能阻止 root 重写整条链，远端锚定之间仍有未覆盖窗口。审计记录操作，不记录模型内部思考、完整 patch 秘密或完整环境变量。

## 六、发布协议与授权核验

### 6.1 不因换语言改变 Manifest

继续使用 idea.md 的版本 1 契约：

- 发布身份：`schema_version`、`app_id`、`environment`、`release_id`、`generation`。
- 构建来源：`source_repository_id`、`source_sha`、`build_run_id`。
- 部署内容：`image`、`platform`、`config_revision`、`migration_mode`。
- 审批来源：`ops_commit`、`approval_run_id`、`approval_run_attempt`、`expires_at`。

不增加任意 command、volumes、URL、secret_path 或 Compose 文本。远端 manifest 无权配置 CORS/TLS 档位。GitHub 数字 ID 的 JSON 表示方式须在 Schema 中统一，Rust 侧使用无损解析，不通过浮点数转换。

验证流水线：

```text
响应字节限制/UTF-8/重复键检查
→ 固定 JSON Schema（包含嵌套 additionalProperties=false）
→ 严格 DTO 与领域类型
→ 宿主允许列表
→ GitHub 来源和审批核验
→ 不可变请求内容绑定
→ generation/有效期/历史冲突检查
→ 规范化哈希与授权证据持久化
```

对 digest、SHA、时间、平台做显式解析，不假定 Schema 的 `format` 一定默认执行断言。只有全部核验通过后才能更新可信 generation 水位，避免伪造一个极大序号阻塞后续合法发布。

### 6.2 Authorizer 的完整检查

1. 从固定 ops repository ID 查询 Release，不按未验证的资产 URL任意下载。
2. 查询指定 run 和 attempt，核对仓库、workflow ID、分支、事件类型、head_sha、状态和结论。
3. 检查指定 attempt 中受审批发布 job 实际成功；完整处理 jobs 分页，不能只看整个 workflow 的 success。
4. 按不可变 head_sha 校验受信 workflow 与其调用脚本的版本/摘要；可复用 workflow、action 和配置也需纳入信任策略，不跟随可变引用。
5. 按同一 head_sha 读取 `requests/prod.json`，逐项比较 source_sha、build_run_id、image、config_revision、migration_mode。
6. 校验固定 Release 命名、关联的实际 ops 提交和 attempt；不能把可变 tag 或字符串 `main` 当成不可变提交证明。
7. `generation` 必须等于可信 API 的该 workflow `run_number`；有效期不能超出可信 run 时间与本地主机最大授权时长限定。
8. 检查镜像库、平台、配置版本及本地接受记录；同一 release_id 内容变化直接拒绝。

候选来源仍由受保护 release workflow 验证，不能仅相信 candidate.json 自报字段。批准发布的 job 不执行 app 代码。Environment 的人工审批配置由 setup 和独立验收确认；job 名称本身不能证明发生了人工审批。

GitHub Release 资产不视为天然签名对象。MVP 依赖 GitHub/TLS、受保护 workflow、管理员和主机策略；离线验证若成为需求，另设计签名、签名者约束、密钥轮换和撤销。

### 6.3 序号、重试与幂等

持久记录 `highest_seen_generation`（已授权接受的最高序号）、`last_successful_generation`、active、previous、pending、failed release 集合及 manifest hash。

- 相同 release_id/hash 且仍是当前成功版本：核对实际运行状态，返回 `ALREADY_APPLIED`，不重新迁移或启动容器。
- 相同 release_id 不同 hash，或相同 generation 不同发布内容：拒绝冲突。
- 低序号及已被后续版本替代的旧成功发布：拒绝复活。
- 已失败版本：隔离，不因 timer 重复轮询无限重试。
- workflow rerun 的 run_number 不会因为 attempt 增加而自然变成更高序号。首版对已接受或失败的发布要求新 workflow run / 新批准，不使用 rerun 绕过同序号冲突。
- 自动回滚是当前已接受事务的补偿，不降低 generation 水位；人工发布旧镜像须有更高 generation 的新批准。
- config_revision 不同即不同发布，不能只按 image digest 判重。

授权过期阻止新事务开始，但不阻止对已持久接受、已产生副作用的事务进行安全恢复。恢复只能使用原来绑定的内容，不允许借“恢复”替换镜像或扩权。

## 七、Deployer 状态机与持久化

### 7.1 运行模型

提供 `deployer run-once`、`deployer status` 和受限诊断命令；手工触发同样走授权与主机锁。不提供任意镜像、任意 shell 或绕过审批的 `--force` 发布入口。

systemd timer 初始轮询间隔 30–60 秒；所有改变部署状态的入口使用同一稳定锁文件和 Linux 文件锁。锁文件不删除重建，不通过文件是否存在判断锁是否持有；避免锁 FD 泄漏给子进程。

每次启动先锁定并恢复 pending transaction，再查询新授权。GitHub 不可达时继续提供当前版本；本地恢复不应无条件依赖 GitHub 当前可用。

### 7.2 状态与副作用

| 状态/阶段 | 操作 | 失败处理 |
| --- | --- | --- |
| Validated | 授权、内容绑定、序号、有效期通过并持久记录 | 不执行 Compose |
| Prepared | 资源预检、按 digest 拉取、迁移条件验证 | 保持旧服务，记录可重试或终止原因 |
| CandidateReady | 非活跃槽位启动、readyz/version/smoke 通过 | 清理已确认归属的候选，不碰旧槽位 |
| Switched | Caddy 已 reload，代理入口确认候选 | 观察期间故障分类后回滚或报警 |
| Committed | 观察通过，原子提交 active/previous | 保留回退资源和完整证据 |
| RollingBack | 恢复旧代理配置并核对业务 | 恢复失败进入 ManualRequired |
| RolledBack / Failed | 终态与失败版本隔离 | 不反复执行失败发布 |
| ManualRequired | 真实状态无法判定或旧版本也失败 | 停止自动发布，保留现场 |

首次部署无 previous，失败使用 `INITIAL_DEPLOY_FAILED`，不得报告“回滚成功”。

### 7.3 Journal 格式与写入

首版采用单机单写者 JSON 状态文件，不引入分布式锁或数据库。状态文件一次原子更新包含 generation 水位、active/previous/pending 与失败记录，避免多个文件分别成功而形成不一致。

pending 至少保存：journal 版本、release_id/hash、已接受 manifest、授权证据摘要、旧/新槽位、镜像与配置版本、代理旧配置位置和哈希、当前阶段、意图/结果、时间、错误、部署档位快照。

每次有副作用的操作：

1. 写同目录临时文件，限制权限，避免符号链接覆盖。
2. 写完整数据并 `sync_all` 临时文件。
3. 在同文件系统内原子 rename 到目标。
4. 打开父目录并同步目录项。
5. 确认意图持久化成功，才执行外部动作。
6. 观察实际结果，再用同样流程持久化结果。

任何同步错误都阻止进入下一步。校验 journal 大小、版本和结构；损坏或未知版本时 fail closed，不自动重置为空状态。目录仅受信账户可写，首版不支持网络文件系统。完整 fsync 的耐久性仍受文件系统、存储和虚拟化平台影响，需实机验证。

### 7.4 崩溃恢复

恢复依据是 journal + 实际代理配置/运行状态 + 容器 digest/标签 + 经代理的版本和业务结果，不是单一 `current.json`。

| 崩溃点 | 重启后处理 |
| --- | --- |
| 记录候选启动意图后、结果前 | 检查候选容器实际存在与归属，继续验证或清理，不盲目再建 |
| 代理磁盘文件已替换但未 reload | 核对运行态；按事务策略继续切流或恢复旧配置 |
| reload 成功但 Switched 未持久化 | 通过受限本地管理信息和实际入口识别流量目标；重建观察窗口或回滚 |
| 观察通过但 Committed 未写入 | 重新验证，不仅凭“曾经通过”提交 |
| 回滚过程中再次退出 | 幂等继续恢复旧配置并检查旧业务 |
| 代理、journal、容器互相矛盾 | 不猜测；ManualRequired 并告警 |

`Drop`、析构和内存状态机不能处理断电或 SIGKILL。测试必须覆盖“意图写入后/副作用后/结果写入前”等窗口；SIGKILL 测试不等同于掉电耐久性测试。

## 八、Compose、Caddy 与外部进程

### 8.1 受控命令执行层

只暴露固定能力，例如 `pull_candidate`、`start_slot`、`inspect_slot`、`validate_proxy`、`reload_proxy`，不暴露 `exec(string)`。

- 使用受信绝对路径和独立 argv；不使用 `sh -c` 拼接远端输入。
- `env_clear` 后只注入必要环境；固定工作目录、PATH、Docker 配置目录，不继承代理、HOME 或调用者的 Compose 覆盖变量。
- 即使不用 Shell，也必须拒绝选项注入和不可信配置文件；参数只来自已验证类型及受信模板。
- 关闭 stdin；并发排空 stdout/stderr，限内存、限磁盘，超过上限后丢弃多余输出或按策略终止，不能堵住管道导致死锁。
- 每个操作设置超时，管理进程组/作业 cgroup，终止后等待和回收直接子进程。
- `kill_on_drop` 只是兜底；`timeout` 丢弃 future 不等于命令已退出，也不等于整棵进程树已停止。
- Compose CLI 被杀后，Docker daemon 可能仍执行已收到的操作。超时后必须 inspect 并协调真实状态，不能立即当作“未执行”。

测试命令层时使用假可执行程序模拟挂起、大输出、退出码和后代进程；生产配置不能被 Agent 改成该测试程序。

### 8.2 双槽位与镜像身份

固定两个 project：`student-app-blue`、`student-app-green`，宿主绑定 `127.0.0.1:18080/18081`。数据库独立 project，发布时不重建数据库。Compose 模板、secret 挂载和资源上限均来自受信宿主配置。

按 `registry/repository@sha256:...` 拉取，禁止 tag 漂移；同时核对平台、容器实际镜像和来源标识。多架构 index digest、平台 manifest digest、Docker image config ID 不是同一概念，不能直接把三个字符串要求相等；首版可限定单架构产物以减少歧义。

通过固定标签记录 app_id、release_id、manifest_hash、slot，清理时同时核对对象归属。严禁使用无范围的 prune/down 清理共享资源。备用 registry 若不能保持协议认可的 digest，需重新生成和批准对应发布，不能默默替换镜像地址。

### 8.3 Caddy 切流

1. 从受信模板渲染候选配置，只替换允许的槽位 upstream 与本地档位参数。
2. 保存并持久记录旧配置及哈希。
3. `caddy validate` 验证候选配置。
4. 写切流意图并持久化。
5. 原子替换受信 Caddyfile，调用固定 reload 入口。
6. 经真实代理入口验证版本、readiness 和无破坏性的业务 smoke。
7. 完成观察窗口后提交；失败时恢复磁盘旧文件、再次 reload、验证旧业务。

Caddy 管理接口仅供受信本地控制面访问，优先使用受限 Unix socket 或严格限制的本地端点，不暴露给 Agent 或业务容器。应用容器使用独立网络，不能因“绑定 loopback”就假设任何隔离方案下都不可达。

validate 成功不代表 reload 成功，reload 返回成功也不替代实际流量验证。观察时间初始取 60–120 秒，回滚计时从检测失败到旧业务恢复，不把完整观察窗口混入回滚耗时。

## 九、构建执行器与沙箱

### 9.1 MVP 隔离方案

优先使用专用构建节点上的临时 VM；环境内无 GitHub/生产秘密、无宿主目录、无生产网络路由。若使用 rootless BuildKit/容器方案，必须单独验证宿主边界和网络边界，不能宣称与 VM 等价。

Broker 只调度受信执行器，不能直接挂载 Docker socket。执行器若需控制 VM/构建运行时，其管理凭据留在控制端，绝不进入执行用户代码的 guest。生产 Deployer 的 Docker 权限与构建执行器完全分离。

作业协议包含 `job_id`、会话、不可变工作区快照标识、preset、资源预算和截止时间。上传快照拒绝路径穿越、特殊设备、逃逸链接和解压炸弹；Broker/执行器不接受 Agent 提供的宿主绝对路径。

### 9.2 固定预设和生命周期

首版预设为有限的 `test`、`container-build` 等；每种语言的命令由受信模板选择。即便命令固定，项目脚本、编译器插件和 Dockerfile 仍可执行任意代码，因此隔离不可省略。

执行流程：验证归属 → 幂等登记 job → 创建临时环境 → 复制快照 → 配额内执行 → 收集限长脱敏结果 → 销毁环境。执行器重启后扫描已登记作业，回收孤儿环境；超时和取消不仅终止客户端等待，还要实际结束作业。

作业默认不向 GHCR 推镜像，沙箱构建结果仅用于反馈。生产候选仍由 app main 的可信 CI 重新构建和发布，不将 Agent 提交的镜像视为可信候选。

### 9.3 依赖下载与网络策略

网络禁令要区分“禁止生产/内网访问”和“完全离线”。MVP 选择并记录以下一种模式：

- 预置依赖和镜像缓存，构建阶段无网络；缓存缺失时给出明确失败。
- 在独立无秘密下载阶段通过受控代理访问固定依赖源，随后进入离线构建。

不能简单允许任意外网 HTTPS；访问控制要覆盖 DNS 解析后的地址、重定向、IPv6、link-local、云元数据、内部网段及 DNS 重绑定。Rust 控制端只负责配置和验证，最终边界由网络命名空间、防火墙、代理或 VM 网络实现。

缓存按信任域分离，不允许不可信构建写入会被可信发布构建直接信任的共享缓存；依赖 lockfile、校验和与镜像 digest 都要保留。

## 十、主机权限、部署档位与数据库

### 10.1 权限现实与 systemd

生产 Deployer 需要 Docker 控制能力、写部署状态和切换 Caddy。控制 rootful Docker socket 通常等价于较高宿主权限，不能因为进程 UID 非 root 就声称已完成最小权限隔离。

MVP 将 Deployer 明确列为受信高权限主体：二进制、配置、模板和 systemd 单元由管理员维护，不接受 app 内容作为配置。Broker 和沙箱部署在其他权限域，生产上无面向 Agent 的监听端口。

根据实际需求验证 systemd 的 `NoNewPrivileges`、文件系统只读、读写路径允许列表、资源上限和 syscall 限制；不机械复制会破坏 Docker/Caddy 的配置。主机 secrets、App 私钥、镜像拉取凭据按主体分别提供，避免以 CLI 参数传入秘密。外部进程和重定向日志同样禁止泄露认证信息。

### 10.2 档位沿用但明确限制

`/etc/student-app/profile.conf` 继续是受信档位配置，可采用严格 TOML 语法但不改变控制权归属：

- `tls_mode=http/internal/auto`，`cors_mode=open/allowlist`，以及固定访问方式。
- http 仅用于无真实敏感数据的受控演示，不声称具有传输保密性。
- internal 证书由测试客户端显式信任 CA，不全局关闭 TLS 校验。
- CORS open 只适合不携带跨域凭据的演示；不能把 `Access-Control-Allow-Origin: *` 与凭据请求混为一谈。
- 统一管理 OPTIONS、允许方法/头和响应头，避免代理与应用重复输出冲突 CORS 头；CORS 不是服务端鉴权。
- 档位进入 journal 和告警；Agent、manifest 和镜像均不能修改它。

### 10.3 数据迁移和备份

Rust Deployer 不内嵌通用 SQL 执行 API。首版仅 `none/expand`，由受信迁移入口使用独立 DDL 身份执行，绑定审核后的迁移内容、checksum、锁和超时。`expand` 不是任意 SQL 的安全标签，仍须审查和 N/N−1 兼容性测试。

双槽位不各自自动迁移；数据库和持久文件不随 Compose 应用槽位清理。应用回滚不撤销数据库写入、外部调用或不可逆动作。

备份继续使用数据库原生工具和既有脚本，Rust 只记录前置条件及结果。保留每日一致性备份、异地副本和空白实例恢复演练，独立测量 RPO≤24h、RTO≤60min。

## 十一、错误、观测与重试

建议稳定错误码分为：`AUTH_*`、`POLICY_*`、`GITHUB_*`、`MANIFEST_*`、`JOURNAL_*`、`PREFLIGHT_*`、`CANDIDATE_*`、`PROXY_*`、`ROLLBACK_*`、`SANDBOX_*`。

每个错误明确三项：是否允许有限重试、是否需要补偿、是否要求人工处理。禁止把所有错误统一视为“重试整个发布”。

- 授权、Schema、hash 和路径策略失败：拒绝，不重试副作用。
- 网络、限流：预算内退避；写 API 先查结果再重试。
- 候选失败：保持旧服务，隔离本次发布。
- 切流后应用版本故障：事务内回滚。
- 数据库/主机公共故障：分类告警，不在两槽位间反复切换。
- 状态存储异常或恢复无法判定：停止自动化并保留现场。

部署事件记录 release_id、generation、source_sha、digest、config_revision、approval_run/attempt、阶段耗时、active/previous、结果和错误码。监控指标不把每个 release_id 当成长期高基数标签；详细关联信息留在日志与证据包。

## 十二、测试与 CI

### 12.1 Rust 内部测试

| 层次 | 重点 |
| --- | --- |
| 单元测试 | Schema、领域类型、权限、generation、有效期、错误分类 |
| 属性测试 | 任意非法路径被拒绝、状态转换不遗漏、规范化稳定、序号不倒退 |
| GitHub mock | attempt、分页、job 跳过、资产变更、限流、超时、重定向 |
| 命令层测试 | 固定 argv、环境清理、输出上限、子进程取消、后代进程回收 |
| journal 测试 | 写入失败、磁盘满、损坏/未知版本、各副作用窗口恢复 |
| 集成测试 | 真实 Docker/Compose/Caddy 的 digest、切流、回滚和幂等 |
| 实机故障 | SIGKILL、主机重启、网络故障；必要时单独做掉电模拟 |

外部接口 mock 只能验证自己的逻辑；至少一个专用 GitHub 测试仓库运行真实 App、ruleset、Environment 和 Release 流程，确认套餐支持人工门禁且 API 假设成立。

### 12.2 必须继承的黑盒验收

按 idea.md 的 SEC01–SEC10、DEP01–DEP07、DATA01–DATA02、OPS01–OPS02 执行，不因语言变化减少验收。额外增加 Rust 实现重点用例：

- 重复 JSON 键、未知字段、超大和深度异常输入、非法 Schema 外部引用。
- 高 generation 伪造记录不能污染可信水位。
- 同 run 不同 attempt、同序号冲突与授权过期后的事务恢复。
- timeout 之后 CLI/子进程/daemon 副作用的实际状态。
- Caddy 磁盘配置与运行态不同步时的恢复。
- 本地审计写入失败、远端日志不可达、outbox 满时的行为。
- App token 无法绕过分支保护；Agent 无法访问 Caddy 管理口和生产 Docker。
- 无 secrets 的构建环境、缓存隔离与依赖下载网络控制。

指标沿用：固定 20 次正常批准发布至少 19 次成功；定义的 10 次故障演练全部恢复；越权必测全部拒绝；应用回退目标≤120秒；重复 manifest 零新增部署副作用；灾难恢复独立实测。可用性连续 7 天观察从系统稳定后才开始计时，未满窗口不得声称已完成；短期样本不推断长期 SLA。

### 12.3 CI 入口

依赖冻结后提供稳定入口，例如：

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --locked -- -D warnings
cargo test --workspace --locked
cargo build --workspace --release --locked
cargo audit
cargo deny check
pytest tests/security tests/deployment tests/data tests/operations
```

以上为计划命令，当前文档仓库尚无 Cargo 工程，不能直接运行。特权集成测试应在独立临时 runner/VM 执行，不在持有生产秘密的 runner 上运行不可信 PR。

Rust 的 `build.rs`、过程宏和依赖构建同样能够执行代码。控制面源码构建也要分离检查、构建与发布凭据；不能因使用 Cargo 就绕过 CI 信任分层。

## 十三、23 天实施顺序与止损门禁

以下排期以有 Rust 经验的四人团队为前提，是实施预算，不是交付保证。A 负责应用/数据，B 负责 CI/权限，C 负责 Deployer，D 负责 Broker/执行器。

| 阶段 | A 应用/数据 | B CI/权限 | C Deployer | D Broker/执行器 | 门禁 |
| --- | --- | --- | --- | --- | --- |
| D1–2 | 应用契约、示例范围 | GitHub 套餐/门禁、setup/bootstrap | 工具链、Schema、journal、Compose/Caddy 原型 | API、威胁模型、App API spike、隔离方案 | 冻结协议；确认真实 API 和目标主机可用 |
| D3–5 | 两语言应用本地闭环 | PR/main 构建、digest/candidate | 授权、锁、原子写、最小切流/恢复纵向链路 | repo.read、会话、隔离构建原型 | 原型通过；失败则缩非核心范围或采用 Go，不删安全边界 |
| D6–8 | 迁移兼容测试 | 发布请求、SBOM、扫描 | preflight、拉取、双槽位 | proposal.submit、路径策略、ci.read | App 无越权，候选失败不影响旧服务 |
| D9–11 | 端到端 smoke | Environment、获批 manifest | authorizer 全量、Caddy、幂等 | 构建作业协议、幂等和回收 | 真实批准发布完整跑通 |
| D12–15 | 备份恢复 | 权限负面测试 | 全状态机、回滚、崩溃恢复 | sandbox.build、审计/outbox | Agent 接入前，控制面边界及恢复基本可验 |
| D16–18 | 攻击与业务支持 | 批准证据 | DEP 故障注入 | Agent 闭环、SEC 测试 | 禁止靠开放权限修联调问题 |
| D19–20 | 全量验收与证据 | 同左 | 同左 | 同左 | 记录原始失败和重试，不只保留最后成功 |
| D21–23 | 彩排、文档、缓冲、演示 | 同左 | 同左 | 同左 | 标注未完成观察窗口与已知限制 |

### 早期 Rust 原型的最小判定集

用 2–3 天作为探索预算，在授权测试环境中验证：

1. GitHub App token 与真实 run/job/attempt/Release API 能正确调用。
2. 固定 digest 的候选能启动并经 Caddy 切流。
3. 候选失败、切流后失败有正确结果，不把“当前能访问”当成完整成功。
4. 在至少一个切流前窗口和一个切流后窗口 SIGKILL，重启后能解释并收敛状态。
5. 超时清理、重复执行和旧 generation 拒绝有可检查证据。

原型允许使用固定可信 fixture 隔离部分部署逻辑，但不得据此宣布真实授权验证完成。若这些基础行为无法在预算内稳定演示，停止继续堆功能，先解决集成风险。

可以延后的内容：额外语言模板、多架构、离线签名、UI、复杂调度、更多缓存优化。不可延后的内容：生产审批、Agent 权限隔离、来源绑定、重放防护、journal、回滚和关键负面测试。减少范围时应明确标注偏离原完整版，不继续宣称“全部完成”。

## 十四、交付物与完成定义

1. 三个独立 Rust 组件及可重复构建入口、工具链锁定、依赖清单和版本信息。
2. 版本化 Schema、领域错误码、跨语言 canonical JSON 测试向量。
3. Broker 四动作、GitHub App 权限、会话管理、路径策略、审计及撤销 runbook。
4. Deployer 授权核验、序号与幂等、状态机、journal、Caddy/Compose 适配、恢复 runbook。
5. 独立构建环境、受限作业协议、资源和网络策略、孤儿作业回收。
6. Bootstrap/setup、systemd 单元、受信模板、secret 目录与轮换说明。
7. 两个不同语言应用的完整流水线，以及数据库兼容、备份恢复证据。
8. SEC/DEP/DATA/OPS 原始结果、部署事件和审计链验证记录。
9. 临时档位、信任假设、已知限制和尚未完成的观察项清单。

完成标准不是“代码用 Rust 写完”，而是同一安全目标在真实边界上通过验收。Rust 方案若不能提供与 Go 方案相同的授权和故障证据，不视为等价替代。

## 十五、参考与待确认事项

### 一手技术资料

- [Tokio Command：进程配置与取消行为](https://docs.rs/tokio/latest/tokio/process/struct.Command.html)
- [Tokio Child：drop 与子进程生命周期](https://docs.rs/tokio/latest/tokio/process/struct.Child.html)
- [Octocrab：GitHub API 客户端](https://docs.rs/octocrab/latest/octocrab/struct.Octocrab.html)
- [Octocrab GitHub App 认证示例](https://docs.rs/crate/octocrab/latest/source/examples/github_app_authentication_manual.rs)
- [jsonschema：草案支持、校验与外部引用](https://docs.rs/jsonschema/latest/jsonschema/)
- [Serde container attributes](https://serde.rs/container-attrs.html)
- [RFC 8785：JSON Canonicalization Scheme](https://www.rfc-editor.org/rfc/rfc8785)
- [Caddy 命令行](https://caddyserver.com/docs/command-line)
- [GitHub workflow jobs REST API](https://docs.github.com/en/rest/actions/workflow-jobs)
- [GitHub deployments / environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/managing-environments-for-deployment)

`latest` 文档仅用于选型参考，正式实现须记录锁定版本对应的 API 和 feature。

### D1–2 必须确认

- 团队 Rust 经验及三个组件的实际维护人。
- Ubuntu LTS 版本、CPU 架构、磁盘类型与蓝绿容量。
- GitHub 套餐和仓库可见性是否支持要求的审批/保护门禁。
- App 的实际最小权限、job 标识和 run attempt API 行为。
- 构建节点/临时 VM 来源、生命周期、依赖下载和缓存边界。
- Rust/Python 共用 Schema、重复键策略、规范化算法和 ID 表示。
- Caddy 管理接口权限、内部 CA 信任与正式 TLS 升级方式。
- 依赖精确版本、编译目标、锁文件、安全扫描和二进制分发方式。

**最终决策：已有 Rust 能力则采用 Rust 控制面；以交付为先且团队熟悉 Go 时保留 Go。无论选择哪种语言，协议、安全边界和验收证据必须保持一致。**
