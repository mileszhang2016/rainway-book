# 第十七章 控制台基础操作

## 本章目标

通过本章，读者将掌握壬远 AI 网关（Rainway AI Gateway）Dashboard 的访问方式、界面组织与基础操作流程，理解控制台背后的核心概念（产品、域名、实例池、Provider、Cluster、Entity 等），并能够独立完成首次配置。具体包括：

- 如何登录 Dashboard 并修改默认账号；
- 控制台各导航入口的职责与数据对应关系；
- 用户、Token 与权限 Scope 的管理方式；
- 全局配置视图与配置版本/变更记录的查看方法；
- 从创建 Provider 到使流量生效的完整首次配置流程。

---

## 17.1 Dashboard 访问方式与默认账号

Dashboard 是壬远 AI 网关的管理控制台（Admin Console），它以 Web UI 形式调用 AI Gateway API 的 OpenAPI v1 接口，完成策略与配置的可视化管理。在本地或测试环境启动 AI Gateway API 后，默认可通过浏览器访问：

```
http://api-server:8183/
```

其中 `8183` 为 AI Gateway API 的服务端口（ServerPort），可在 `conf/ai_gateway_api.toml` 的 `[Server]` 段落中修改；监控端口（MonitorPort）默认为 `8284`，用于暴露指标与健康检查，不直接提供控制台界面。

首次登录时，系统预置管理员账号：

| 项目 | 默认值 |
|------|--------|
| 用户名 | `admin` |
| 密码 | `admin` |

登录成功后，Dashboard 会保存由 `/auth/session-keys` 生成的 session key，并在后续请求中以 `Authorization: Session {session_key}` 的方式携带鉴权信息。生产环境中应在首次登录后立即修改默认密码，并为不同运维人员创建独立账号。

---

## 17.2 控制台界面导览

Dashboard 的导航由 `/meta` 接口动态返回，对应 `conf/nav_tree.toml` 中定义的导航树。当前管理员（admin）视角下的主导航包括资源管理、路由管理、消费者管理、用户管理四大类，与 OpenAPI v1 的模块划分基本一致。界面布局通常如下：

```
┌─────────────────────────────────────────────┐
│  Logo / 产品名称                [用户头像 ▼]  │
├──────────┬──────────────────────────────────┤
│          │                                  │
│ 资源管理  │  Provider 管理                   │
│  · 实例池 │  · Cluster 管理                  │
│  · Cluster│  · 模型定价管理                   │
│  · 模型定价│                                  │
│          ├──────────────────────────────────┤
│ 路由管理  │  高级路由规则                     │
│  · 高级规则│                                  │
│          ├──────────────────────────────────┤
│ 消费者管理 │  API-Key 管理                    │
│  · API-Key│  · Entity 管理                   │
│  · Entity │                                  │
│          ├──────────────────────────────────┤
│ 用户管理  │  用户 / Token 管理                │
│          │                                  │
└──────────┴──────────────────────────────────┘
```

每个导航项对应一组 OpenAPI 资源：

| 导航项 | 对应 OpenAPI 端点 | 主要职责 |
|--------|-------------------|----------|
| Provider 管理 | `/providers` | 维护模型提供方、后端实例池、API-Key 明文与模型协议 |
| Cluster 管理 | `/clusters` | 维护转发集群、LLM 配置、Key 权重与路由策略 |
| 模型定价管理 | `/model-prices` | 维护模型在不同 provider 与 tier 下的价格 |
| 高级路由规则 | `/global-route-rules` 等 | 维护 global / entity / api-key 三级路由规则 |
| API-Key 管理 | `/api-keys` | 创建、启用/禁用、配额绑定与密钥查看 |
| Entity 管理 | `/entities`、`/entity-types` | 维护组织架构、模型黑白名单、配额与限流策略 |
| 用户管理 | `/auth/users`、`/auth/tokens` | 维护控制台用户与机器 Token |

---

## 17.3 产品/域名/实例池等基础概念

控制台中的诸多操作都围绕以下基础概念展开，理解它们是正确使用 Dashboard 的前提。

### 17.3.1 产品（Product）

产品（Product）是壬远 AI 网关中的顶层资源隔离单位，对应数据库中的 `products` 表。历史上它曾用于区分不同业务线，并在中间件 `McProductProbe` 中解析产品线上下文。当前版本中，控制台面向管理员时默认工作在单一产品视图下，后续若扩展多租户能力，产品将成为权限隔离与配置分组的基础。

### 17.3.2 域名（Domain）

域名（Domain）对应 `domains` 表，是流量进入 BFE 数据面的入口标识。控制面在导出 Server Data 配置时，会将域名、基础/高级路由规则与集群配置组装成 BFE 可消费的 `HostTable`、`RouteTable` 与 `ClusterConf`。Dashboard 中的路由规则配置最终都会映射到具体的域名命中条件上。

### 17.3.3 实例池（Instance Pool）

实例池（Instance Pool）在 Provider 数据模型中通过 `instance_pool` 字段定义，描述下游 AI 服务的真实后端地址、端口与权重。与早期将实例信息直接写在 Cluster 中不同，当前架构把实例池收敛到 Provider 中，供多个 Cluster 复用。创建或更新 Provider 时，控制面会根据 `instance_pool` 自动生成实例池、子集群并完成绑定；修改 Provider 的实例池时，也会同步刷新引用该 Provider 的所有 Cluster 所生成的实例池。

### 17.3.4 Provider 与 Cluster

Provider（提供商）回答“下游是谁、能访问哪些模型、如何认证、后端在哪里”的问题；Cluster（集群）回答“流量如何转发、用哪些模型、Key 权重如何分配”的问题。二者解耦后，Cluster 通过 `llm_config.provider` 强引用 Provider，而 Provider 可以被多个 Cluster 共享。详细设计动机与数据模型参见 [第九章 Provider 与 Cluster 设计](../design/chapter09-provider-and-cluster.md)。

### 17.3.5 Entity 与 API-Key

Entity（实体）用于表达组织架构，例如部门、团队或项目。每个 Entity 拥有独立的模型白名单/黑名单、配额计划（QuotaPlan）、限流策略（RateLimitPolicy）与路由规则。API-Key 则是调用方访问 AI 网关的凭证，可与 Entity 关联以继承其配额与限流策略，也可拥有独立的 api-key 级路由规则。

---

## 17.4 用户与权限管理

Dashboard 的用户与权限由 `/auth` 接口族管理，相关定义详见 `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/auth.md`。

### 17.4.1 用户（User）

用户是登录 Dashboard 的自然人账号，当前版本仅支持管理员用户（`is_admin=true`），即拥有 System 权限。创建用户时需要指定 `user_name`、`password` 与 `is_admin`；密码不能等于用户名或其逆序。管理员可通过 Dashboard 的“用户管理”入口完成新增、删除与密码重置。

### 17.4.2 Session Key 与 Token

- **Session Key**：由 `/auth/session-keys` 根据用户名和密码生成，用于 Dashboard 登录后的请求鉴权，格式为 `Authorization: Session {session_key}`。Session 有过期时间，默认由 `SessionExpireInDay` 控制。
- **Token**：由 `/auth/tokens` 创建，主要用于机器调用或数据面组件（如 Conf Agent、BFE）访问 InnerAPI。Token 分为两种 Scope：
  - `System`：全部权限，包括全局配置、产品线资源和导出资源；
  - `Support`：仅导出类资源，供 BFE 数据面模块导出配置。

生产环境中，应为 Dashboard 操作人员创建独立用户，为 Conf Agent 等机器客户端创建 `Support` Scope 的专用 Token，避免使用 `System` Token 直接暴露给数据面。

---

## 17.5 全局配置视图

Dashboard 的“全局配置视图”帮助管理员纵览当前系统的关键资源状态，通常包括：

- **路由表（Route Table）**：通过 `/route-tables` 查看 `global`、`entity`、`api_key` 三类路由表的启用状态与所有者；
- **Global 路由规则**：通过 `/global-route-rules` 查看或编辑全局默认路由，决定未命中其他规则时的流量去向；
- **Provider / Cluster 列表**：展示已配置的 Provider、Cluster 及其健康状态引用关系；
- **API-Key / Entity 列表**：展示已发放的 API-Key、关联的 Entity 与配额余额。

全局路由规则示例（JSON 视图）如下：

```json
{
  "enabled": true,
  "rules": [
    {
      "name": "global-default",
      "cond": "default_t()",
      "targets": [
        {
          "cluster_name": "deepseek-cluster",
          "model": "",
          "weight": 100
        }
      ],
      "fallbacks": []
    }
  ]
}
```

其中 `cond` 为 BFE 条件表达式，`targets` 中同一规则的权重之和必须等于 100，`fallbacks` 用于指定降级目标。

---

## 17.6 配置版本与变更记录查看

壬远 AI 网关采用基于 MD5 签名与版本号的配置导出机制，详细设计参见 [第十三章 配置导出与版本控制设计](../design/chapter13-config-export-and-version-control.md)。在 Dashboard 中，管理员可以在“全局配置”或“版本管理”入口查看以下信息：

- **配置主题（Topic）**：如 `route_rule`、`ai_route`、`mod_api_key_rule` 等；
- **当前版本号**：格式为 `YYYYMMDDHHMMSS`，例如 `20260102120000`；
- **MD5 签名**：对配置内容（版本号置零后）计算得出的签名；
- **最近变更时间**：签名变化即视为一次有效配置变更；
- **与上一版本的差异**：部分视图支持对比两次导出之间的配置差异。

该机制保证：若配置内容未发生变化，Conf Agent 拉取时返回 `Data: null`，避免无意义的热加载；一旦内容变化，控制面会生成新的版本号并返回全量配置，BFE 据此完成热更新。

---

## 17.7 通过 Dashboard 进行首次配置的完整流程

以下是一个从空环境到流量可转发的最小配置流程，适用于首次使用 Dashboard 的读者。

### 步骤 1：登录并修改默认密码

使用默认账号 `admin/admin` 登录后，进入“用户管理”，将 `admin` 密码修改为强密码，并根据团队需要新建其他管理员账号。

### 步骤 2：创建 Provider

进入“Provider 管理”，点击创建，填写：

- `name`：Provider 唯一标识，例如 `deepseek`；
- `model_endpoint`：模型发现端点，默认 `https://api.deepseek.com/v1/models`；
- `models`：支持的模型列表，例如 `["deepseek-chat"]`；
- `keys`：后端认证 key 的明文，例如 `{ "name": "key-primary", "key": "sk-..." }`；
- `instance_pool`：后端实例，例如 `{ "addr": "api.deepseek.com", "port": 443, "weight": 100 }`；
- `model_protocols`：协议类型，例如 `["openai"]`。

创建成功后，控制面会自动根据 `instance_pool` 生成实例池和子集群。

### 步骤 3：创建 Cluster

进入“Cluster 管理”，创建 Cluster 并引用上一步的 Provider：

- `name`：Cluster 唯一标识，例如 `deepseek-cluster`；
- `llm_config.provider`：选择 `deepseek`；
- `llm_config.models`：选择该 Cluster 可转发的模型子集；
- `llm_config.keys`：引用 Provider 中的 key 并设置权重，权重之和须为 100；
- `basic`、`sticky_sessions`、`passive_health_check`：按需调整，未传时使用 AI 网关场景默认值。

### 步骤 4：配置 Global 路由规则

进入“高级路由规则”或“Global 路由规则”，启用默认规则并指定转发目标为 `deepseek-cluster`。例如：

```json
{
  "enabled": true,
  "rules": [
    {
      "name": "default-to-deepseek",
      "cond": "default_t()",
      "targets": [
        { "cluster_name": "deepseek-cluster", "model": "", "weight": 100 }
      ],
      "fallbacks": []
    }
  ]
}
```

### 步骤 5：创建 API-Key（可选但推荐）

进入“API-Key 管理”，创建供客户端调用的 API-Key，并可选择关联 Entity 以继承配额与限流策略。创建完成后，Dashboard 会展示该 API-Key 的密钥，请务必妥善保存。

### 步骤 6：验证配置生效

完成以上步骤后，可通过以下方式验证：

1. 在 Dashboard 的“版本管理”中查看 `ai_route`、`route_rule` 等 Topic 的版本号是否已更新；
2. 触发 Conf Agent 拉取或等待其定时轮询，确认 BFE 已完成热加载；
3. 使用 curl 或客户端发送请求，验证流量是否按 Global 规则转发到 `deepseek-cluster`。

---

## 17.8 操作注意事项

在使用 Dashboard 进行日常运维时，应注意以下事项：

1. **默认账号安全**：`admin/admin` 仅用于首次登录，生产环境必须立即修改密码并限制访问来源。
2. **权限模型现状**：当前版本用户只有 `System` 权限，暂不支持非管理员用户。若未来版本引入 `Support` 等更多角色，应按最小权限原则分配。
3. **Token 管理**：为 Conf Agent 或自动化脚本创建 Token 时，优先使用 `Support` Scope，避免使用 `System` Token 进行只读导出操作。
4. **Provider 变更的级联影响**：修改 Provider 的 `instance_pool`、`keys` 或 `models` 会触发引用它的 Cluster 同步更新；删除 Provider 前必须确保无 Cluster 引用，否则将返回 `409 Conflict`。
5. **Cluster 删除的引用检查**：删除 Cluster 前，系统会检查其是否被 Global / Entity / API-Key 路由规则引用；若被引用，需先解除引用或修改规则。
6. **配置生效延迟**：Dashboard 中的修改写入 MySQL 后，需经 Conf Agent 拉取并触发 BFE 热加载才能生效。不要期望配置保存后立即在数据面生效。
7. **版本号变化才代表真实变更**：即使多次点击保存，只要配置内容的 MD5 签名未变，`config_versions` 中的版本号就不会增加，数据面也不会重新加载。
8. **API-Key 明文只显示一次**：创建 API-Key 时，Dashboard 通常只会在创建成功后展示一次密钥，后续无法再次查看完整明文，请妥善保管。

---

## 本章小结

本章介绍了壬远 AI 网关 Dashboard 的基础操作。主要内容包括：

- Dashboard 默认通过 `http://api-server:8183/` 访问，初始账号为 `admin/admin`；
- 控制台导航由 `/meta` 动态返回，覆盖资源管理、路由管理、消费者管理与用户管理四大模块；
- 产品、域名、实例池、Provider、Cluster、Entity、API-Key 是控制台中的核心概念，理解它们的职责与引用关系是正确配置的前提；
- 用户、Session Key、Token 构成 Dashboard 与 API 的鉴权体系，Token 分为 `System` 与 `Support` 两种 Scope；
- 全局配置视图可纵览路由表、Global 路由规则与各类资源；
- 配置版本基于 MD5 签名与 `YYYYMMDDHHMMSS` 版本号实现增量同步；
- 首次配置应按照“Provider → Cluster → Global 路由规则 → API-Key → 验证生效”的顺序进行；
- 日常运维需注意默认账号安全、级联引用检查、配置生效延迟与 Token 最小权限原则。

掌握本章内容后，读者即可在 Dashboard 中完成壬远 AI 网关的初始化和基础管理操作，为后续章节中 Provider、Cluster、API-Key、限流等专项配置打下基础。

---

## 参考文档

- `ai-gateway-api/README.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/README.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/auth.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/providers.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/clusters.md`
- `ai-gateway-api/conf/ai_gateway_api.toml`
- `ai-gateway-api/conf/nav_tree.toml`
- [第六章 控制面核心设计：AI Gateway API](../design/chapter06-control-plane-design.md)
- [第九章 Provider 与 Cluster 设计](../design/chapter09-provider-and-cluster.md)
- [第十三章 配置导出与版本控制设计](../design/chapter13-config-export-and-version-control.md)
