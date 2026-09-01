# 第十六章 安全设计

## 本章目标

通过本章，读者将理解：

- 壬远AI网关在传输层、认证授权层、策略执行层分别提供了哪些安全机制；
- API-Key 作为请求链路的直接凭证，如何在控制面存储、如何随 Entity 继承策略、如何在数据面被校验；
- 控制面的认证授权模型（Visitor、Scope、Feature-Action）及其与 OpenAPI / InnerAPI 的集成方式；
- TLS/HTTPS 在控制面与数据面的配置要点；
- 访问日志审计字段如何支撑安全事件追溯与计费对账；
- Redis 中 Quota Key 与 Rate-Limit Key 的清理触发条件与敏感数据保护实践；
- 限流与配额如何作为安全防线防止滥用与成本失控；
- 模型白名单与黑名单如何按 Entity 层级继承；
- 生产环境推荐的安全配置示例。

---

## 安全设计总体视图

壬远AI网关的安全设计采用**分层防御（Defense in Depth）**思想：从外到内依次为传输层安全、身份认证、权限授权、请求准入、策略执行、数据清理与审计追溯。每一层都有独立的校验与拦截能力，即使某一层被绕过，后续层次仍可继续防护。

```
┌─────────────────────────────────────────────────────────────────┐
│                     客户端 / 业务系统                            │
└───────────────────────┬─────────────────────────────────────────┘
                        │
           ┌────────────┴────────────┐
           ▼                         ▼
    TLS/HTTPS 传输加密          客户端 IP / 子网白名单
           │                         │
           └────────────┬────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                        BFE 数据面                                │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ IP 子网控制  │  │ API-Key    │  │ 模型白名单 / 黑名单      │ │
│  │             │  │ 认证与配额  │  │ (来自 API-Key / Entity) │ │
│  └─────────────┘  └──────┬──────┘  └─────────────────────────┘ │
│                          ▼                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ RPM 限流    │  │ TPM 限流    │  │ 并发限流                │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
│                          ▼                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 协议适配、路由转发、后端模型服务调用                      │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                      AI Gateway API 控制面                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ 用户/Token  │  │ Feature-    │  │ API-Key / Entity        │ │
│  │ 认证        │  │ Action 授权 │  │ 管理                    │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ Quota Plan  │  │ Rate Limit  │  │ Redis Key 清理          │ │
│  │ 配额计划    │  │ Policy 策略 │  │ (Quota / Rate-Limit)    │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

上图展示了安全机制在控制面与数据面的分布。控制面负责策略制定、凭证管理与数据清理；数据面负责实时执行认证、限流、配额与黑名单。下文将分层展开。

---

## API-Key 安全存储与传输

### API-Key 的角色与存储

在壬远AI网关中，**API-Key** 是业务系统调用大模型服务时的直接凭证。控制面将 API-Key 存储在 `api_keys` 表中，字段包括 key 值、状态、过期时间 `expired_time`、允许的子网 `allowed_subnets`、挂载的 Entity 等。数据面 BFE 在 `mod_ai_token_auth` 中根据请求头中的 API-Key 进行校验。

API-Key 与 **Entity（业务组织单元）** 通过 `api_keys.entity_id` 关联。挂载到 Entity 后，API-Key 会继承该 Entity 及其父级 Entity 的：

- 模型白名单 `allow_models` 与黑名单 `block_models`；
- 配额计划 QuotaPlan；
- 限流策略 RateLimitPolicy；
- 路由规则 RouteRules。

这种设计使得管理员可以在组织层面统一管控策略，而 API-Key 只作为最终使用凭证。详见 `ai-gateway-api/design-docs/sys-design/details/API-Key与Entity关联及模型继承.md`。

### 传输安全

API-Key 在 HTTP Header 中传输，必须通过 **TLS/HTTPS** 加密，防止中间人窃取或篡改。生产环境中应关闭明文 HTTP 入口，强制 HTTPS 访问。同时建议：

- 定期轮换 API-Key，对长期未使用的 Key 设置过期时间；
- 利用 `allowed_subnets` 限制可调用的客户端 IP 段；
- 在 Dashboard 中避免完整展示 API-Key，仅显示前缀与最后几位；
- API-Key 删除后立即清理其在 Redis 中的 Quota Key 与 Rate-Limit Key，防止残留数据被复用。

---

## TLS/HTTPS 配置

### 数据面 TLS 终止

BFE 作为数据面入口，支持 HTTPS 监听与 TLS 终止。TLS 相关配置位于 `bfe_config/bfe_tls_conf/` 与 `conf/tls_conf/` 目录下，包括证书文件、私钥文件、TLS 规则表等。管理员可以为不同域名配置不同证书，并指定最低 TLS 版本与加密套件。

生产环境建议：

- 禁用 TLS 1.0/1.1，仅启用 TLS 1.2 及以上版本；
- 使用由可信 CA 签发的证书，避免自签名证书；
- 证书与私钥文件权限设置为 `600`，避免被非授权用户读取；
- 开启 HSTS（HTTP Strict Transport Security）头，强制客户端使用 HTTPS。

### 控制面 HTTPS

AI Gateway API 与 Dashboard 也应通过 HTTPS 暴露管理接口。虽然当前默认配置以 HTTP 启动，但生产部署时应在 AI Gateway API 前部署 Nginx/Envoy 等反向代理完成 TLS 终止，或直接在应用层启用 TLS。管理面涉及 API-Key 创建、Token 分发、密码修改等敏感操作，必须加密传输。

---

## 认证授权机制

### 访问者模型

AI Gateway API 的认证授权模块位于 `model/iauth`，负责 OpenAPI 与 InnerAPI 的访问者身份识别与权限控制。该模块复用了 BFE 历史代码中 `users` 表同时存储“用户”与“Token”的设计，通过 `type` 字段区分：

| `type` 值 | 含义 | 典型用途 |
|----------|------|---------|
| `0` | 普通用户 | Dashboard 管理员登录、人工操作 |
| `1` | Token | 程序化调用、Conf Agent / BFE 拉取 InnerAPI |

代码中通过 `Visitor` 统一抽象用户与 Token：

```go
type Visitor struct {
    User  *User
    Token *Token
}
```

`Visitor` 实现 `Loginer` 接口，统一提供 `GetName`、`GetScopes`、`GetType`、`IsAdmin` 方法，后续授权校验只关心 `Visitor`，不关心底层是用户还是 Token。详见 `ai-gateway-api/design-docs/sys-design/details/认证授权机制.md`。

### 四种认证方式

控制面支持四种认证方式：

```go
const (
    AuthTypePassword   = "Password"
    AuthTypeSessionKey = "Session"
    AuthTypeToken      = "Token"
    AuthTypeSkip       = "Skip"
)
```

| 方式 | 请求头示例 | 适用场景 | 是否写入 Session |
|------|-----------|---------|----------------|
| `Password` | `Authorization: Password <base64(user:pass)>` | 登录获取 Session Key | 是 |
| `Session` | `Authorization: Session <session_key>` | 常规 OpenAPI 调用 | 否，仅校验 ticket 是否过期 |
| `Token` | `Authorization: Token <token_value>` | 程序化调用 / InnerAPI | 否，Token 长期有效 |
| `Skip` | `Authorization: Skip System` | 调试跳过 | 否，生成伪造 Visitor |

`Skip` 认证仅在 `RunTime.SkipTokenValidate = true` 时生效，**严禁在生产环境开启**。

### Feature-Action 权限模型

授权采用 **Feature（功能维度）+ Action（操作维度）** 模型：

```go
type FeatureAuthorition struct {
    Feature Feature
    Action  Action
}
```

`Action` 使用位掩码：`ActionRead`、`ActionUpdate`、`ActionCreate`、`ActionDelete`、`ActionExport` 等。每个 OpenAPI 端点通过 `Authorizer` 字段声明所需权限，例如：

```go
var APIKeyCreateRoute = &xreq.Endpoint{
    Path:       "/api-keys",
    Method:     http.MethodPost,
    Authorizer: iauth.FA(iauth.FeatureAPIKey, iauth.ActionCreate),
}
```

`AuthorizeManager.Authorizate` 执行以下步骤：

1. 从 `context` 取出 `Visitor`；
2. 若 `Visitor.IsAdmin()` 为 `true`，直接放行；
3. 遍历 `Visitor.GetScopes()`，在 `scope2permission` 中查找对应 Feature 的 Action 位图；
4. 判断所需 Action 是否被允许；
5. 若需要，再校验 Visitor 与当前产品线的绑定关系。

### 中间件链

控制面的全局中间件链为：

```
HTTP Request
    │
    ▼
MCRecovery ──► MCLogger ──► MCCors
    │
    ▼
/open-api/v1: McProductProbe ──► McUserProbe
/inner-api/v1: McUserProbe
```

`McUserProbe` 负责解析 `Authorization` Header 并完成认证，将 `Visitor` 写入 `context`；`McProductProbe` 负责从 URL Path 中解析产品线上下文。每个带 `Authorizer` 的 Endpoint 会单独创建 Subrouter 并挂载鉴权中间件。

---

## 访问日志审计

### 数据面错误码与日志字段

BFE 数据面在请求处理各阶段（认证、限流、配额、转发）会返回结构化错误响应，并通过 `mod_access_pb3` 输出访问日志。与安全审计相关的日志字段包括：

| 日志字段 | 说明 |
|----------|------|
| `ai_auth_reject_reason` | 认证/配额拒绝时的错误码 |
| `ai_auth_reject_quota_plans` | 因配额不足被拒绝时，余量不足的 Quota Plan ID 列表 |
| `ai_auth_hit_quota_plans` | 认证通过且余额充足的 Quota Plan ID 列表 |
| `ai_rate_limit_hits` | 触发的限流策略及规则名列表 |

典型错误码包括 `NO_API_KEY`、`INVALID_API_KEY`、`KEY_DISABLED`、`KEY_EXPIRED`、`SUBNET_NOT_ALLOWED`、`MODEL_NOT_ALLOWED`、`QUOTA_EXHAUSTED`、`RPM_LIMIT_EXCEEDED`、`TPM_LIMIT_EXCEEDED` 等。详见 `bfe/docs/zh_cn/sys_design/ai_error_codes.md`。

### 错误响应体

数据面错误响应采用 OpenAI 兼容格式，便于上游业务系统统一处理：

```json
{
  "error": {
    "code": "QUOTA_EXHAUSTED",
    "type": "quota_error",
    "message": "Quota plan qplan-0001 exhausted.",
    "param": null,
    "details": {
      "api_key": "ak-2v8x9k3m7p",
      "key_id": "key-001",
      "quota_plan_id": "qplan-0001",
      "limit_type": "api_key_quota",
      "model": "gpt-4",
      "retry_after_seconds": 0
    }
  }
}
```

通过集中收集访问日志，安全团队可以识别异常调用模式，例如：

- 某个 API-Key 在短时间内频繁触发 `INVALID_API_KEY` 或 `KEY_DISABLED`；
- 特定 IP 段大量触发 `QUOTA_EXHAUSTED`；
- 某个模型被大量请求但多数返回 `MODEL_NOT_ALLOWED`。

---

## Redis Key 清理与敏感数据保护

### Redis 中的敏感 Key

控制面运行时会向 Redis 写入两类关键状态 Key：

- **Quota Key**：记录 API-Key / Entity 的实时剩余额度，格式为 `QUOTA_<api-key-token>` 或 `QUOTA_<entity-id>`；
- **Rate-Limit Key**：记录限流规则在当前窗口内的已消耗 Token 数 / 请求数，格式如 `RL_TPM_rlp-<policyID>_<name>` 与 `RL_RPM_rlp-<policyID>_<name>`。

BFE 实际访问时还会拼接前缀 `default_bfe_<policyId>_...`，形成完整 Key。

### 清理触发场景

为避免 API-Key、Entity 被删除后 Redis 中残留无用 Key，控制面在以下场景主动清理：

1. API-Key 删除：清理该 API-Key 的 Quota Key 及其直接绑定 RateLimitPolicy 的 Rate-Limit Key；
2. Entity 删除：清理该 Entity 的 Quota Key 及其直接绑定 RateLimitPolicy 的 Rate-Limit Key；
3. API-Key 更新导致 rate-limit 规则被删除：按规则 `name` 精确匹配并清理旧规则 Key；
4. Entity 更新导致 rate-limit 规则被删除：同上，仅清理当前 Entity 的 Key。

配额 plan 的常规变更（改配额、改单位、切换 unlimited）不会删除 Quota Key，只会覆盖其值。清理操作通过 `QuotaCache.DeleteKeys` 使用 Redis Pipeline 批量执行 `UNLINK`（小 Key 可直接 `DEL`）。详见 `ai-gateway-api/design-docs/sys-design/details/Redis Key 清理机制.md`。

### 敏感数据保护

- Redis 应开启认证（`requirepass`）与 TLS 连接，避免未授权访问；
- 避免在日志中打印完整 API-Key 或 Redis Key；
- 对 Redis 实例进行网络隔离，仅允许 AI Gateway API 与 BFE 访问；
- 定期审计 Redis 中是否存在长期未更新的孤立 Key。

---

## 限流与配额作为安全防线

### 多层级策略继承

API-Key 可挂载到 Entity，从而继承 Entity 层级向上的配额计划与限流策略。导出到 BFE 时：

- 收集 API-Key 自身及 Entity 层级向上的所有**非无限**配额计划；
- 收集 API-Key 自身及 Entity 层级向上的所有**启用**限流策略；
- 每个配额计划生成独立的 `QuotaPlan.RedisKey`，API-Key 可能同时受多个 Redis Key 控制。

这种设计使得安全策略可以在组织、部门、项目、应用多个层级生效，避免单点配置遗漏。

### 防止滥用与成本失控

限流与配额是防止 API-Key 泄露或滥用后的第二道防线：

- **RPM/TPM/并发限流**：防止突发流量压垮后端或耗尽服务商配额；
- **Quota Plan**：按 Token 数或金额设置预算上限，防止成本失控；
- **模型白名单/黑名单**：限制可调用的模型范围，避免调用高成本模型。

当触发限流或配额耗尽时，BFE 返回标准化错误响应，例如 `RPM_LIMIT_EXCEEDED`、`QUOTA_EXHAUSTED`，业务系统可根据 `retry_after_seconds` 等字段进行退避重试。

---

## 模型白名单与黑名单

### 模型白名单与黑名单

壬远AI网关通过 Entity 继承机制提供模型级黑白名单：

- `allow_models`：取层级交集，只有同时满足所有层级限制的模型才允许访问；
- `block_models`：取层级并集，任一父级禁止的模型都会被拦截。

若 API-Key 自身 `allow_models` 与 Entity 继承结果交集为空，则该 API-Key 导出时会被禁用（`Enabled=false`），从数据面直接拒绝请求。

### 请求准入流程

```
客户端请求
    │
    ▼
┌─────────────────┐
│ IP 子网白名单    │ ──► 未命中则返回 SUBNET_NOT_ALLOWED
└────────┬────────┘
         │ 命中
         ▼
┌─────────────────┐
│ API-Key 认证     │ ──► 失败返回 NO_API_KEY / INVALID_API_KEY 等
└────────┬────────┘
         │ 通过
         ▼
┌─────────────────┐
│ 模型白名单/黑名单 │ ──► 失败返回 MODEL_NOT_ALLOWED
└────────┬────────┘
         │ 通过
         ▼
┌─────────────────┐
│ 配额 / 限流检查  │ ──► 失败返回 QUOTA_EXHAUSTED / RPM_LIMIT_EXCEEDED 等
└────────┬────────┘
         │ 通过
         ▼
    后端模型服务
```

上图展示了从网络层到应用层的逐层拦截顺序。IP 子网白名单、API-Key 认证、模型黑白名单、配额与限流依次执行，形成多层准入控制。

---

## 安全配置示例

### 控制面运行时安全配置

`ai_gateway_api.toml` 中与安全直接相关的配置项：

```toml
[RunTime]
# 生产环境必须关闭，严禁设为 true
SkipTokenValidate = false

# SQL 日志仅调试时开启
RecordSQL = false

# Session Key 过期时间，建议不超过 7 天
SessionExpireInDay = 7

# 关闭响应 Debug 信息
Debug = false
```

### 数据面 TLS 配置示例

BFE 的 TLS 配置通常位于 `conf/tls_conf/tls_rule_conf.data`，示例片段：

```json
{
  "Config": {
    "example_product": {
      "DefaultNextProtos": "http/1.1;h2",
      "CertName": "example_product",
      "SniConf": null
    }
  },
  "DefaultSubProtocols": "http/1.1;h2",
  "DefaultMaxVersion": "VersionTLS13",
  "DefaultMinVersion": "VersionTLS12"
}
```

证书文件 `conf/tls_conf/example_product.crt` 与私钥 `conf/tls_conf/example_product.key` 应严格限制文件权限。

### API-Key 安全策略示例

创建 API-Key 时建议同时配置：

```json
{
  "name": "app-llm-proxy",
  "entity_id": "ent-project-x",
  "expired_time": "2026-01-01T00:00:00Z",
  "allowed_subnets": ["10.0.0.0/24", "192.168.1.0/24"],
  "allowed_models": ["gpt-4", "gpt-3.5-turbo"]
}
```

配合 Entity 层级，可在更高层设置 `block_models: ["gpt-4-32k"]`，实现自上而下的模型管控。

---

## 本章小结

- 壬远AI网关采用分层防御思想，安全机制覆盖传输层、认证授权层、请求准入层与策略执行层。
- API-Key 是请求链路的直接凭证，应通过 HTTPS 传输，并结合 Entity 继承实现组织级策略管控。
- 控制面采用 Visitor 抽象与 Feature-Action 权限模型，支持 Password、Session、Token、Skip 四种认证方式；生产环境必须关闭 `SkipTokenValidate`。
- 数据面 BFE 输出结构化错误响应与访问日志字段，支撑安全审计、异常检测与计费对账。
- Redis 中的 Quota Key 与 Rate-Limit Key 在 API-Key / Entity 删除或策略变更时会被主动清理，避免残留数据带来的安全风险。
- 限流与配额是防止 API-Key 泄露、滥用和成本失控的关键防线，支持 API-Key 与 Entity 多层级继承。
- IP 子网控制、API-Key 认证、模型黑白名单、配额与限流共同构成请求准入的安全屏障，按“IP 白名单 → API-Key 认证 → 模型黑白名单 → 配额/限流”顺序逐层拦截。

---

## 参考文档

- `ai-gateway-api/design-docs/sys-design/details/认证授权机制.md`
- `ai-gateway-api/design-docs/sys-design/details/API-Key与Entity关联及模型继承.md`
- `ai-gateway-api/design-docs/sys-design/details/Redis Key 清理机制.md`
- `bfe/docs/zh_cn/sys_design/ai_error_codes.md`
- `ai-gateway-api/docs/zh_cn/config_param.md`
