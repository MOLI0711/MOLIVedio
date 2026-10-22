# Go 视频知识分析平台开发文档

版本：V1.1 开发基线
日期：2026-10-22
配套文档：[完整技术实现方案](./video-agent-technical-design.md)

本手册用于从零开发视频上传、资料处理、检索问答和 Agent 管理功能。按 M0 到 M5 的顺序完成开发任务，每项任务都要有可检查的输入、产物和验收结果。架构、状态机与版本规则沿用配套技术方案；这里落实到代码边界、配置、数据库迁移、联调和测试。

文中的目录、配置键、CLI 和 Makefile 目标是待实现的项目约定，当前交付不包含这些应用代码。代码示例只覆盖标明的接口；具体依赖和模型版本在 M0 连通测试后锁定。

## 1 开发目标与实施路线

交付一个 Go 仓库和一个 Vue 工作台：用户上传有语音的课程或技术视频，查看章节摘要，通过文字提问，必要时由 Agent 查看相关画面，最后用可点击证据定位到视频原处。管理页支持 Agent 草稿、发布版本、运行记录和费用信息。

首版采用 Go/go-zero、Eino、MySQL、Redis、Kafka、Elasticsearch、SeaweedFS/COS、FFmpeg、Tesseract 和云模型 API。etcd、多视频问答、MCP、K8s、链接下载和模型训练不进入本轮开发任务。

| 阶段 | 形成的能力 | 开始下一阶段前的检查 |
|---|---|---|
| M0 服务连通 | 环境、供应商适配、依赖版本和样本 | 上传、ASR、向量、工具调用和图片输入分别跑通 |
| M1 视频资料库 | 登录、上传、媒体处理、字幕与截图定位 | 5 个样本可处理；中断上传可续传；时间位置可核验 |
| M2 检索基线 | 建索引、双路召回、固定检索问答与引用 | 30 道问题有基线；权限和索引版本过滤通过 |
| M3 Agent 工作台 | Eino、看图、摘要、会话、配置发布、SSE | 追问复用素材；回答能回到原证据；页面重连不丢状态 |
| M4 可靠执行 | Kafka 接管、自动恢复、取消、异常治理 | 关键故障注入通过；过期执行不能发布结果 |
| M5 评测交付 | 评测、性能报告、监控、部署和演示 | 实验可复核；完整环境可从空库启动 |

M1 就建立持久任务表、基本租约和提交校验；M4 再完善消息接管、自动回收与故障覆盖。不要到 M4 才给内存任务系统补数据库状态。

## 2 环境准备

### 2.1 本机与外部服务

本机准备 Git、Go、Node.js、Docker Compose、FFmpeg/ffprobe、Tesseract。Go 和 Node 的具体版本必须与项目依赖兼容，写入版本文件；OCR 语言包必须包含 chi_sim 与 eng。媒体程序若只在 Worker 容器内运行，则在容器内执行检查。

```bash
go version
node --version
docker compose version
ffmpeg -version
ffprobe -version
tesseract --list-langs
```

外部配置包括 COS 私有 bucket、ASR 凭证、文本/视觉/Embedding 模型权限。先用短样本验证，避免把完整视频当成第一个付费请求。ASR 与 VLM 只收到当前操作需要的音频或截图。

本地媒体保存在 SeaweedFS；长音频通过 COS 暂存供云 ASR 下载。云端不能访问开发机的 localhost 地址。VLM 使用图片请求体或它可访问的短期 URL。上传、ASR、图像理解三个网络路径要分别验证。

### 2.2 两种地址配置

| 调用方 | 应使用的地址 | 常见错误 |
|---|---|---|
| 容器内 Worker | Compose 服务名，例如 mysql:3306 | 在容器里用 localhost 连接其他容器 |
| 宿主机运行的 API | 映射到宿主机的端口 | 使用只有容器 DNS 才能解析的服务名 |
| 用户浏览器 | 浏览器实际可访问的 API/S3 公共端点 | 签发包含容器服务名的上传 URL |
| 云 ASR/VLM | 可从供应商网络访问的对象地址或请求体 | 把内网 S3 地址传给供应商 |

S3 上传签名使用与浏览器请求一致的 host、路径风格及签名参数，前端不能自行替换 URL 的 host。开发 bucket 配置正确的 CORS，并向浏览器暴露分片上传需要读取的 ETag；完成时仍由服务端向存储核验分片。

### 2.3 版本清单

在 docs/dependency-lock.md 记录 Go/Node、Go modules、前端 lockfile、数据库与中间件镜像 digest、FFmpeg/Tesseract、OCR 语言包、模型 ID/维度和验证日期。提交 go.mod、go.sum 和 package-lock.json，前端统一使用 npm。

Go 依赖通过固定版本引入，执行 go mod tidy 后检查变更，不把一次无关升级混入业务提交。依赖管理机制参考 [Go 官方文档](https://go.dev/doc/modules/managing-dependencies)。

## 3 代码结构与模块职责

```text
video-agent/
  cmd/
    api/main.go
    worker/main.go
    coordinator/main.go
    devtool/main.go
  api/openapi.yaml
  internal/
    api/                 路由、DTO、认证、CSRF、SSE handler
    app/                 应用服务与依赖装配
    domain/              ID、状态、配置快照、错误与结果类型
    repository/mysql/    SQL、事务与迁移兼容
    upload/              分片上传和存储对账
    workflow/            步骤依赖、领取、重试、取消、租约
    media/               ffprobe、音频、抽帧、OCR、时间轴
    retrieval/           ES 写入、BM25/kNN、RRF、证据选择
    agent/               Eino、Prompt、工具、预算与引用校验
    provider/            ASR、Chat、Vision、Embedding 客户端
    storage/             S3/COS、对象 key、签名、manifest
    event/               Outbox、Kafka、Inbox、持久 SSE
    observability/       slog、Trace、指标
  web/src/
    api/                 HTTP 与 SSE 客户端
    pages/               视频、会话、Agent、运行详情页
    components/          播放器、证据、上传、进度组件
    stores/              页面状态和事件去重
  configs/
  migrations/
  deploy/
  eval/
  tests/fixtures/
  tests/integration/
  docs/
  Makefile
```

依赖方向：HTTP handler → 应用服务 → 领域接口；MySQL、ES、S3 和供应商客户端实现接口，由入口装配。领域对象不依赖 HTTP、Eino 或具体数据库客户端。

handler 负责参数、身份和响应映射，应用服务负责业务流程，repository 方法封装完整事务。go-zero 的路由或生成代码只作为传输层，不在多个生成的 logic 中复制任务状态转换。

API 不执行 FFmpeg、不等待模型生成、不持有长事务。Worker 不读取 HTTP Cookie；它使用经过授权并持久化的运行上下文。模型工具不能接收客户端随意指定的 owner_id 或内部对象路径。

## 4 配置和开发命令

### 4.1 配置文件约定

使用 configs/local.example.yaml 和 configs/container.example.yaml。下面展示项目自定义配置结构；以 _env 结尾的字段表示加载器要解析的环境变量名称，不能直接当作凭证值，也不假设 go-zero 自动完成这层解析。

```yaml
app:
  env: local
  http_addr: "127.0.0.1:8080"
  public_origin: "http://localhost:5173"
mysql:
  dsn_env: "VA_MYSQL_DSN"
redis:
  addr: "127.0.0.1:6379"
  password_env: "VA_REDIS_PASSWORD"
kafka:
  brokers: ["127.0.0.1:9092"]
  media_topic: "video.media.dispatch"
  agent_topic: "video.agent.dispatch"
search:
  endpoint: "http://127.0.0.1:9200"
  api_key_env: "VA_ES_API_KEY"
  index_name: "video-segments-embed01-map01"
  embedding_profile: "embed01"
storage:
  backend: "s3"
  internal_endpoint: "http://127.0.0.1:8333"
  browser_endpoint: "http://127.0.0.1:8333"
  bucket: "video-agent-dev"
  access_key_env: "VA_S3_ACCESS_KEY"
  secret_key_env: "VA_S3_SECRET_KEY"
  path_style: true
providers:
  asr_ref: "tencent-asr-dev"
  chat_ref: "chat-dev"
  vision_ref: "vision-dev"
  embedding_ref: "embedding-dev"
workflow:
  dispatch_mode: "db_dev"
  lease_seconds: 120
  heartbeat_seconds: 30
  max_failure_retries: 3
limits:
  upload_max_bytes: 2147483648
  video_max_duration_ms: 7200000
  upload_part_bytes: 8388608
  upload_parallelism: 3
  ffmpeg_concurrency: 1
  ocr_concurrency: 2
  model_concurrency: 2
agent:
  max_model_iterations: 6
  max_tool_calls: 8
  max_vision_calls: 2
  max_frames_per_call: 4
  execution_timeout_seconds: 180
```

provider_ref 指向服务端配置表：包含固定模型 ID、凭证环境变量名、允许的 endpoint、超时、模型能力和计费字段。Embedding 的实际维度必须在 M0 填写并与 ES mapping 校验，不给一个未验证的默认维度。

配置加载顺序为默认值 → 指定 YAML →明确支持的环境覆盖。缺少启用功能必需的密钥或模型 ID 时启动报错；关闭该功能时不要求无关密钥。日志只打印脱敏配置。

db_dev 是 M1～M3 的开发分发方式：创建步骤与 Outbox 后，由开发接管器调用同一个 AcceptDispatch 事务，不发送 Kafka；接管成功后标记事件已交付，避免持续扫描同一条记录。M4 将唯一分发入口切为 kafka，移除 db_dev 分支。阶段状态仍经过 QUEUED → READY，不另造一套状态机。

### 4.2 必须实现的命令入口

| 命令 | 应执行的动作 | 成功条件 |
|---|---|---|
| make doctor | 工具版本、配置、依赖可达性检查 | 脱敏报告；必需项失败时非零退出 |
| make infra-up | 启动当前阶段所需的 Compose 服务 | 依赖健康检查通过 |
| make migrate | 执行尚未应用的迁移 | 并发迁移互斥；已应用迁移重复执行无副作用 |
| make seed | 按已完成迁移初始化开发用户、默认 Agent 和小样本清单 | 可重复执行，不重置已有密码或覆盖业务数据；0005 前只创建用户和素材清单 |
| make api | 宿主机启动 API | readiness 通过 |
| make worker | 启动媒体/Agent 开发执行器；M1 内含基本协调循环 | 心跳、依赖推进、外部轮询到期分发可用；注册步骤与并发限制可见 |
| make coordinator | 启动 M4 消息与恢复协调器 | Outbox/Inbox、重试、租约扫描可观测 |
| make web | 启动 Vite；代理 /v1 到 API | 浏览器同源调用 API/SSE |
| make test-unit | 无外部依赖的 Go 单元测试 | 不需要云密钥 |
| make test-integration | 独立测试库和真实中间件测试 | 不访问开发/生产库，不调用付费模型 |
| make test-fault | 可控故障注入 | 每个场景有数据库和产物断言 |
| make smoke-live | 小样本真实供应商连通测试 | 显式启用；报告实际调用和费用来源 |
| make eval | 固定配置运行冻结问题集 | 输出逐题明细、指标和配置清单 |

Compose 用 search、queue、observe 等 profile 按阶段启用可选服务；基础数据服务默认启用。启动完整环境要显式包含全部所需 profile，不能把未启动依赖的日志误当业务失败。机制参考 [Docker Compose profiles](https://docs.docker.com/compose/how-tos/profiles/)。

### 4.3 启动和检查顺序

在命令实现后，从新代码仓库根目录运行：

```bash
cp configs/local.example.yaml configs/local.yaml
make infra-up
make doctor
make migrate
make seed
```

分别在终端运行 make api、make worker、make web；M4 再运行 make coordinator。检查 /healthz 的进程存活和 /readyz 的必需依赖，不以日志出现“started”替代可用性检查。关闭时停止领取新工作、给已有调用有限收尾时间，再退出；媒体子进程也必须结束。

## 5 统一数据契约

### 5.1 命名和时间

数据库字段 video_version_id 与 API/检索字段 content_version 指向同一个不可变版本 ID，不能分别生成。视频位置统一为整数毫秒；数据库业务时间为 UTC DATETIME(6)。step_key 标识具体步骤实例，例如 asr:chunk-03；step_type 标识执行器类型，例如 ASR，避免运行代码依赖任意字符串拆分。

索引版本由 index_name + index_build_id 表示；索引构建失败重试使用新的 build_id。ASR 的 submit_generation、队列的 dispatch_seq、执行的 attempt_no、失败的 retry_count 分开保存。

### 5.2 检索接口示例

以下为可独立定义的领域接口。scope 由服务器加载，不直接映射模型提供的 JSON。工具 Schema 只暴露查询词、有限时间范围等业务参数。

```go
package contracts

import (
    "context"
    "time"
)

type ExecutionPermit struct {
    JobID      string
    StepID     string
    WorkerID   string
    AttemptNo  uint64
    LeaseUntil time.Time
}

type RetrievalScope struct {
    OwnerID        string
    VideoID        string
    ContentVersion string
    IndexName      string
    IndexBuildID   string
}

type SearchInput struct {
    Query   string
    StartMS *int64
    EndMS   *int64
    Limit   int
}

type Evidence struct {
    ID             string
    Kind           string
    ContentVersion string
    StartMS        int64
    EndMS          int64
    FrameAtMS      *int64
    Text           string
    FrameIDs       []string
}

type SearchResult struct {
    Evidence []Evidence
    Mode     string
}

type Retriever interface {
    Search(context.Context, RetrievalScope, SearchInput) (SearchResult, error)
    Read(context.Context, RetrievalScope, []string) ([]Evidence, error)
}
```

ExecutionPermit.LeaseUntil 用于本地提前停止，最终提交仍由数据库时间与条件更新裁决。句子/片段引用使用 StartMS/EndMS；截图引用使用 FrameAtMS，验证器按 Kind 区分区间与时刻，不能为单帧伪造持续时间。Evidence 不携带长期签名 URL，展示时凭资产 ID 鉴权签发。

### 5.3 关键接口及事务边界

| 接口/方法 | 输入与产出 | 必须保证 |
|---|---|---|
| UploadService.Create/Complete | 上传声明、幂等键 → 会话或 video/job | 合并对象与数据库不构成一个事务，需要完成对账 |
| JobRepository.AcceptDispatch | event_id、step_id、dispatch_seq | Inbox 与 QUEUED→READY 同事务 |
| JobRepository.ClaimReady | worker_kind、容量 → 执行 permit | 短事务增加 attempt，设置租约 |
| JobRepository.Renew | permit → 新租约 | 过期或旧 attempt 返回 ErrLeaseLost |
| JobRepository.CommitResult | permit、manifest → 阶段提交 | 验证 job/step；完成、下游 Outbox 同事务 |
| JobRepository.ScheduleExternalWait | permit、external_job_id、下次轮询 | 保存当前远端 job 指针并释放执行槽 |
| JobRepository.Cancel/FailTerminal | job_id、原因 → 终态处理 | 撤销运行 token，清理活动会话标记 |
| ASRProvider.Submit/Poll | 固定输入与配置 / 远端 job_id | 状态不明独立返回，不伪装成明确失败 |
| ArtifactStore.Put/Head/Sign | 对象引用与内容 | key 带版本和 attempt；上传成功不代表业务发布 |
| RunRepository.SaveToolResult | 当前 permit、logical_call_id、结果 | 工具结果、用量与证据登记一致 |
| RunRepository.SaveCheckpoint | 完整消息组、恢复游标 | assistant/tool 配对，预算不回退 |
| EventRepository.Append/ListAfter | 当前执行权 / run、seq | 先持久化再推送；可重复读取 |

事务型方法返回完整业务结果，不暴露可被 handler 任意组合的 UpdateStatus。事务内不得调用模型、Kafka、S3 或 FFmpeg。通过 errors.Is/errors.As 区分 ErrNotFound、ErrConflict、ErrLeaseLost、ErrRetryable、ErrCommitUnknown、ErrSubmissionUnknown 和 ErrBudgetExceeded，传输层再映射错误码。

数据库明确回滚的死锁可以有限重试整个纯数据库事务；Commit 返回不确定结果时先读回幂等记录、步骤和已发布 manifest，再决定后续动作。数据库重试不能顺带重新发送已经发出的 ASR/LLM 请求。

## 6 数据库迁移与事务开发

### 6.1 迁移批次

| 批次 | 建立的表或字段 | 最早阶段 |
|---|---|---|
| 0001 身份与请求 | users、auth_sessions、quota_guards、api_idempotency | M1 |
| 0002 媒体与上传 | videos、video_versions、upload_sessions、media_assets | M1 |
| 0003 基础任务 | jobs、job_steps、job_attempts、external_jobs、outbox_events、inbox_events | M1 |
| 0004 视频内容 | transcript_sentences、video_frames、video_segments | M1 |
| 0005 Agent 与问答 | agents、agent_versions、sessions、messages、agent_runs、tool_invocations、model_calls、run_checkpoints、run_events | M2，先用默认配置 |
| 0006 队列异常与调度索引 | quarantined_events、必要扫描索引和约束 | M4 |

先创建表，再建立存在循环引用的关系；外键选择与删除策略写入迁移说明。原视频上传早于分析版本，允许其 version/step 关联为空，但 owner/video 归属不可缺失。配置快照 JSON 不承载需要频繁过滤的状态字段。

每份迁移有版本与 checksum。已经应用的迁移不直接修改，新增迁移修正。迁移命令加数据库互斥，失败退出，不自动删除数据“重来”。测试回滚只使用隔离测试库；正式数据变更优先向前修复。

### 6.2 优先建立的唯一约束

- api_idempotency：(owner_id, scope, idempotency_key)。
- job_steps：(job_id, step_key)；job_attempts：(step_id, attempt_no)。
- outbox_events：(step_id, dispatch_seq, event_type)；inbox_events：(consumer_name, event_id)。
- external_jobs：(step_id, submit_generation)，provider_ref/region 内约束请求 ID。
- video_versions：(video_id, generation)；video_segments：(video_version_id, ordinal)。
- messages：(session_id, seq)；run_events：(run_id, seq)；tool_invocations：(run_id, logical_call_id)。

READY 领取、租约回收、外部轮询、Outbox 发布分别需要匹配过滤条件的索引。用 EXPLAIN 和真实批量数据检查扫描范围，不能只在三条测试记录上判断性能。

### 6.3 事务必须遵守的顺序

跨表锁顺序：quota_guard → video → session → job → step → agent_run，同类多行按 ID 排序，不需要的行略过。终态操作涉及会话/视频时必须先锁这些父行，再锁任务。

| 事务 | 原子动作 |
|---|---|
| 完成上传/重处理 | 幂等与配额检查；在接纳请求时递增 desired_generation；建立版本、job、步骤、Outbox |
| 创建问答 | 锁会话并确认无活动 run；复制会话固定版本；创建消息、run、job、Outbox；设置 active_run_id |
| 接管消息 | 插入 Inbox；校验 dispatch_seq；匹配的 QUEUED 改 READY |
| 领取执行 | 校验 job；READY 改 RUNNING；增加 attempt；首次领取把 job 改 RUNNING |
| 发布成功 | 检查有效租约、当前 attempt 和允许执行的 job；发布产物；推进依赖/Outbox |
| 最终回答 | 在最后步骤提交中保存答案、助手消息和终态事件；更新 run/job；清理 active_run_id |
| 取消/最终失败 | 终结或取消相关步骤；增加运行步骤 token 并清租约；旧执行不能再提交 |

远端动作发生在事务外，所以先落意图、后执行、再提交结果。重复消息、重复点击和旧 Worker 的处理应依赖数据库条件与唯一约束，不依赖进程内 map。

## 7 M0 连通与定版任务

| 编号 | 要实现或确认的内容 | 验收产物 |
|---|---|---|
| M0-01 | 仓库骨架、配置加载、doctor、版本清单 | 缺必需配置可读报错；真实密钥不写日志 |
| M0-02 | S3 分片上传、ListParts、Complete、Head、签名读取；COS 暂存 | 一份小文件分片往返内容一致；浏览器端点可访问 |
| M0-03 | ffprobe/FFmpeg/Tesseract 小样本；ASR Submit/Poll | 保存音轨、实际帧时间、句级转写及原始响应 |
| M0-04 | 文本工具调用、结构化输出、视觉图像输入、Embedding | 固定模型 ID；返回维度与 ES 测试 mapping 一致 |
| M0-05 | mock 适配器与合成 fixture | 无云密钥可跑大部分开发测试；真实调用单独启用 |

provider smoke 程序每次只调用一类服务，输出脱敏请求 ID、耗时、状态和 usage 来源。文本工具调用用一个无副作用测试工具；图片用自制图形；Embedding 用固定中英短句。先确认供应商能力，再写复杂 Agent 逻辑。

## 8 M1 视频资料库开发任务

| 编号 | 代码与动作 | 完成标准 |
|---|---|---|
| M1-01 | internal/api 认证中间件、Cookie 会话、CSRF、初始化账号 | 用户 A 无法读取用户 B 资源；退出后 Cookie 无效 |
| M1-02 | internal/upload 与 API；浏览器上传组件 | 8 MiB 分片、最多 3 路；关闭页面后可按存储实际分片续传 |
| M1-03 | internal/workflow 基础状态、心跳、依赖推进、外部轮询到期分发、db_dev 接管器 | QUEUED→READY→RUNNING 可查询；重启后继续调度已持久化 ASR；重复接管不产生重复执行权 |
| M1-04 | internal/media 探测、音频、抽帧、OCR | 不经 shell 拼接；取消能结束进程组；实际时间戳可核验 |
| M1-05 | internal/provider ASR、SUBMITTING 意图、current_external_job_id、SUBMISSION_UNKNOWN 分支 | 重启后按 job_id 轮询原任务；状态不明不自动重提；正常轮询不增加失败次数 |
| M1-06 | BUILD_SEGMENTS、字幕列表和截图展示 | 句子和帧属于同一版本；字幕与截图可跳到对应视频位置 |

上传 Complete 必测“对象已合并，DB 未完成”的恢复：先核验原对象并执行同一幂等完成事务，不能要求客户端重新上传。multipart ETag 不作为整文件 MD5。

音频和画面统一到播放器 0 起点的毫秒时间。分块音频保留实际切割起点；字幕转为全局时间后再去重。OCR 失败只有在明确降级策略下才能标记 SKIPPED；ASR 无有效语音按首版范围返回不支持。

M1 只验收已实现媒体步骤。尚未接入的 BUILD_INDEX 不领取，不把“有字幕”提前标成“已可问答”。5 个样本用独立测试任务逐个核验，记录媒体产物后显式取消未完成任务并释放配额；不将永久 RUNNING 当成验收完成。整体 PREPROCESS 成功与 active_content_version 发布在 M2 索引接入后验收，阶段测试与整条流水线验收分开记录。

## 9 M2 检索与固定问答任务

| 编号 | 代码与动作 | 完成标准 |
|---|---|---|
| M2-01 | ES mapping、Bulk 写入、build_id 隔离 | 逐条处理部分失败；旧构建不可见；全部可搜才发布版本 |
| M2-02 | BM25、kNN、应用层 RRF、候选去重 | 两路同样过滤 owner/video/content/build；过滤前置于 kNN |
| M2-03 | Read、时间邻域、上下文裁剪 | 只返回限定证据；长视频不会把全部字幕塞入上下文 |
| M2-04 | 默认 Agent/会话/run；固定检索后单次生成 | 返回 claims/evidence_ids；服务端渲染正文；伪造引用被拒绝 |
| M2-05 | 30 问基线和检索评测脚本 | 逐题保留召回与评分；无证据问题不强行回答 |

先实现固定“检索→生成→验证”流程，再接 Eino 自主补检索。两者共用 Retriever、证据对象和输出校验，因此可以在相同数据上比较工具循环的实际收益。

查询向量与索引使用相同 embedding_profile。BM25 和向量分数不直接相加；RRF 按排名融合，初值 k=60。相同索引内容下比较 BM25、向量和混合召回，OCR 收益另做对照。

没有搜索结果时可返回 answer_status=insufficient_evidence；这属于成功完成的业务结果，不等同于 job FAILED。依赖异常、格式始终不合规、超时等才按错误规则处理。

## 10 M3 Agent 与工作台任务

| 编号 | 代码与动作 | 完成标准 |
|---|---|---|
| M3-01 | agent_versions 草稿、编辑、发布、停用 | 已发布版本不可改；旧 run 保留原配置 |
| M3-02 | Eino Runner、四个工具、服务端权限上下文 | 模型只能传业务参数；越权 ID 在工具内再次拒绝 |
| M3-03 | inspect_frames、补抽帧、VLM、证据注册 | 每次最多 4 帧、最多 2 次；画面结论对应真实帧 ID |
| M3-04 | 分层章节摘要、会话历史和必要压缩 | 每层保留原证据；追问不重新执行 ASR/OCR |
| M3-05 | 工具意图/结果、完整消息组、用量预留 | 已完成工具可复用；历史没有孤立 tool_call_id |
| M3-06 | SSE 持久事件、页面快照、重连、证据播放器 | 按 seq 去重；断线不取消；过期游标重新取快照 |

### 10.1 Eino 接入次序

先用记录式 fake model 返回确定的工具调用，再接真实模型。第一条路径只调用 search_segments 与 read_segments；引用验证通过后，再开放 inspect_frames 与 get_video_outline。

模型迭代上限和工具次数上限独立实现；多个工具在一次模型响应里出现时逐项计数。BudgetGuard 在实际调用前预留，调用后结算；LLM、VLM 重试都要记录，不交给多层隐藏重试放大请求。Eino 构造与中间件 API 对齐锁定版本，参考 [ChatModelAgent 官方文档](https://www.cloudwego.io/docs/eino/core_modules/eino_adk/agent_implementation/chat_model/)。

工具参数验证、权限、预算、引用与结果发布由程序执行，不能只写进 Prompt。各工具结果为结构化对象和有限摘录，不返回全视频、任意文件路径或未经授权链接。

### 10.2 工具契约

| 工具 | 模型参数 | 服务端追加或校验 | 返回 |
|---|---|---|---|
| search_segments | query、时间范围、受限 limit | 当前用户、视频、内容/索引构建版本 | 有来源标识的短摘录和证据 ID |
| read_segments | segment_ids、受限邻域 | ID 归属、数量、上下文预算 | 原始句子/OCR、帧 ID |
| inspect_frames | 时间窗、观察问题 | 访问、帧数、视觉预算、实际帧时间 | observations、frame_ids、limitations |
| get_video_outline | 可选章节范围 | 当前版本和摘要状态 | 章节及其原始证据；无摘要时返回明确状态 |

补抽帧保存后，当前有效 attempt 将其登记进资产和本次 evidence_snapshot。缓存命中也要重新鉴权并注册证据；缓存保存对象 key，不保存过期签名 URL。VLM 只能引用实际提供的帧，不能决定新的资源 ID。

### 10.3 崩溃恢复接入

保存模型完整工具调用消息与 PENDING 意图 → 执行工具 → 提交结果与用量 → 在同一短事务提交完整 assistant/tool 消息组、对应证据快照和新恢复游标 → 进入下一次模型调用。

恢复时先找已提交工具结果，补齐未完成项和配对消息，再以历史重新进入 Runner.Run。Eino 主动 interrupt 的状态才走框架 Resume 路径；普通崩溃不保证恢复任意指令位置。预算以数据库累计使用和预留为准，不能用旧 checkpoint 覆盖最新支出。

## 11 M4 可靠任务开发任务

| 编号 | 代码与动作 | 完成标准 |
|---|---|---|
| M4-01 | Outbox 发布租约、Kafka ack、固定 event_id | 发送成功后崩溃可以重发；不在 DB 事务里等 broker |
| M4-02 | Kafka 消费、Inbox、offset 管理 | DB commit 后才确认；旧 dispatch_seq 不改变步骤 |
| M4-03 | 自动回收租约、退避、调度、终态保护 | 旧 token 续租/提交失败；终态不倒退 |
| M4-04 | 完善 ASR 在途配额、提交不明故障注入和供应商找回验证 | 当前提交指针正确；轮询不重提；SUBMISSION_UNKNOWN 可核对 |
| M4-05 | 取消、删除、重处理、存储/索引孤立产物清理 | 取消先撤销发布权；新旧 generation 不互相覆盖 |
| M4-06 | Agent 工具边界恢复与 SSE 重放 | 已完成结果复用；证据和消息配对；预算累计 |
| M4-07 | Redis 故障策略、每用户任务配额、优雅停机 | 缓存失效不放开无限付费任务；退出不遗留子进程 |

消息只携带 schema_version、event_id、job_id、step_id、dispatch_seq、worker_kind 与 trace_id。媒体与模型结果经对象引用传递，Kafka 不传大文件。每个分区先串行接管短事务，执行池独立并发。

切换 Kafka 前停掉开发接管器，只为当前仍是 QUEUED 的 dispatch_seq 保留或补齐待发布 Outbox，保留稳定 event_id。READY/RUNNING/终态的历史开发事件不作为新任务再次发布；已有有效执行可继续完成。M1 已有的基本协调循环此时迁入独立 coordinator，确保只有已选定部署模式启动它，避免同一进程功能被装配两次。完成切换后移除 db_dev 配置和入口，只有 Kafka 接管能把新分发步骤推进到 READY。

取消后旧 Worker 可能仍在外部供应商计算，但它不能再续租或发布。供应商明确支持幂等/查询找回时才自动处理提交不确定窗口；本地 request_id 不构成外部恰好一次保证。

## 12 API 与前后端联调

### 12.1 首批接口契约

以 api/openapi.yaml 作为请求/响应真源。接口增加字段或改变错误码时同步更新 mock、前端类型和契约测试。时间字段一律带 _ms，ID 使用字符串。

| 链路 | 接口 | 前端依赖 |
|---|---|---|
| 登录 | POST /v1/auth/login、GET /v1/auth/session、POST /v1/auth/logout | Cookie 会话与写请求 CSRF token |
| 上传 | POST /v1/uploads、POST /v1/uploads/{id}/part-urls、GET /v1/uploads/{id}/parts、POST /v1/uploads/{id}/complete | 分片进度、重试和续传 |
| 视频 | GET /v1/videos、GET /v1/videos/{id}、GET /v1/videos/{id}/playback-url | 版本、可用能力和播放 |
| 内容 | GET /v1/videos/{id}/segments、GET /v1/assets/{id}/url | 字幕分页、证据截图 |
| 任务 | GET /v1/jobs/{id}、POST /v1/jobs/{id}/cancel、POST /v1/jobs/{id}/retry | 阶段、错误、显式重试 |
| Agent读取 | GET /v1/agents、GET /v1/agents/{id}/versions、GET /v1/agent-versions/{id} | 选择Agent、查看配置；返回值不含供应商密钥 |
| Agent写入 | POST /v1/agents、POST /v1/agents/{id}/versions、PATCH /v1/agent-versions/{id}、POST /v1/agent-versions/{id}/publish | 配置草稿和发布 |
| 会话 | POST /v1/sessions、GET /v1/sessions/{id}/messages、POST /v1/sessions/{id}/runs | 固定版本和历史 |
| 问答 | GET /v1/runs/{id}、GET /v1/runs/{id}/events、POST /v1/runs/{id}/cancel | 快照、SSE 与取消 |

完整接口还包括摘要、重处理、视频删除与 Agent 停用，按配套技术方案实现。M1 可先用 GET job 轮询展示媒体进度；SSE 首版用于 Agent Run，不伪装成已经提供整个上传过程的实时字节进度。

segments 与 playback-url 支持 content_version 查询参数；问答页必须传会话固定版本，服务端验证该版本属于当前视频且用户可访问。只有普通视频列表/详情允许默认读取 active 版本，不能让旧会话引用搭配新版本字幕或预览。

### 12.2 问答契约示例

提交 POST /v1/sessions/{id}/runs，携带 Cookie、CSRF header 和 Idempotency-Key，body 只包含问题；视频和 Agent 版本从会话读取。

```json
{
  "question": "这张架构图里，Kafka 和 MySQL 各自承担什么职责？"
}
```

返回 HTTP 202：

```json
{
  "run_id": "run-example",
  "job_id": "job-example",
  "status": "QUEUED",
  "events_url": "/v1/runs/run-example/events"
}
```

完成后的 GET run 示例；其中 IDs 为说明用占位符，不能作为真实业务标注：

```json
{
  "run_id": "run-example",
  "status": "SUCCEEDED",
  "content_version": "version-example",
  "latest_event_seq": 18,
  "result": {
    "answer_status": "answered",
    "claims": [
      {
        "text": "MySQL 保存任务状态，Kafka 分发任务事件。",
        "evidence_ids": ["segment:s-01", "frame:f-01"]
      }
    ],
    "evidence": [
      {"id": "segment:s-01", "kind": "transcript", "start_ms": 20000, "end_ms": 35000},
      {"id": "frame:f-01", "kind": "frame", "timestamp_ms": 24000, "asset_id": "asset-example"}
    ],
    "limitations": []
  }
}
```

模型内部输出使用技术方案中的 status；API 映射为 result.answer_status，避免与 run.status 混淆。正文从 claims 渲染，客户端不接收另一份可绕过引用规则的自由回答。

### 12.3 SSE 客户端行为

进入页面先 GET run 快照，取得 latest_event_seq，再连接 events?after_seq=N。新建 EventSource 无法任意设置 Last-Event-ID header，因此首次/刷新页面用查询参数；浏览器同一连接自动重连时使用其 Last-Event-ID，服务端优先采用有效 header 游标。两种路径统一为“读取 seq 大于游标”。

```text
id: 18
event: answer.completed
data: {"run_id":"run-example","result_url":"/v1/runs/run-example"}

```

快照中的状态、结果引用与 latest_event_seq 必须来自一致的数据库读取视图。收到终态事件后再读取完整快照并关闭连接；若初次快照已经是终态，直接显示结果，不建立持续 SSE。若终态事件恰好在快照与订阅之间产生，持久事件回放会补齐。相同 seq 丢弃；新 attempt 展示 run.retrying，不拼接旧草稿。游标过期收到 snapshot.required 时重新 GET 快照。

SSE 首版发送阶段、工具与最终结果事件；前端不要等待并不存在的 token 事件。用 HttpOnly Cookie 鉴权；写请求 CSRF token 由登录响应提供、仅保存在前端内存，刷新后由鉴权的 GET /v1/auth/session 重新获取。页面部署和开发代理保持 API 同源。

### 12.4 页面联调顺序

先做视频列表和播放器，使用本地 fixture 验证媒体元数据加载后跳转、Range 播放、签名 URL 到期刷新。再做固定问答页，用 mock 的 QUEUED/RUNNING/SUCCEEDED/FAILED/CANCELLED 状态联调，最后增加 Agent 配置、看图证据与运行详情。

点击字幕或帧证据时把毫秒转换为 currentTime 秒值，使用后端确认的时间范围；前端不解析模型给出的跳转 URL。409 按 error.code 处理：ACTIVE_RUN_CONFLICT 附带当前 run_id 供查看，IDEMPOTENCY_KEY_CONFLICT 或版本冲突展示对应说明。429 展示配额提示；503 提供有界重试。网络结果不明时重用同一 Idempotency-Key 核对，不创建新键盲目提交。

## 13 测试样本与评测数据

### 13.1 合成 fixture

在 tests/fixtures 下建立两个用户、同一视频的 v1/v2、同一版本的有效/孤立索引构建。合成内容只用于测试逻辑，不计入真实效果报告。

| fixture | 内容 | 检查目的 |
|---|---|---|
| identity_scope | 用户 A/B，同名但不同归属视频 | 词法、向量、工具、对象下载都隔离 |
| timeline_offset | 非零轨道起点、变帧率、分块重叠 | 不按帧序号推时间；重叠边界正确 |
| keyword_and_semantic | 术语精确命中与语义改写 | 双路召回和 RRF |
| visual_only | 自制流程图，字幕不包含连线关系 | 看图工具和真实 frame_id 校验 |
| no_answer | 视频未出现的主题 | 返回证据不足 |
| provider_scenarios | PENDING→SUCCEEDED、429、超时、提交不明 | 外部任务恢复与预算 |
| version_switch | v1 已有会话，v2 后发布 | 追问仍读 v1；孤立 build 不可见 |

ASR mock 暴露可配置脚本：第一次 Submit 返回 job_id；随后两次 Poll 返回 PENDING，第三次成功。另设“远端已接受但客户端收到超时”场景，验证 SUBMISSION_UNKNOWN 不自动再 Submit。

### 13.2 真实样本与标注

先用 5 个视频、30 问构建开发基线，再扩展到 15～20 个视频、约 100 问。优先自录中文技术讲解与许可允许的课程素材。数据清单记录来源、作者、许可、语言、时长和 checksum；开发集与测试集按视频/课程来源划分。

eval/questions.jsonl 每行包含 question_id、video_id、content_version、question、answerable、requires_visual、expected_key_points、acceptable_evidence_intervals、split 和 annotation_version。

评测输出按实验目录保存 config.json、per_question.jsonl、retrieval_metrics.json、answer_review.csv、usage.json 与 report.md。配置包含 Prompt、模型、工具、切片、向量、索引和代码 commit，保留失败样本，不只展示成功案例。

比较相同字幕/OCR 下的 BM25、向量、混合检索、混合加 Agent 补检索、再加视觉工具；另比较同一混合检索下“仅字幕”和“字幕+OCR”。视觉问题单独汇总，避免其提升被整体平均掩盖。

## 14 自动化测试与故障验收

### 14.1 测试层级

| 类型 | 应覆盖 | 依赖 |
|---|---|---|
| 单元 | 时间转换、RRF、预算、引用验证、状态合法性 | fake clock、fake provider、固定 fixture |
| 仓储集成 | 幂等、锁、租约、Outbox/Inbox、终态事务 | 真实 MySQL，独立测试库 |
| 服务集成 | S3 multipart、ES过滤/Bulk、Kafka接管、SSE | 测试 Compose；模型使用 mock |
| 真实连通 | ASR、工具调用、视觉、Embedding | 显式开关与小样本云请求 |
| 端到端 | 登录→上传→处理→问答→引用跳转 | 浏览器自动化与完整环境 |

测试时钟/执行屏障用于控制续租和竞争顺序，不依靠随机 sleep 碰运气。Go race detector 用于实际执行到的并发路径，不能替代数据库竞争测试。参考 [Go race detector 文档](https://go.dev/doc/articles/race_detector)。

```bash
go test ./...
go test -race ./internal/...
go vet ./...
```

上面命令约定默认测试不调用外部服务；集成测试使用独立标签或 Makefile 入口。前端 package.json 提供 typecheck、test、build；业务联调页通过后再做浏览器端到端测试。

### 14.2 故障测试必须检查的结果

| 测试编号 | 控制点 | 数据与行为断言 |
|---|---|---|
| F01 | 同一幂等请求并发提交 | 仅一个资源 ID；相同 key 不同 body 为 409 |
| F02 | 对象完成后、DB提交前退出 | 重试返回同一会话/video/job；无需重传 |
| F03 | Outbox发送后、标记前退出 | 相同 event_id 重投；同一步仅一次有效接管 |
| F04 | Inbox提交后、offset前退出 | 重投后 READY/RUNNING 不倒退 |
| F05 | Worker失联，旧进程恢复 | 新 attempt 可运行；旧 token 续租/提交均失败 |
| F06 | ASR提交后响应丢失 | 无幂等找回能力时不自动重提；状态明确可核对 |
| F07 | ES部分失败/旧构建继续写 | 发布只指向完整有效 build；查询看不到孤立数据 |
| F08 | 成功提交与取消竞争 | 按事务先后只有一个合法结果；活动会话标记清除 |
| F09 | 工具意图、结果、消息组三处分别退出 | 已完成产物复用；消息配对；证据完整；预算不清零 |
| F10 | SSE断线、重复、过期游标 | 不丢最终状态；相同seq不重复；过期走快照 |
| F11 | Redis/Kafka分别停用 | Redis不丢状态/放开配额；Kafka恢复后Outbox继续分发 |
| F12 | v2发布后在v1会话追问 | 视频、Agent、索引和证据仍使用会话固定版本 |
| F13 | ASR最终失败时OCR仍执行 | job FAILED；另一分支不能再推进索引；租约和会话标记清理 |
| F14 | 接近用户活动任务上限时并发提交 | 配额锁使实际活动数不超过配置上限 |

仓储测试直接构造过期租约；进程级测试使用可开关 failpoint 或测试屏障，在明确位置终止 Worker。故障开关仅用于测试构建/环境，不能在普通 API 暴露“让进程退出”接口。

## 15 M5 交付与日常提交标准

| 编号 | 工作 | 验收 |
|---|---|---|
| M5-01 | 数据来源与人工标注、冻结集、消融 | 指标和逐题原始记录一致，失败样本可追查 |
| M5-02 | 后端负载、模型延迟与费用报告 | 区分mock和真实API；注明机器、样本与配额 |
| M5-03 | Compose、迁移、监控、备份恢复 | 空库启动可重现；做一次数据恢复演练 |
| M5-04 | README、API、运行手册与演示 | 一条上传→分析→问答→引用→恢复的演示路径 |

每项开发任务提交时附任务编号、变更说明、验证命令、样本/配置和仍未解决的问题。数据库、接口和前端类型一起变更；不把“能编译”当作涉及状态机任务的唯一验收。

持续集成依次执行格式检查、静态检查、单元测试、前端类型/构建；涉及数据库或消息的修改再执行对应集成测试。云模型测试单独触发，普通提交不能自动消耗真实模型额度。

发布前从干净数据库运行迁移和 seed，通过上传、处理、问答及取消闭环；检查依赖版本固定、密钥未入库、旧版本引用仍可读。README 展示实际可用能力和测量结果，不提前填入目标性能。

## 16 联调排障表

| 现象 | 优先检查 | 处理方向 |
|---|---|---|
| 浏览器上传403，后端可上传 | S3签名host、CORS、时钟、请求头 | 修正签名与浏览器实际请求，不换host重用签名 |
| 云ASR下载失败 | COS对象、签名有效期、外部可达性 | 使用云暂存；明确失败后才决定重提 |
| 字幕和画面时间偏移 | 原轨道起点、音频块实际offset、预览映射 | 用带已知时间标记的样本逐段定位 |
| Kafka无积压但任务不执行 | step是否READY、worker_kind、租约、执行池 | 区分接管与执行；检查DB扫描和容量 |
| Outbox反复发布 | broker确认后DB更新、发布租约 | 验证event_id稳定；Inbox去重应保持有效 |
| ES查不到已写内容 | build_id过滤、Bulk逐项错误、refresh | 确认可搜后发布；不要去掉权限过滤“修复” |
| Agent循环不断 | 重复参数、工具预算、模型迭代、证据不足分支 | 程序停止循环，返回不足或受限结果 |
| 重连后看不到答案 | 快照游标、事件保留、代理缓冲、Cookie | 从持久快照恢复，不重新启动问答 |
| 取消后仍有费用 | 外部请求是否已提交、供应商取消能力 | 保留调用记录；平台结果不再发布 |

排障使用 request_id、job_id、step_id、attempt_no、run_id、event_id 关联日志。日志不能包含完整 Cookie、签名 URL 查询参数、密钥或不必要的用户全文。

## 17 第一个开发迭代

按以下顺序完成第一轮，不同时铺开所有中间件与页面：

- [ ] 建立仓库目录、配置加载、doctor 和版本清单。
- [ ] 创建测试/开发隔离环境，完成 M0 的供应商短样本验证。
- [ ] 完成 0001～0003 迁移及基础仓储事务。
- [ ] 完成登录与资源归属校验。
- [ ] 完成分片上传、续传、Complete 对账。
- [ ] 完成 PROBE 与音频提取，任务状态可查询。
- [ ] 接入异步 ASR，验证保存 job_id 后重启仍查询原任务。
- [ ] 执行 0004 迁移，持久化句级字幕与帧信息。
- [ ] 用播放器核验转写时间，再继续抽帧/OCR与检索。

每项完成后更新任务记录中的实际状态与验证证据。开发中若要替换中间件、模型或版本语义，先记录原因与影响，再修改配套技术方案和本手册，避免实现与文档分叉。

## 18 参考实现的使用方式

[DOVideo-AI](https://github.com/Xiaoc7r/DOVideo-AI) 用于参考视频预处理、统一内容片段与问答业务组织。Go 模块按本手册接口独立实现；Java 框架迁移、RocketMQ→Kafka 和向量库→ES 分别验证，不能照搬确认消息和恢复逻辑。

需要借用代码时先固定上游 commit、核验该版本许可证并保留要求的声明。对照源码研究记录放 docs/reference-notes.md，说明借鉴模块与本项目修改；最终 README 保持与实际实现一致。
