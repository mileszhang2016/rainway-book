# 第十八章 控制台基础操作

## 本章目标

通过本章，读者将掌握壬远 AI 网关（Rainway AI Gateway）Dashboard 的访问方式、界面组织与基础操作流程，理解控制台背后的核心概念（AI 网关实例池、模型服务商、AI 业务集群、Entity、API Key、路由表等），并能够独立完成首次配置。具体包括：

- 如何登录 Dashboard 并修改默认账号；
- 控制台各导航入口的职责与数据对应关系；
- 用户、Token 与权限 Scope 的管理方式；
- 控制台通用交互约定（抽屉表单、编辑模式、通知）；
- 配置版本跟踪机制；
- 从实例池登记到第一次 curl 调用成功的完整首次配置流程。

---

## Dashboard 访问方式与默认账号

Dashboard 是壬远 AI 网关的管理控制台（Admin Console），它以 Web UI 形式调用 AI Gateway API 的 OpenAPI v1 接口，完成策略与配置的可视化管理。在本地或测试环境启动 AI Gateway API 后，默认可通过浏览器访问登录页：

```
http://api-server:8183/login
```

直接访问 `http://api-server:8183/` 未登录时也会重定向到登录页。其中 `8183` 为 AI Gateway API 的服务端口（ServerPort），可在 `conf/ai_gateway_api.toml` 的 `[Server]` 段落中修改；监控端口（MonitorPort）默认为 `8284`，用于暴露指标与健康检查，不直接提供控制台界面。

首次登录时，系统预置管理员账号：

| 项目 | 默认值 |
|------|--------|
| 用户名 | `admin` |
| 密码 | `admin` |

登录成功后，Dashboard 会保存由 `/auth/session-keys` 生成的 session key，并在后续请求中以 `Authorization: Session {session_key}` 的方式携带鉴权信息。生产环境中应在首次登录后立即修改默认密码，并为不同运维人员创建独立账号。

---

## 控制台界面导览

登录成功后，控制台呈现「左侧导航 + 右侧内容区」的布局。导航由 `/meta` 接口动态返回，对应 `conf/nav_tree.toml` 中定义的导航树。当前管理员视角下的主导航包括资源管理、消费者管理、路由管理、用户管理四大类：

```
AI 网关
├─ 资源管理
│   ├─ AI 网关实例池      数据面转发引擎地址登记
│   ├─ 模型服务商          实例池、协议、模型与 Keys
│   ├─ AI 业务集群        引用服务商，配置转发策略
│   └─ 模型定价           模型价格维护与费用核算
├─ 消费者管理
│   ├─ Entity 管理       类型 / 组织（配额、限流、模型访问控制）
│   └─ API Key 管理      调用凭证签发与治理
├─ 路由管理
│   └─ 路由表            Global / Entity / API-Key 路由规则
└─ 用户管理              控制台账号与 Token（管理员）
```

每个导航项对应一组 OpenAPI 资源：

| 导航项 | 对应 OpenAPI 端点 | 主要职责 |
|--------|-------------------|----------|
| AI 网关实例池 | `/server-data`（导出侧） | 登记数据面 BFE 引擎地址，供控制面下发配置 |
| 模型服务商 | `/providers` | 维护模型提供方、后端实例池、API-Key 明文与模型协议 |
| AI 业务集群 | `/clusters` | 维护转发集群、LLM 配置、Key 权重与路由策略 |
| 模型定价 | `/model-prices` | 维护模型在不同 provider 与 tier 下的价格 |
| Entity 管理 | `/entity-types`、`/entities` | 维护组织类型、组织架构、模型黑白名单、配额与限流策略 |
| API Key 管理 | `/api-keys` | 创建、启用/禁用、配额绑定与密钥查看 |
| 路由表 | `/global-route-rules`、`/route-tables` 等 | 维护 global / entity / api-key 三级路由规则 |
| 用户管理 | `/auth/users`、`/auth/tokens` | 维护控制台用户与机器 Token |

### 通用交互约定

- **列表页**：搜索区 + 操作按钮 + 表格 + 分页；多数列表支持「20 条/页」切换。
- **抽屉表单**：创建 / 编辑在右侧抽屉完成，底部「提交 / 取消」（向导类为「下一步 / 上一步」）。
- **编辑模式**：路由规则等高风险配置需先「进入编辑模式」，改完「本地保存」，再点「提交并生效」（提交后自动退出编辑模式），未提交的修改不影响线上流量。
- **通知**：操作结果以右上角通知呈现；错误通知（如「数据重复」）不自动消失，点 × 关闭。

---

## 控制台中的核心概念

控制台中的诸多操作都围绕以下概念展开：AI 网关实例池、Provider（模型服务商）、Cluster（AI 业务集群）、Entity（组织）、API-Key、路由表、配额计划与限流策略。

这些概念的完整定义、相互关系与设计动机已在 [第五章 壬远AI网关架构与核心概念](../design/chapter05-system-architecture.md#核心概念) 中统一介绍。本节仅说明它们在 Dashboard 中的对应入口：

| 控制台导航项 | 对应概念 | 说明 |
|---|---|---|
| 资源管理 → AI 网关实例池 | AI 网关实例池 | 登记数据面 BFE 引擎地址 |
| 资源管理 → 模型服务商 | Provider | 维护模型提供方、后端实例池、模型协议与认证密钥 |
| 资源管理 → AI 业务集群 | Cluster | 引用 Provider，配置转发策略、Key 权重与超时 |
| 资源管理 → 模型定价 | Model Price | 维护模型在不同 Provider 与时段下的单价 |
| 消费者管理 → Entity 管理 | Entity | 维护组织架构、配额、限流与模型访问控制 |
| 消费者管理 → API Key 管理 | API-Key | 签发调用凭证并绑定到 Entity |
| 路由管理 → 路由表 | Route Table | 维护 Global / Entity / API-Key 三级路由规则 |

理解这些概念是正确使用 Dashboard 的前提，建议在阅读本章前先浏览第五章的“核心概念”节。

---

## 用户与权限管理

Dashboard 的用户与权限由 `/auth` 接口族管理，相关定义详见 `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/auth.md`。

### 用户（User）

用户是登录 Dashboard 的自然人账号，当前版本仅支持系统管理员角色。创建用户时需要指定用户名、密码与确认密码：

- 用户名最多 64 字符，仅允许字母、数字、点（`.`）、下划线（`_`）、中划线（`-`），不能以点、下划线、中划线开头或结尾；保留名 `admin`、`root`、`system` 不可用。
- 密码 8–128 字符，不能包含空格，不能与用户名相同或为其逆序。

管理员可通过 Dashboard「用户管理 → 用户」页签完成新增、删除与密码重置。修改自己密码时需要填写原密码，提交成功后自动退出登录；修改他人密码无需原密码。

### Session Key 与 Token

- **Session Key**：由 `/auth/session-keys` 根据用户名和密码生成，用于 Dashboard 登录后的请求鉴权，格式为 `Authorization: Session {session_key}`。Session 有过期时间，默认由 `SessionExpireInDay` 控制。
- **Token**：由 `/auth/tokens` 创建，是内部程序访问 API Server（管理面 API 服务，非数据面转发入口）的鉴权凭证。Token 分为两种 Scope：
  - **系统管理（System）**：拥有所有资源的完整管理权限；
  - **内部支持（Support）**：仅可导出部分资源的数据，适用于内部运维排查、数据备份等只读场景，不可创建、编辑或删除任何资源。

生产环境中，应为 Dashboard 操作人员创建独立用户，为 Conf Agent 等机器客户端创建 `Support` Scope 的专用 Token，避免使用 `System` Token 直接暴露给数据面。

---

## 核心模块速查

控制台各模块通过独立的列表页与表单页完成管理。以下是常用模块的入口与职责：

| 模块路径 | 列表页能力 | 关键操作 |
|----------|-----------|---------|
| 资源管理 → AI 网关实例池 | 查看已登记的数据面 BFE 地址 | 新增、编辑、删除实例池 |
| 资源管理 → 模型服务商 | 查看 Provider 列表与引用关系 | 创建 Provider、维护实例池 / 模型 / Keys |
| 资源管理 → AI 业务集群 | 查看 Cluster 列表及所属服务商 | 创建 Cluster、配置转发策略、Key 权重 |
| 资源管理 → 模型定价 | 查看模型价格列表 | 导入 / 编辑模型价格、维护 Provider 时段模板 |
| 消费者管理 → Entity 管理 | 查看 Entity 类型与组织树 | 创建类型 / 组织、配置配额 / 限流 / 模型访问控制 |
| 消费者管理 → API Key 管理 | 查看 API Key 列表及挂载组织 | 创建 Key、重置配额、查看密钥 |
| 路由管理 → 路由表 | 查看 Global / Entity / API-Key 三级路由表 | 启用 / 禁用路由表、编辑路由规则 |
| 用户管理 | 查看控制台用户与 Token | 创建用户、创建 Token、修改密码 |

路由规则示例（JSON 视图）如下：

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

其中 `cond` 为 BFE 条件表达式，`targets` 中同一规则的权重之和必须等于 100，`fallbacks` 用于指定降级目标。路由规则编辑需先进入编辑模式，本地保存后再提交生效。

---

## 配置版本跟踪机制

壬远 AI 网关采用基于 MD5 签名与版本号的配置导出机制，详细设计参见 [第二十一章 配置导出与版本控制设计](../design/chapter14-config-export-and-version-control.md)。控制面为每类配置主题（Topic）维护当前版本号与签名，例如 `route_rule`、`ai_route`、`mod_api_key_rule` 等：

- **当前版本号**：格式为 `YYYYMMDDHHMMSS`，例如 `20260102120000`；
- **MD5 签名**：对配置内容（版本号置零后）计算得出的签名；
- **最近变更时间**：签名变化即视为一次有效配置变更。

该机制保证：若配置内容未发生变化，Conf Agent 拉取时返回 `Data: null`，避免无意义的热加载；一旦内容变化，控制面会生成新的版本号并返回全量配置，BFE 据此完成热更新。配置版本信息主要通过 InnerAPI 暴露给 Conf Agent 与运维排查工具，Dashboard 当前重点面向资源管理，版本详情可参考控制面日志或 InnerAPI 导出接口。

---

## 通过 Dashboard 进行首次配置的完整流程

以下是一个从空环境到第一次调用成功的最小配置流程，对应控制台真实导航入口。更详细的字段说明与截图可参考 `ai-gateway-web/docs/zh-cn/11-scenarios.md`。

### 步骤 1：登录并修改默认密码

使用默认账号 `admin/admin` 登录后，进入「用户管理 → 用户」，将 `admin` 密码修改为强密码，并根据团队需要新建其他管理员账号。

### 步骤 2：确认 AI 网关实例池

进入「资源管理 → AI 网关实例池」，确认已登记数据面 BFE 引擎地址。单机部署时通常已有一条默认记录；若为空，点击编辑新增一行，填写 BFE 所在机器的 IP 与端口（默认 `8080`）。

> 注意：此处登记的是数据面转发引擎地址，不是后端模型服务地址。

### 步骤 3：创建模型服务商

进入「资源管理 → 模型服务商」，点击「创建服务商」：

- 名称：Provider 唯一标识，例如 `demo-provider`；
- 实例池：后端模型服务的 IP / 端口 / 权重，例如 `172.19.1.187:13801`；
- 模型协议：如 `openai`；
- 服务鉴权 Keys：若后端需要鉴权则填写 Key 名称与明文；
- 模型列表：点击「获取」拉取可用模型并选择，例如 `doubao-pro-32k`。

创建成功后，控制面会自动根据实例池生成后端实例池和子集群。

### 步骤 4：创建 AI 业务集群

进入「资源管理 → AI 业务集群」，点击「创建集群」，按向导分步完成：

1. **基础配置**：填写集群名称、协议、是否启用会话保持；
2. **超时和重传**：通常保持默认值；
3. **被动健康检查**：通常保持默认；
4. **所属服务商与模型**：选择上一步创建的 Provider，并勾选允许转发的模型；
5. **Key 权重**：引用 Provider 中的 Key 并设置权重，权重之和须为 100。

提交后列表出现新集群即表示创建成功。

### 步骤 5：创建 Entity 组织（可选）

如需按组织统一配置配额、限流或模型访问控制，进入「消费者管理 → Entity 管理」：

- 先创建 Entity 类型（如 `team`）；
- 再创建 Entity 组织（如 `dev-team`），并绑定配额计划、限流策略与模型黑白名单。

### 步骤 6：签发 API Key

进入「消费者管理 → API Key 管理」，点击「创建」：

- 名称：API Key 标识；
- 所属组织：选择步骤 5 中创建的 Entity（可选）；
- 过期时间、允许子网、允许模型等按需填写。

创建成功后，Dashboard 会展示该 API Key 的明文，请务必妥善保存，后续无法再次查看完整密钥。

### 步骤 7：配置 API-Key 路由规则

进入「路由管理 → 路由表」，找到该 API Key 对应的 `apikey` 路由表，进入编辑模式并添加规则。例如：

```json
{
  "enabled": true,
  "rules": [
    {
      "name": "default-to-demo",
      "cond": "req_path_prefix_in(\"/\", false)",
      "targets": [
        { "cluster_name": "demo-cluster", "model": "", "weight": 100 }
      ],
      "fallbacks": []
    }
  ]
}
```

本地保存后点击「提交并生效」，未提交前不会影响线上流量。

### 步骤 8：验证配置生效

完成以上步骤后，可通过以下方式验证：

1. 触发 Conf Agent 拉取或等待其定时轮询，确认 BFE 已完成热加载；
2. 使用 curl 发送请求，验证流量是否按规则转发到目标集群：

```bash
curl -H "Authorization: <API Key>" \
  -H "Content-Type: application/json" \
  -d '{"model":"doubao-pro-32k","messages":[{"role":"user","content":"hello"}]}' \
  http://<bfe-address>:8080/v1/chat/completions
```

---

## 操作注意事项

在使用 Dashboard 进行日常运维时，应注意以下事项：

1. **默认账号安全**：`admin/admin` 仅用于首次登录，生产环境必须立即修改密码并限制访问来源。
2. **权限模型现状**：当前版本控制台用户固定为系统管理员角色；Token 分为 `System`（完整管理权限）与 `Support`（只读导出权限）。为 Conf Agent 或自动化脚本创建 Token 时，优先使用 `Support` Scope。
3. **Provider 变更的级联影响**：修改 Provider 的实例池、Keys 或模型列表会触发引用它的 Cluster 同步更新；删除 Provider 前必须确保无 Cluster 引用，否则将返回 `409 Conflict`。
4. **Cluster 删除的引用检查**：删除 Cluster 前，系统会检查其是否被 Global / Entity / API-Key 路由规则引用；若被引用，需先解除引用或修改规则。
5. **路由规则编辑模式**：路由表配置需先「进入编辑模式」，本地保存后再「提交并生效」。未提交的修改不影响线上流量，提交后才会生成新版本并触发 Conf Agent 拉取。
6. **配置生效延迟**：Dashboard 中的修改写入 MySQL 后，需经 Conf Agent 拉取并触发 BFE 热加载才能生效。不要期望配置保存后立即在数据面生效。
7. **版本号变化才代表真实变更**：即使多次点击保存，只要配置内容的 MD5 签名未变，`config_versions` 中的版本号就不会增加，数据面也不会重新加载。
8. **API-Key 明文只显示一次**：创建 API Key 时，Dashboard 通常只会在创建成功后展示一次密钥，后续无法再次查看完整明文，请妥善保管。

---

## 本章小结

本章介绍了壬远 AI 网关 Dashboard 的基础操作。主要内容包括：

- Dashboard 默认通过 `http://api-server:8183/login` 访问，初始账号为 `admin/admin`；
- 控制台导航为「左侧导航 + 右侧内容区」，覆盖资源管理、消费者管理、路由管理、用户管理四大模块；
- AI 网关实例池、模型服务商、AI 业务集群、模型定价、Entity 组织、API Key、路由表是控制台中的核心概念，理解它们的职责与引用关系是正确配置的前提；
- 控制台采用抽屉表单、列表页、编辑模式等通用交互，路由规则需先本地保存再提交生效；
- 用户、Session Key、Token 构成 Dashboard 与 API 的鉴权体系，Token 分为 `System` 与 `Support` 两种 Scope；
- 配置版本基于 MD5 签名与 `YYYYMMDDHHMMSS` 版本号实现增量同步，主要暴露给 InnerAPI 与 Conf Agent；
- 首次配置应按照“实例池 → 模型服务商 → AI 业务集群 → Entity（可选） → API Key → 路由规则 → curl 验证”的顺序进行；
- 日常运维需注意默认账号安全、级联引用检查、路由规则编辑模式、配置生效延迟与 Token 最小权限原则。

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
- `ai-gateway-web/docs/zh-cn/00-README.md`
- `ai-gateway-web/docs/zh-cn/02-overview.md`
- `ai-gateway-web/docs/zh-cn/11-scenarios.md`
- [第六章 控制面核心设计：AI Gateway API](../design/chapter06-control-plane-design.md)
- [第十一章 Provider 与 Cluster 设计](../design/chapter10-provider-and-cluster.md)
- [第二十一章 配置导出与版本控制设计](../design/chapter14-config-export-and-version-control.md)
