# GoWind Admin 后端组件使用文档

本文档梳理后端服务使用的核心组件，包括在本项目中的实际配置方式、代码位置和官方文档链接。

---

## 组件总览

| 组件 | 版本 | 用途 | 官方文档 |
|------|------|------|---------|
| go-kratos | v2.9.2 | 微服务 HTTP 框架 | [go-kratos.dev](https://go-kratos.dev/docs/) |
| ENT | v0.14.6 | ORM（Schema as Code） | [entgo.io](https://entgo.io/) |
| Google Wire | v0.7.0 | 编译期依赖注入 | [github.com/google/wire](https://github.com/google/wire) |
| go-redis | v9.18.0 | Redis 客户端 | [redis.uptrace.dev](https://redis.uptrace.dev/) |
| Asynq | v0.26.0 | 分布式任务队列（Redis 后端） | [github.com/hibiken/asynq](https://github.com/hibiken/asynq/wiki) |
| MinIO Go SDK | v7.0.99 | S3 兼容对象存储 | [minio-go.min.io](https://minio-go.min.io/) |
| golang-jwt | v5.3.1 | JWT 令牌签发与验证 | [github.com/golang-jwt/jwt](https://github.com/golang-jwt/jwt) |
| OPA | v1.12.1 | 基于策略的授权引擎 | [openpolicyagent.org](https://www.openpolicyagent.org/docs/) |
| Casbin | v2.135.0 | RBAC/ABAC 授权（可选） | [casbin.org](https://casbin.org/docs/overview) |
| gopher-lua | v1.1.1 | Lua 脚本扩展引擎 | [github.com/yuin/gopher-lua](https://github.com/yuin/gopher-lua) |
| kratos-bootstrap | v0.1.16 | 应用启动引导框架 | [github.com/tx7do/kratos-bootstrap](https://github.com/tx7do/kratos-bootstrap) |
| kratos-authn | v1.1.9 | 认证中间件 | [github.com/tx7do/kratos-authn](https://github.com/tx7do/kratos-authn) |
| kratos-authz | v1.1.7 | 授权中间件 | [github.com/tx7do/kratos-authz](https://github.com/tx7do/kratos-authz) |
| go-crud | v0.0.50 | 通用 CRUD 仓储封装 | [github.com/tx7do/go-crud](https://github.com/tx7do/go-crud) |
| pgx | v5.8.0 | PostgreSQL 驱动 | [github.com/jackc/pgx](https://github.com/jackc/pgx) |
| SSE Transport | v1.3.2 | 服务端推送事件 | [github.com/tx7do/kratos-transport](https://github.com/tx7do/kratos-transport) |

---

## 1. go-kratos — 微服务 HTTP 框架

**官方文档**：https://go-kratos.dev/docs/

### 项目中的使用方式

Kratos 是整个后端的骨架框架，负责 HTTP 服务、中间件链、配置加载和服务生命周期管理。

**服务启动入口**：`app/admin/service/cmd/server/main.go`

```go
func newApp(ctx *bootstrap.Context, hs *http.Server, as *asynq.Server, ss *sse.Server) *kratos.App {
    return bootstrap.NewApp(ctx, hs, as, ss) // 注册 HTTP + Asynq + SSE 三个 Transport
}
```

**HTTP Server 配置**：`app/admin/service/internal/server/rest_server.go`

中间件注册顺序：
1. 标准日志中间件（Kratos 内置）
2. API 审计日志中间件（自定义）
3. JWT 认证中间件（kratos-authn，带白名单跳过）
4. OPA/Casbin 授权中间件（kratos-authz）
5. 应用日志中间件（自定义）

**配置文件**：`app/admin/service/configs/server.yaml`

```yaml
server:
  rest:
    addr: ":7788"
    timeout: 10s
    enable_swagger: true
    cors:
      origins: ["*"]
    middleware:
      enable_logging: true
      enable_recovery: true
      enable_tracing: true
      enable_validate: true
      enable_circuit_breaker: true
```

### 关键概念

- **Transport**：Kratos 抽象的传输层，本项目使用了 HTTP、Asynq、SSE 三种
- **Middleware**：链式中间件，通过 `middleware.Chain()` 组合
- **Selector**：选择性应用中间件，支持白名单路径跳过认证

---

## 2. ENT — ORM 框架

**官方文档**：https://entgo.io/

### 项目中的使用方式

ENT 用于定义数据库 Schema、自动建表、生成类型安全的查询代码。

**Schema 定义目录**：`app/admin/service/internal/data/ent/schema/`

示例 Schema（`user.go`）：

```go
type User struct {
    ent.Schema
}

func (User) Mixin() []ent.Mixin {
    return []ent.Mixin{
        mixin.AutoIncrementId{},   // 自增主键
        mixin.TimeAt{},            // created_at / updated_at
        mixin.OperatorID{},        // 操作人 ID
        mixin.SwitchStatus{},      // 启用/禁用状态
        mixin.TenantID{},          // 租户 ID（多租户）
    }
}

func (User) Fields() []ent.Field {
    return []ent.Field{
        field.String("username").Unique(),
        field.String("nickname").Optional(),
        // ...
    }
}
```

**ENT 客户端初始化**：`app/admin/service/internal/data/ent_client.go`

```go
func NewEntClient(ctx *bootstrap.Context) *entCrud.EntClient[*ent.Client] {
    cfg := ctx.GetConfig()
    client := entBootstrap.NewEntClient(cfg, driverFunc)
    if cfg.Data.Database.GetMigrate() {
        // 自动执行数据库迁移
    }
    return client
}
```

**代码生成命令**：

```bash
# 在 backend/ 下执行
make ent

# 或手动执行
ent generate \
    --feature privacy \
    --feature entql \
    --feature sql/modifier \
    --feature sql/upsert \
    --feature sql/lock \
    ./internal/data/ent/schema
```

启用的 Feature Flags：
- `privacy`：行级隐私策略
- `entql`：类型安全的过滤表达式
- `sql/modifier`：自定义 SQL 修饰符
- `sql/upsert`：UPSERT 支持
- `sql/lock`：行锁（SELECT FOR UPDATE）

**配置文件**：`app/admin/service/configs/data.yaml`

```yaml
data:
  database:
    driver: "postgres"               # 也支持 "mysql"
    source: "host=localhost port=5432 user=postgres password=*Abcd123456 dbname=gwa sslmode=disable"
    migrate: true                    # 自动建表
    max_idle_connections: 25
    max_open_connections: 25
    connection_max_lifetime: 300s
```

---

## 3. Google Wire — 依赖注入

**官方文档**：https://github.com/google/wire/blob/main/docs/guide.md

### 项目中的使用方式

Wire 在编译期生成依赖注入代码，替代手写的初始化链。

**Injector 定义**：`app/admin/service/cmd/server/wire.go`

```go
//go:build wireinject

func initApp(ctx *bootstrap.Context) (*kratos.App, func(), error) {
    wire.Build(
        dataProviders.ProviderSet,      // 40+ 仓储 + 基础设施
        serverProviders.ProviderSet,    // HTTP / Asynq / SSE Server
        serviceProviders.ProviderSet,   // 业务 Service 层
        newApp,
    )
    return nil, nil, nil
}
```

**三个 ProviderSet**：

| ProviderSet | 位置 | 提供内容 |
|-------------|------|---------|
| `dataProviders` | `internal/data/providers/wire_set.go` | Redis 客户端、ENT 客户端、MinIO 客户端、所有 Repo、认证器、授权器 |
| `serverProviders` | `internal/server/providers/wire_set.go` | REST Server、Asynq Server、SSE Server |
| `serviceProviders` | `internal/service/providers/wire_set.go` | 所有业务 Service（用户、角色、权限、租户等） |

**代码生成命令**：

```bash
make wire
# 或
go run -mod=mod github.com/google/wire/cmd/wire ./cmd/server
```

生成文件 `wire_gen.go`，包含完整的依赖构建链。

---

## 4. go-redis — Redis 客户端

**官方文档**：https://redis.uptrace.dev/

### 项目中的使用方式

Redis 用于缓存（用户 Token、字典数据）和作为 Asynq 任务队列的 Broker。

**初始化**：`app/admin/service/internal/data/data.go`

```go
func NewRedisClient(ctx *bootstrap.Context) (*redis.Client, func(), error) {
    cfg := ctx.GetConfig()
    cli := redisClient.NewClient(cfg.Data, l)
    return cli, func() { cli.Close() }, nil
}
```

**Token 缓存**：`app/admin/service/internal/data/user_token_cache.go`

用户登录后的 Token 存入 Redis，用于：
- Token 有效性校验（防止注销后继续使用）
- 多设备登录管理

**配置文件**：`app/admin/service/configs/data.yaml`

```yaml
data:
  redis:
    addr: "localhost:6379"
    password: "*Abcd123456"
    dial_timeout: 10s
    read_timeout: 0.4s
    write_timeout: 0.6s
```

---

## 5. Asynq — 分布式任务队列

**官方文档**：https://github.com/hibiken/asynq/wiki/Getting-Started

### 项目中的使用方式

Asynq 以 Redis 为后端，实现异步任务和定时任务调度。

**Server 初始化**：`app/admin/service/internal/server/asynq_server.go`

```go
func NewAsynqServer(ctx *bootstrap.Context, taskService *service.TaskService) (*asynqServer.Server, error) {
    srv := bootstrapAsynq.NewAsynqServer(cfg.Server.Asynq)

    // 注册任务处理器
    asynqServer.RegisterSubscriber(srv, task.BackupTaskType, taskService.AsyncBackup)

    // 从数据库加载并启动所有定时任务
    taskService.StartAllTask(appViewer.NewSystemViewerContext(ctx.Context()), &emptypb.Empty{})
    return srv, nil
}
```

**任务管理**：`app/admin/service/internal/service/task_service.go`

- `RegisterTaskScheduler()`：连接 Asynq Scheduler
- `NewTask()`：创建一次性异步任务
- `NewPeriodicTask()`：创建周期定时任务（cron 表达式）
- `RemovePeriodicTask()`：移除定时任务
- `StartAllTask()`：从数据库加载所有已启用的定时任务

**配置文件**：`app/admin/service/configs/server.yaml`

```yaml
server:
  asynq:
    uri: "redis://:*Abcd123456@localhost:6379/1"   # 使用 Redis DB 1
    codec: "json"
    concurrency: 10                                 # 并发 Worker 数
    shutdown_timeout: 10s
    queues:
      critical: 10                                  # 优先级权重
      default: 5
      low: 1
    enable_strict_priority: true                    # 严格优先级模式
```

---

## 6. MinIO Go SDK — 对象存储

**官方文档**：https://minio-go.min.io/

### 项目中的使用方式

MinIO 用于文件上传/下载，兼容 S3 协议。

**封装层**：`pkg/oss/minio.go`

```go
type MinIOClient struct {
    mc         *minio.Client
    conf       *conf.OSS
    hmacSecret []byte
}
```

**核心方法**：

| 方法 | 用途 |
|------|------|
| `EnsureBucketExists(bucket)` | 幂等创建 Bucket |
| `GetUploadPresignedUrl(bucket, object, expires)` | 生成预签名上传 URL |
| `BucketExists(bucket)` | 检查 Bucket 是否存在 |
| `MakeBucket(bucket)` | 创建 Bucket |

**Bucket 按内容类型分类**：images、documents、files 等。

**配置文件**：`app/admin/service/configs/oss.yaml`

```yaml
oss:
  minio:
    endpoint: "localhost:9000"
    upload_host: "localhost:9000"
    download_host: "localhost:9000"
    access_key: "root"
    secret_key: "*Abcd123456"
    use_ssl: false
```

---

## 7. golang-jwt — JWT 令牌

**官方文档**：https://github.com/golang-jwt/jwt

### 项目中的使用方式

JWT 用于用户身份认证，签发 Access Token 和 Refresh Token。

**Token Claims 结构**：`pkg/jwt/user_token_payload.go`

```go
const (
    ClaimFieldUserName  = "sub"    // 用户名
    ClaimFieldUserID    = "uid"    // 用户 ID
    ClaimFieldTenantID  = "tid"    // 租户 ID
    ClaimFieldRoleCodes = "roc"    // 角色编码数组
    ClaimFieldDataScope = "ds"     // 数据权限范围
    ClaimFieldOrgUnitID = "ouid"   // 组织单元 ID
)
```

**Token 有效期**：`app/admin/service/internal/data/authenticator.go`

| 类型 | 默认有效期 |
|------|-----------|
| Access Token | 15 分钟 |
| Refresh Token | 7 天 |
| 时间容差 (Leeway) | 60 秒 |

**认证流程**：

1. 用户登录 -> 签发 Access Token + Refresh Token
2. 请求携带 `Authorization: Bearer <token>`
3. 认证中间件解析 Token，校验签名和有效期
4. 从 Redis 检查 Token 是否已注销
5. 将 Claims 注入请求上下文

**配置文件**：`app/admin/service/configs/auth.yaml`

```yaml
authn:
  type: "jwt"
  jwt:
    method: "HS256"          # 支持 HS256/RS256/ES256/Ed25519 等
    key: "some_api_key"      # 签名密钥
```

---

## 8. OPA / Casbin — 授权引擎

### OPA（Open Policy Agent）

**官方文档**：https://www.openpolicyagent.org/docs/

项目默认使用 OPA 作为授权引擎。

**策略加载**：`pkg/authorizer/authorizer.go`

```go
// 从嵌入的 rego 文件加载 RBAC 策略模型
msg=load custom OPA model: rbac.rego

// 从数据库加载角色-API 映射关系
// 每个角色对应一组可访问的 API（pattern + method）
type OpaPolicyPath struct {
    Pattern string `json:"pattern"`
    Method  string `json:"method"`
}
```

**授权判断流程**：

1. 从 JWT Claims 中提取用户角色
2. 根据请求的 Path + Method 查询 OPA 策略
3. OPA Rego 引擎判定是否允许访问

### Casbin（可选替代）

**官方文档**：https://casbin.org/docs/overview

```yaml
authz:
  type: "opa"      # 切换为 "casbin" 可使用 Casbin 引擎
```

Casbin 策略格式：`[p, roleCode, path, method, domain]`

**配置文件**：`app/admin/service/configs/auth.yaml`

```yaml
authz:
  type: "opa"              # 可选: opa, casbin, noop
```

---

## 9. gopher-lua — Lua 脚本引擎

**官方文档**：https://github.com/yuin/gopher-lua

### 项目中的使用方式

通过 Lua 脚本实现业务逻辑的灵活扩展，无需重新编译 Go 代码。

**Lua API 模块目录**：`pkg/lua/api/`

| 模块文件 | Lua 模块名 | 功能 |
|----------|-----------|------|
| `cache.go` | cache | Redis 缓存操作（Get/Set/Delete） |
| `crypto.go` | crypto | 加密/解密操作 |
| `eventbus.go` | eventbus | 事件发布 |
| `hook.go` | hook | Hook 系统 |
| `logger.go` | logger | 日志输出 |
| `oss.go` | oss | 对象存储操作 |

**Lua 中使用示例**：

```lua
local cache = require("cache")
local val = cache.get("user:1001")

local logger = require("logger")
logger.info("User fetched: " .. val)
```

**Go 端注册示例**：`pkg/lua/api/cache.go`

```go
func RegisterCache(L *lua.LState, rdb *redis.Client) {
    cacheModule.RawSetString("get", L.NewFunction(func(L *lua.LState) int {
        key := L.CheckString(1)
        val, err := rdb.Get(context.Background(), key).Result()
        L.Push(convert.ToLuaValue(L, val))
        return 1
    }))
}
```

---

## 10. SSE — 服务端推送事件

**官方文档**：https://github.com/tx7do/kratos-transport

### 项目中的使用方式

SSE 用于后端向前端实时推送事件（如通知、消息等）。

**Server 初始化**：`app/admin/service/internal/server/sse_server.go`

```go
func NewSseServer(ctx *bootstrap.Context) *sseServer.Server {
    srv := sse.NewSseServer(cfg.Server.Sse,
        sseServer.WithSubscriberFunction(func(streamID sseServer.StreamID, sub *sseServer.Subscriber) {
            l.Infof("subscriber [%s] connected", streamID)
        }),
    )
    return srv
}
```

**配置文件**：`app/admin/service/configs/server.yaml`

```yaml
server:
  sse:
    addr: ":7789"
    codec: "json"
    path: "/events"
    auto_stream: true
    auto_reply: false
```

前端通过 `EventSource` 连接 `http://localhost:7789/events` 接收实时事件。

---

## 11. go-crud — 通用 CRUD 仓储封装

**官方文档**：https://github.com/tx7do/go-crud

### 项目中的使用方式

go-crud 提供了泛型化的仓储基类，减少 CRUD 样板代码。

**仓储定义示例**：`app/admin/service/internal/data/user_repo.go`

```go
type UserRepo struct {
    entClient *entCrud.EntClient[*ent.Client]
    repository *entCrud.Repository[
        ent.UserQuery, ent.UserSelect,
        ent.UserCreate, ent.UserCreateBulk,
        ent.UserUpdate, ent.UserUpdateOne,
        ent.UserDelete,
        predicate.User,
        identityV1.User, ent.User,      // Proto 类型 ↔ ENT 类型
    ]
    mapper *mapper.CopierMapper[identityV1.User, ent.User]
}
```

**使用到的子包**：

| 子包 | 用途 |
|------|------|
| `go-crud/entgo` | ENT ORM 的泛型 Repository 封装 |
| `go-crud/pagination` | 分页查询工具 |
| `go-crud/viewer` | 数据访问上下文（租户隔离、数据权限） |
| `go-crud/api` | 通用 CRUD API Proto 定义 |

---

## 12. kratos-bootstrap — 应用启动引导

**官方文档**：https://github.com/tx7do/kratos-bootstrap

### 项目中的使用方式

kratos-bootstrap 封装了 Kratos 的启动流程，统一管理配置加载、日志、注册中心、数据库连接等。

**启动流程**：`app/admin/service/cmd/server/main.go`

```go
func runApp() error {
    ctx := bootstrap.NewContext(context.Background(), &conf.AppInfo{
        Project: serviceid.ProjectName,
        AppId:   serviceid.AdminService,
        Version: version,
    })
    return bootstrap.RunApp(ctx, initApp)   // initApp 由 Wire 生成
}
```

**Bootstrap 自动加载的配置文件**（`configs/` 目录下）：

| 文件 | 内容 |
|------|------|
| `server.yaml` | HTTP/SSE/Asynq 服务配置 |
| `data.yaml` | 数据库 + Redis 连接配置 |
| `auth.yaml` | 认证（JWT）+ 授权（OPA/Casbin）配置 |
| `oss.yaml` | MinIO 对象存储配置 |
| `logger.yaml` | 日志级别和输出方式 |
| `client.yaml` | gRPC 客户端配置 |

---

## 请求处理全链路

```
HTTP Request
  │
  ├─ Kratos HTTP Server (:7788)
  │   ├─ Recovery 中间件
  │   ├─ Logging 中间件
  │   ├─ API 审计日志中间件
  │   ├─ JWT 认证中间件（kratos-authn）
  │   │   └─ Token 验证 → Redis 校验 → Claims 注入上下文
  │   ├─ OPA 授权中间件（kratos-authz）
  │   │   └─ 角色 + Path + Method → OPA Rego 策略判定
  │   └─ 路由到 Service Handler
  │
  ├─ Service 层（业务逻辑）
  │   ├─ 调用注入的 Repository
  │   ├─ 可选：发布异步任务到 Asynq
  │   └─ 可选：推送 SSE 事件
  │
  ├─ Data 层（Repository）
  │   ├─ go-crud 泛型仓储
  │   ├─ ENT ORM 查询构建
  │   ├─ Mapper: Proto ↔ ENT 类型转换
  │   └─ PostgreSQL (pgx) / MySQL 驱动
  │
  └─ Response
```
