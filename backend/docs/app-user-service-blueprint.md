# `app/user/service` 新增服务落地蓝图

## 1. 目标

在保持当前仓库工程规范不变的前提下，新增一个仅面向用户端的服务应用 `app/user/service`。

本蓝图覆盖以下内容：

- 目录规划
- proto 规划
- wire 组装
- server 注册
- 认证与鉴权改造点
- `trader` 能力的落地方式

本蓝图的前提结论是：

- 保留当前大仓结构
- 不直接切回标准 `kratos-layout`
- 按当前项目的 `buf + wire + ent + kratos-bootstrap` 体系新增服务

---

## 2. 先说结论

推荐方案：

1. 新增独立应用目录 `app/user/service`
2. 继续复用当前仓库的工程约定，而不是单独引入一套新的 Kratos 默认目录结构
3. 用户端协议新增一层包装 proto：`api/protos/user/service/v1/*`
4. 领域模型尽量复用已有 `identity/authentication` proto
5. `trader` 先按业务复杂度分两档处理

`trader` 分两档：

- 如果 `trader` 只是“用户的一种业务身份/角色”，先放进 `app/user/service`，不新增独立 app
- 如果 `trader` 有独立资料、独立生命周期、独立表结构，再新增 `trader` 领域 proto 和 repo

---

## 3. 当前仓库的关键约束

新增 `app/user/service` 时，必须先认清下面几个约束：

### 3.1 当前服务不是标准 `kratos-layout`

当前项目虽然基于 Kratos，但工程体系已经深度定制：

- 启动走 `tx7do/kratos-bootstrap`
- 依赖注入走 `wire`
- 协议生成走 `buf`
- 数据访问走 `ent` 和 `go-crud`
- OpenAPI 模板是按 `admin` 单独定制的

因此不能把 `kratos new` 生成的目录直接当最终结构。

### 3.2 `internal` 包不能跨服务复用

这是最重要的实际限制。

当前很多核心实现都放在：

- `app/admin/service/internal/data`
- `app/admin/service/internal/data/ent`
- `app/admin/service/internal/service`

由于 Go 的 `internal` 规则，`app/user/service` 不能直接 import `app/admin/service/internal/...`。

这意味着：

- 不能直接复用 `admin` 下的 repo
- 不能直接复用 `admin` 下的 ent client
- 不能直接复用 `admin` 下的 service 实现

因此新增 `app/user/service` 时，必须二选一：

1. 复制一份 `admin` 的最小数据层到 `user`
2. 先做一轮抽取，把公共数据访问能力迁到 `pkg` 或共享包

为了控制风险，建议分两阶段做：

- 第一阶段：复制最小可用实现，优先跑通
- 第二阶段：再抽共享层

### 3.3 当前 `ClientType_app` 只在 proto 层存在，代码层未完整落地

当前仓库已经在协议里定义了：

- `authentication.service.v1.ClientType.admin`
- `authentication.service.v1.ClientType.app`

但实现并未完整支持 `app`：

- `app/admin/service/internal/data/data.go` 里 `NewClientType()` 固定返回 `admin`
- `app/admin/service/internal/data/authenticator.go` 里只初始化了 `AdminAuthenticator`
- `app/admin/service/internal/service/authentication_service.go` 的 `RefreshToken` 还写死 `ClientType_admin`

所以用户端服务的认证层不能直接照搬后台实现。

---

## 4. 推荐目录蓝图

建议新增如下目录：

```text
backend/
├─ api/
│  ├─ protos/
│  │  ├─ user/
│  │  │  └─ service/
│  │  │     └─ v1/
│  │  │        ├─ i_authentication.proto
│  │  │        ├─ i_user_profile.proto
│  │  │        └─ i_trader.proto
│  │  └─ trader/
│  │     └─ service/
│  │        └─ v1/
│  │           └─ trader.proto
│  ├─ gen/
│  ├─ buf.gen.yaml
│  └─ buf.user.openapi.gen.yaml
├─ app/
│  ├─ admin/
│  │  └─ service/
│  └─ user/
│     └─ service/
│        ├─ Makefile
│        ├─ README.md
│        ├─ cmd/
│        │  └─ server/
│        │     ├─ main.go
│        │     ├─ wire.go
│        │     └─ assets/
│        ├─ configs/
│        │  ├─ auth.yaml
│        │  ├─ client.yaml
│        │  ├─ data.yaml
│        │  ├─ logger.yaml
│        │  ├─ oss.yaml
│        │  └─ server.yaml
│        └─ internal/
│           ├─ data/
│           │  ├─ data.go
│           │  ├─ authenticator.go
│           │  ├─ token_checker.go
│           │  ├─ providers/
│           │  │  └─ wire_set.go
│           │  ├─ ent/
│           │  │  └─ schema/
│           │  ├─ user_repo.go
│           │  ├─ user_credential_repo.go
│           │  ├─ tenant_repo.go
│           │  ├─ membership_repo.go
│           │  ├─ role_repo.go
│           │  ├─ org_unit_repo.go
│           │  ├─ position_repo.go
│           │  └─ trader_repo.go
│           ├─ service/
│           │  ├─ authentication_service.go
│           │  ├─ user_profile_service.go
│           │  ├─ trader_service.go
│           │  └─ providers/
│           │     └─ wire_set.go
│           └─ server/
│              ├─ rest_server.go
│              ├─ asynq_server.go
│              ├─ sse_server.go
│              └─ providers/
│                 └─ wire_set.go
└─ pkg/
```

---

## 5. 目录创建策略

### 5.1 快速落地策略

最快的做法不是从零写，而是：

1. 复制 `app/admin/service` 为 `app/user/service`
2. 保留启动骨架、configs、wire、server provider
3. 删除与用户端无关的 service 和 repo
4. 逐步收缩成用户端需要的最小集合

这样做的原因：

- 当前 `admin` 服务已经完整打通了 `bootstrap/wire/ent/buf/openapi`
- 新服务最难的是把这些链路重新接起来
- 先复制可运行骨架，风险最小

### 5.2 不建议的做法

不建议直接用 `kratos new app/user --nomod` 作为最终落地目录，原因：

- 生成的目录仍然需要大幅适配当前仓库
- 生成的 Makefile、proto 目录、server 组装方式，与现有仓库不一致
- 最终你还是得把它改造成当前项目的风格

可以把 `kratos` CLI 当草稿生成器，但不要把生成结果直接并入仓库主线。

---

## 6. proto 蓝图

### 6.1 继续使用“双层 proto”结构

延续当前 admin 的做法：

- 领域层：`identity/service/v1`、`authentication/service/v1`、`trader/service/v1`
- 应用包装层：`user/service/v1`

也就是说：

- `UserProfile`、`Authentication` 尽量继续复用 `identity/authentication`
- `app/user/service` 对外暴露的 HTTP 路由放在 `user/service/v1` 包装层

### 6.2 用户端包装 proto 建议

建议新增：

- `api/protos/user/service/v1/i_authentication.proto`
- `api/protos/user/service/v1/i_user_profile.proto`
- `api/protos/user/service/v1/i_trader.proto`

包名建议：

```proto
package user.service.v1;
```

HTTP 路由前缀建议：

- `/user/v1/login`
- `/user/v1/logout`
- `/user/v1/refresh-token`
- `/user/v1/register`
- `/user/v1/me`
- `/user/v1/traders`

说明：

- 路由前缀建议用 `user`
- token 的 `client_type` 仍然使用 `app`
- 不建议把 HTTP 路由写成 `/app/v1/*`，因为 `app` 更像客户端类型，不是明确业务域

### 6.3 `trader` 的 proto 设计

#### 方案 A：`trader` 只是用户端业务接口

如果 `trader` 只是用户端对某些用户数据的视图，不新增领域 proto，只新增：

- `api/protos/user/service/v1/i_trader.proto`

service 直接返回：

- `identity.service.v1.User`
- 或一个 `user.service.v1.TraderView`

#### 方案 B：`trader` 是新领域

如果 `trader` 是独立业务实体，新增：

- `api/protos/trader/service/v1/trader.proto`
- `api/protos/user/service/v1/i_trader.proto`

其中：

- `trader.proto` 定义领域模型和 RPC
- `i_trader.proto` 只负责用户端 HTTP 暴露

### 6.4 `buf` 配置要改的点

#### `api/buf.gen.yaml`

追加新的 `go_package` override：

```yaml
- file_option: go_package
  path: user/service/v1
  value: go-wind-admin/api/gen/go/user/service/v1;userpb

- file_option: go_package
  path: trader/service/v1
  value: go-wind-admin/api/gen/go/trader/service/v1;traderpb
```

#### 新增 `api/buf.user.openapi.gen.yaml`

当前仓库只有 admin 的 OpenAPI 模板，输入路径固定是 `protos/admin/service/v1`。

因此必须新增一个用户端模板，例如：

```yaml
inputs:
  - directory: protos
    paths:
      - protos/user/service/v1
```

输出目录改为：

```yaml
out: ../app/user/service/cmd/server/assets
```

---

## 7. `app/user/service` 的骨架蓝图

### 7.1 `cmd/server/main.go`

做法：

- 直接复制 `app/admin/service/cmd/server/main.go`
- 把 `AppId` 改成新的 `serviceid.UserService`

需要先在 `pkg/serviceid/service_id.go` 增加：

```go
const (
    AdminService = "admin-service"
    UserService  = "user-service"
)
```

### 7.2 `cmd/server/wire.go`

做法：

- 直接复制 `app/admin/service/cmd/server/wire.go`
- 把 import 路径全部替换成 `app/user/service/...`

### 7.3 `configs/`

做法：

- 先整体复制 `app/admin/service/configs`
- 再修改端口和服务名

建议至少改：

- `server.yaml`：REST/SSE 端口不要与 admin 冲突
- `logger.yaml`：模块名改成 user-service
- `client.yaml`：如果依赖其他服务，按用户端调用链调整

---

## 8. data 层蓝图

### 8.1 第一阶段建议：复制最小可用 data 层

由于 `internal` 包限制，推荐第一阶段直接从 `admin` 复制最小可用集合到 `app/user/service/internal/data`。

用户端最常见需要的 repo：

- `user_repo.go`
- `user_credential_repo.go`
- `tenant_repo.go`
- `membership_repo.go`
- `role_repo.go`
- `org_unit_repo.go`
- `position_repo.go`

如果 `trader` 是独立领域，再加：

- `trader_repo.go`

### 8.2 ent schema 的处理建议

这里有两个可执行选项。

#### 选项 A：直接复制 `admin` 的 schema 子集

复制到：

- `app/user/service/internal/data/ent/schema`

至少需要：

- `user.go`
- `user_credential.go`
- `tenant.go`
- `role.go`
- `permission.go`
- `org_unit.go`
- `position.go`
- `membership.go`
- `membership_role.go`
- `membership_org_unit.go`
- `membership_position.go`
- `user_role.go`
- `user_org_unit.go`
- `user_position.go`

如果 `trader` 独立建表，再加：

- `trader.go`

优点：

- 快速
- 不需要改现有工程边界

缺点：

- 会出现 schema/repo 重复

#### 选项 B：抽共享数据访问层

把通用 schema/repo 抽到非 `internal` 共享包，例如：

- `pkg/store/identity`
- `pkg/store/authentication`

优点：

- 长期整洁

缺点：

- 初次改动面大
- 会影响 admin 现有代码

建议：

- 当前先选 A
- 等 `user` 跑通后再考虑抽共享层

### 8.3 `data.go`

复制 admin 版后，关键修改是：

```go
func NewClientType() authenticationV1.ClientType {
    return authenticationV1.ClientType_app
}
```

这会影响：

- `TokenChecker`
- `AuthenticationService`
- token 缓存键空间

### 8.4 `providers/wire_set.go`

做法：

- 复制 admin 的 `internal/data/providers/wire_set.go`
- 删除用户端不需要的 repo/provider
- 保留认证链路相关 provider

用户端最小可用集合通常至少包括：

- `NewRedisClient`
- `NewEntClient`
- `NewClientType`
- `NewAuthenticator`
- `NewTokenChecker`
- `NewPasswordCrypto`
- `NewUserTokenCache`
- `NewUserRepo`
- `NewUserCredentialRepo`
- `NewTenantRepo`
- `NewMembershipRepo`
- `NewRoleRepo`
- `NewOrgUnitRepo`
- `NewPositionRepo`

如果要做 route authz，再保留：

- `NewAuthorizerProvider`
- `authorizer.NewAuthorizer`

---

## 9. service 层蓝图

### 9.1 第一批建议先落的 service

先只做这几个：

- `AuthenticationService`
- `UserProfileService`
- `TraderService`

不建议一开始把 admin 的全部 service 都复制过来。

### 9.2 `AuthenticationService` 的处理原则

不能直接照抄 admin 版。

原因：

- admin 的登录逻辑是“后台访问授权”
- 当前 admin 的授权逻辑会校验后台权限码
- 用户端不应该依赖 `SystemAccessBackendPermissionCode`

因此用户端 `AuthenticationService` 应按下面方式改：

#### 登录

- 继续复用 `user_credential_repo.VerifyCredential`
- 继续复用 `user_repo.Get`
- 继续生成 `ClientType_app` 的 token

#### 授权判定

把“能否登录”的判断从“是否有后台权限”改成“是否允许使用用户端”。

可选策略：

1. 只校验用户状态为 `NORMAL`
2. 如果启用租户，校验租户状态正常
3. 如果需要 trader 权限，再基于角色或 trader 状态判断

#### 注册

保留对外开放：

- `RegisterUser`

#### 刷新 token

不能再写死：

```go
req.ClientType = trans.Ptr(authenticationV1.ClientType_admin)
```

必须改成：

```go
req.ClientType = trans.Ptr(s.clientType)
```

### 9.3 `UserProfileService`

这个 service 可以高度复用 admin 的思路。

建议保留接口：

- `GetUser`
- `UpdateUser`
- `ChangePassword`
- `UploadAvatar`
- `DeleteAvatar`

用户端路由建议：

- `GET /user/v1/me`
- `PUT /user/v1/me`
- `POST /user/v1/me/password`
- `POST /user/v1/me/avatar`
- `DELETE /user/v1/me/avatar`

### 9.4 `TraderService`

建议第一阶段只开放最小接口：

- `List`
- `Get`
- `Follow` 或 `Subscribe`，如果有业务需求

如果 trader 只是用户视图，service 可以直接基于：

- `UserRepo`
- `RoleRepo`
- `MembershipRepo`

做“过滤出 trader 用户”的列表。

如果 trader 是独立实体，service 则依赖：

- `TraderRepo`

---

## 10. server 层蓝图

### 10.1 `rest_server.go`

做法：

- 复制 `app/admin/service/internal/server/rest_server.go`
- 改 import 路径
- 删除 admin 专用注册
- 只注册用户端 service

保留的注册大致应变成：

- `userV1.RegisterAuthenticationServiceHTTPServer`
- `userV1.RegisterUserProfileServiceHTTPServer`
- `userV1.RegisterTraderServiceHTTPServer`

Swagger 标题改成：

```go
swaggerUI.WithTitle("GoWind User")
```

### 10.2 白名单

用户端白名单通常至少包括：

- 登录
- 注册
- 刷新 token
- 验证码接口（如果未来有）

也就是说：

- `Login`
- `RegisterUser`
- `RefreshToken`

必须加入 `rpc.AddWhiteList(...)`。

### 10.3 是否保留 Asynq 和 SSE

当前 `newApp(...)` 仍然接收：

- `*http.Server`
- `*asynq.Server`
- `*sse.Server`

为了最小改动，建议第一阶段保留：

- `asynq_server.go`
- `sse_server.go`

即使暂时没有用户端任务和 SSE 场景，也先把骨架留着。

---

## 11. 认证与鉴权改造点

这是新增 `app/user/service` 时最容易漏的部分。

### 11.1 `NewClientType()` 改为 `ClientType_app`

文件：

- `app/user/service/internal/data/data.go`

修改：

```go
func NewClientType() authenticationV1.ClientType {
    return authenticationV1.ClientType_app
}
```

### 11.2 `Authenticator` 必须真正支持 `app`

当前 admin 版 `Authenticator` 的问题是：

- 只初始化了 `AdminAuthenticator`
- `getAuthenticator(ClientType_app)` 没返回实际认证器

用户端要么复制一份并补全，要么顺手把公共逻辑抽出来。

用户端版至少要保证：

- `ClientType_app` 能创建 access token
- `ClientType_app` 能校验 access token
- `ClientType_app` 能校验 refresh token
- `ClientType_app` 能撤销 token

建议做法：

```go
type Authenticator struct {
    AdminAuthenticator authnEngine.Authenticator
    AppAuthenticator   authnEngine.Authenticator
    ...
}
```

并在 `NewAuthenticator()` 中同时初始化两者。

如果当前配置密钥相同，可以先复用同一套 key。

### 11.3 `RefreshToken` 不要写死 admin

文件：

- `app/user/service/internal/service/authentication_service.go`

必须把：

```go
req.ClientType = trans.Ptr(authenticationV1.ClientType_admin)
```

改成：

```go
req.ClientType = trans.Ptr(s.clientType)
```

### 11.4 用户端登录授权不要沿用后台权限码

admin 的认证逻辑本质上是在判定：

- 这个用户有没有后台访问权限

用户端应该改成：

- 这个用户是否是可用用户
- 这个租户是否可用
- 这个用户是否具备用户端或 trader 业务资格

推荐最小授权判断：

1. 用户状态为 `NORMAL`
2. 如果启用租户，租户状态为可用
3. 如果是 trader-only 接口，再额外判定 trader 条件

### 11.5 `TokenChecker` 可以继续复用当前模式

只要 `NewClientType()` 改为 `app`，`TokenChecker` 的整体模式可以沿用。

即：

- middleware 继续使用 `pkg/middleware/auth`
- token payload 继续使用 `authentication.service.v1.UserTokenPayload`

---

## 12. `trader` 的落地建议

### 12.1 第一阶段推荐方案

如果你还没完全确认 trader 的边界，先不要建独立 app，也不要急着建一堆 trader 表。

先做：

- `api/protos/user/service/v1/i_trader.proto`
- `app/user/service/internal/service/trader_service.go`

由 `TraderService` 基于现有：

- `UserRepo`
- `RoleRepo`
- `MembershipRepo`

筛出 trader 列表。

常见判定方式：

- 角色 code = `trader`
- membership 中拥有 trader 角色
- user 扩展字段中标记 trader

### 12.2 第二阶段再决定是否独立领域

满足下面任一条件时，再把 trader 升级成独立领域：

- trader 有独立资料表
- trader 有独立审核流程
- trader 有独立统计/排名/策略资产
- trader 的查询条件和生命周期已经明显脱离 user

这时新增：

- `api/protos/trader/service/v1/trader.proto`
- `app/user/service/internal/data/ent/schema/trader.go`
- `app/user/service/internal/data/trader_repo.go`
- `app/user/service/internal/service/trader_service.go`

---

## 13. wire 蓝图

### 13.1 `internal/service/providers/wire_set.go`

用户端版建议最小先保留：

- `service.NewAuthenticationService`
- `service.NewUserProfileService`
- `service.NewTraderService`

### 13.2 `internal/server/providers/wire_set.go`

通常仍保留：

- `server.NewRestServer`
- `server.NewAsynqServer`
- `server.NewSseServer`
- `server.NewRestMiddleware`

### 13.3 `cmd/server/wire.go`

和 admin 一样：

- data providers
- service providers
- server providers
- `newApp`

---

## 14. Makefile 和生成链路

### 14.1 `app/user/service/Makefile`

做法：

- 复制 `app/admin/service/Makefile`

但有一个关键差异：

- `openapi` 目标不能继续调用 `buf.admin.openapi.gen.yaml`

应该改成：

```makefile
openapi:
	@cd ../../../api && \
	buf generate --template buf.user.openapi.gen.yaml
```

### 14.2 生成顺序

建议使用如下顺序：

1. 新增或修改 proto
2. 在 `api/` 下执行 `buf generate`
3. 在 `app/user/service` 下执行 `make wire`
4. 如果改了 schema，再执行 `make ent`
5. 执行 `make build`

建议最终顺序：

```bash
cd app/user/service
make api
make wire
make ent
make build
```

如果 `Makefile` 已经按上面调整，也可以直接：

```bash
make app
```

---

## 15. 建设顺序建议

为了让改动面可控，建议严格按下面顺序做。

### 第一步：搭空壳服务

完成以下最小目标：

- `app/user/service` 目录存在
- `main.go` 能启动
- `wire` 能生成
- `REST server` 能起来

此时哪怕没有业务接口也没关系。

### 第二步：接入认证

完成以下最小目标：

- `ClientType_app` 生效
- `Login`
- `Logout`
- `RefreshToken`
- `RegisterUser`

先把 token 体系打通。

### 第三步：接入个人资料

完成以下接口：

- `GET /user/v1/me`
- `PUT /user/v1/me`
- `POST /user/v1/me/password`

### 第四步：接入 trader

先做最小读接口：

- `GET /user/v1/traders`
- `GET /user/v1/traders/{id}`

之后再决定要不要扩成独立领域。

### 第五步：收缩和重构

新服务跑通后，再考虑：

- 抽公共 repo
- 抽公共 ent schema
- 抽共享 authenticator

不要在第一阶段同时做大重构。

---

## 16. 最小可执行清单

如果按最小版本落地，第一批必须动到的文件是：

### 新增

- `api/protos/user/service/v1/i_authentication.proto`
- `api/protos/user/service/v1/i_user_profile.proto`
- `api/protos/user/service/v1/i_trader.proto`
- `api/buf.user.openapi.gen.yaml`
- `app/user/service/...`

### 修改

- `pkg/serviceid/service_id.go`
- `api/buf.gen.yaml`

### 复制并调整

- `app/admin/service/cmd/server/*` -> `app/user/service/cmd/server/*`
- `app/admin/service/configs/*` -> `app/user/service/configs/*`
- `app/admin/service/internal/server/*` -> `app/user/service/internal/server/*`
- `app/admin/service/internal/data/*` 的最小子集 -> `app/user/service/internal/data/*`
- `app/admin/service/internal/service/authentication_service.go` -> `app/user/service/internal/service/authentication_service.go`
- `app/admin/service/internal/service/user_profile_service.go` -> `app/user/service/internal/service/user_profile_service.go`

---

## 17. 建议的最终边界

建议中期稳定后的边界如下：

- `app/admin/service`
  - 只做管理后台接口
- `app/user/service`
  - 只做普通用户端接口
- `identity/authentication`
  - 继续作为共享领域协议
- `trader`
  - 先放用户端 app 内
  - 业务成熟后再拆独立领域或独立 app

这样做的好处是：

- 外部接口边界清晰
- 内部仍能复用领域模型
- 不会把 admin 继续膨胀成“大一统接口服务”

---

## 18. 实施建议

实际开工时，推荐按下面策略执行：

1. 先复制 `app/admin/service` 为 `app/user/service`
2. 把 service 数量砍到最小
3. 先打通 `Login/Register/Me`
4. 再接入 `TraderService`
5. 最后再考虑公共层抽取

如果一上来同时做：

- 新 app
- 新 proto
- 新认证
- 新 trader 领域
- 公共 repo 抽取

风险会明显偏高。

当前仓库更适合的策略是：

- 先复制跑通
- 后抽象

