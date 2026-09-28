# 第十九章 控制台基础操作

## 本章目标

通过本章，读者将掌握壬远 AI 网关（Rainway AI Gateway）Dashboard 的访问方式、界面组织与基础操作流程，理解控制台背后的核心概念（模型服务商、AI 业务集群、Entity、API Key、路由表等），并能够独立完成首次配置。具体包括：

- 如何登录 Dashboard 并修改默认账号；
- 控制台各导航入口的职责与数据对应关系；
- 用户、Token 与权限 Scope 的管理方式；
- 控制台通用交互约定（抽屉表单、编辑模式、通知）；
- 配置版本跟踪机制；
- 数据报表（总览 + 明细页签）与操作日志（写操作审计）的查询方式；
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

登录成功后，控制台呈现「左侧导航 + 右侧内容区」的布局。导航由 `/meta` 接口动态返回，对应 `conf/nav_tree.toml` 中定义的导航树。当前管理员视角下的主导航包括资源管理、消费者管理、路由管理、用户管理、数据报表与操作日志等模块：

```
AI 网关
├─ 资源管理
│   ├─ 模型服务商        实例池、协议、模型与 Keys
│   ├─ AI 业务集群      引用服务商，配置转发策略
│   ├─ 模型定价         模型价格维护与费用核算
│   └─ EPP调度          EPP 实例池与集群调度分配
├─ 消费者管理
│   ├─ Entity 管理     类型 / 组织（配额、限流、模型访问控制）
│   └─ API Key 管理    调用凭证签发与治理
├─ 路由管理
│   └─ 路由表          Global / Entity / API-Key 路由规则
├─ 用户管理            控制台账号与 Token（管理员）
├─ 数据报表            运营数据可视化：总览指标、图表与日志明细
└─ 操作日志            写操作审计查询（管理员）
```

每个导航项对应一组 OpenAPI 资源：

| 导航项 | 对应 OpenAPI 端点 | 主要职责 |
|--------|-------------------|----------|
| 模型服务商 | `/providers` | 维护模型提供方、后端实例池、API-Key 明文与模型协议 |
| AI 业务集群 | `/clusters` | 维护转发集群、LLM 配置、Key 权重与路由策略 |
| 模型定价 | `/model-prices` | 维护模型在不同 Provider 与时段下的价格，支持分段计价（peak 忙时） |
| EPP调度 | `/epp-pool`、`/epp-assignments` | 维护 EPP 实例池及 EPP 模式集群的调度分配 |
| Entity 管理 | `/entity-types`、`/entities` | 维护组织类型、组织架构、模型黑白名单、配额与限流策略 |
| API Key 管理 | `/api-keys` | 创建、启用/禁用、配额绑定与密钥查看 |
| 路由表 | `/global-route-rules`、`/route-tables` 等 | 维护 global / entity / api-key 三级路由规则 |
| 数据报表 | `/report/overview`、`/report/timeseries`、`/report/rankings`、`/report/distribution`、`/report/logs` | 运营数据可视化查询，界面分总览、明细两个页签（配置 `[Report]` 后可用，未配置时无数据） |
| 用户管理 | `/auth/users`、`/auth/tokens` | 维护控制台用户与机器 Token |
| 操作日志 | `/operation-logs` | 写操作只读审计查询，含变更摘要与请求上下文 |

### 通用交互约定

- **列表页**：搜索区 + 操作按钮 + 表格 + 分页；多数列表支持「20 条/页」切换（模型定价默认 50 条/页，支持 20–1000 条/页）。
- **抽屉表单**：创建 / 编辑在右侧抽屉完成，底部「提交 / 取消」（向导类为「下一步 / 上一步」）。
- **编辑模式**：路由规则等高风险配置需先「进入编辑模式」，改完「本地保存」，再点「提交并生效」（提交后自动退出编辑模式），未提交的修改不影响线上流量。
- **通知**：操作结果以右上角通知呈现；错误通知（如「数据重复」）不自动消失，点 × 关闭。

---

## 控制台中的核心概念

控制台中的诸多操作都围绕以下概念展开：Provider（模型服务商）、Cluster（AI 业务集群）、Entity（组织）、API-Key、路由表、配额计划与限流策略。

这些概念的完整定义、相互关系与设计动机已在 [第五章 壬远AI网关架构与核心概念](../design/chapter05-system-architecture.md#核心概念) 中统一介绍。本节仅说明它们在 Dashboard 中的对应入口：

| 控制台导航项 | 对应概念 | 说明 |
|---|---|---|
| 资源管理 → 模型服务商 | Provider | 维护模型提供方、后端实例池、模型协议与认证密钥 |
| 资源管理 → AI 业务集群 | Cluster | 引用 Provider，配置转发策略、Key 权重与超时 |
| 资源管理 → 模型定价 | Model Price | 维护模型在不同 Provider 与时段（peak 忙时）下的单价，支持分段计价 |
| 消费者管理 → Entity 管理 | Entity | 维护组织架构、配额、限流与模型访问控制 |
| 消费者管理 → API Key 管理 | API-Key | 签发调用凭证并绑定到 Entity |
| 路由管理 → 路由表 | Route Table | 维护 Global / Entity / API-Key 三级路由规则 |

上述概念对应数据面请求处理顺序（鉴权 → 模型权限 → 限流 → 配额 → 路由 → 转发，详见 [第五章 壬远AI网关架构与核心概念](../design/chapter05-system-architecture.md)）中的不同环节：API-Key 与 Entity 承载鉴权、模型访问控制、限流与配额策略，路由表承载路由匹配规则，Provider 与 Cluster 决定最终的转发目标。

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
| 资源管理 → 模型服务商 | 查看 Provider 列表与引用关系 | 创建 Provider、维护实例池 / 模型 / Keys |
| 资源管理 → AI 业务集群 | 查看 Cluster 列表及所属服务商 | 创建 Cluster、配置转发策略、Key 权重 |
| 资源管理 → 模型定价 | 查看模型价格列表 | 导入 / 编辑模型价格、维护分段计价（peak 忙时时段）模板 |
| 资源管理 → EPP调度 | 查看 EPP 实例池与集群分配关系 | 维护 EPP 实例池、查看 / 覆写 EPP 模式集群的调度分配 |
| 消费者管理 → Entity 管理 | 查看 Entity 类型与组织树 | 创建类型 / 组织、配置配额 / 限流 / 模型访问控制 |
| 消费者管理 → API Key 管理 | 查看 API Key 列表及挂载组织 | 创建 Key、重置配额、查看密钥 |
| 路由管理 → 路由表 | 查看 Global / Entity / API-Key 三级路由表 | 启用 / 禁用路由表、编辑路由规则 |
| 数据报表 | 总览页签：5 项指标卡 + 6 张图表；明细页签：服务端分页日志表格（配置 `[Report]` 后展示，未配置时菜单不出现） | 按时间窗与维度过滤报表 |
| 操作日志 | 按时间范围与条件筛选写操作审计记录 | 查看详情抽屉（变更摘要 before / after） |
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

壬远 AI 网关采用基于 MD5 签名与版本号的配置导出机制，详细设计参见 [第十四章 配置导出与版本控制设计](../design/chapter14-config-export-and-version-control.md)。控制面为每类配置主题（Topic）维护当前版本号与签名，例如 `route_rule`、`ai_route`、`mod_api_key_rule` 等：

- **当前版本号**：格式为 `YYYYMMDDHHMMSS`，例如 `20260102120000`；
- **MD5 签名**：对配置内容（版本号置零后）计算得出的签名；
- **最近变更时间**：签名变化即视为一次有效配置变更。

该机制保证：若配置内容未发生变化，Conf Agent 拉取时返回 `Data: null`，避免无意义的热加载；一旦内容变化，控制面会生成新的版本号并返回全量配置，BFE 据此完成热更新。配置版本信息主要通过 InnerAPI 暴露给 Conf Agent 与运维排查工具，Dashboard 当前重点面向资源管理，版本详情可参考控制面日志或 InnerAPI 导出接口。

---

## 通过 Dashboard 进行首次配置的完整流程

以下是一个从空环境到第一次调用成功的最小配置流程，对应控制台真实导航入口。更详细的字段说明与示例取值可参考 Dashboard 用户手册 `ai-gateway-web/docs/zh-cn/13-scenarios.md`（场景实战）。

### 步骤 1：登录并修改默认密码

使用默认账号 `admin/admin` 登录后，进入「用户管理 → 用户」，将 `admin` 密码修改为强密码，并根据团队需要新建其他管理员账号。

### 步骤 2：创建模型服务商

进入「资源管理 → 模型服务商」，点击「创建服务商」：

- 名称：Provider 唯一标识，例如 `demo-provider`；
- 实例池：后端模型服务的 IP / 端口 / 权重（单实例权重填 100），例如 `172.19.1.187:13801`；
- 模型协议：如 `openai`；
- 服务鉴权 Keys：若后端需要鉴权则填写 Key 名称与明文；
- 模型列表：点击「获取」拉取可用模型并选择，例如 `doubao-pro-32k`。

创建成功后，控制面会自动根据实例池生成后端实例池和子集群。

### 步骤 3：创建 AI 业务集群

进入「资源管理 → AI 业务集群」，点击「创建集群」，按向导分步完成：

1. **基础配置**：填写集群名称（如 `demo-cluster`）、协议（如 `https`）、是否启用会话保持；
2. **超时和重传**：通常保持推荐默认值；
3. **被动健康检查**：通常保持默认；
4. **大模型配置**：选择上一步创建的 Provider（如 `demo-provider`），勾选允许转发的模型（如 `doubao-pro-32k`），并按需引用服务商 Key 设置权重；
5. **复查与检查**：核对汇总信息后点「提交」，集群列表出现新集群即表示创建成功。

### 步骤 4：创建 Entity 组织（可选）

如需按组织统一配置配额、限流或模型访问控制，进入「消费者管理 → Entity 管理」：

- 先创建 Entity 类型（如 `team`，级别 3）；
- 再创建 Entity 组织（如 `dev-team`），允许模型保持 `*` 即可，并按需绑定配额计划、限流策略与模型黑白名单。

### 步骤 5：签发 API Key

进入「消费者管理 → API Key 管理」，点击「创建」：

- 描述：API Key 标识（如 `demo 第一个 Key`）；
- 挂载 Entity：选择步骤 4 中创建的 Entity（可选，跳过步骤 4 则留空）；
- 过期时间：可勾选永不过期，或按需指定；
- 允许子网：默认 `*` 不限制；
- 允许模型等按需填写。

创建成功后，列表中该 API Key 的 Key 值以脱敏形式展示（前 8 位 + `****` + 后 4 位），点击 Key 值可查看完整明文并一键复制，请妥善保存。注意同时记录该 Key 的 Key ID（如 `api-key-1`），下一步配置路由时需按 Key ID 找到对应的 apikey 路由表。Key 值创建后不可修改。

### 步骤 6：配置 API-Key 路由规则

进入「路由管理 → 路由表」，找到属主为步骤 5 中 Key ID 的 `apikey` 路由表，进入编辑模式并添加规则。例如：

```json
{
  "enabled": true,
  "rules": [
    {
      "name": "demo-default",
      "cond": "req_path_prefix_in(\"/\", false)",
      "targets": [
        { "cluster_name": "demo-cluster", "model": "doubao-pro-32k", "weight": 100 }
      ],
      "fallbacks": []
    }
  ]
}
```

本地保存后点击「提交并生效」，未提交前不会影响线上流量。

### 步骤 7：验证配置生效

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

## 报表与用量分析

控制台内置数据报表模块，面向管理员提供运营数据的统一可视化视图：总览指标卡、时序折线图、维度排行、占比分布与日志明细，无需额外部署 Grafana。报表数据来自 `/open-api/v1/report/*` 接口，前提是已按「第十八章 安装部署」完成报表形态配置：轻量形态（`[Report].Backend = "mysql"` + log-reader `mod_log_mysql` 插件）或标准形态（`Backend = "doris"` + Doris/Grafana 链路）。配置 `[Report]` 后控制台展示数据报表菜单；未配置时报表模块不装配，菜单不出现，接口返回 404。

### 页面结构

报表页面分为**全局过滤栏**与**页签区域**两部分：

- **全局过滤栏**：提供时间范围、路由模型、API Key、提供商、主机、流式、状态码等过滤参数（均可选、可组合），并内置快捷时间按钮（1 小时 / 6 小时 / 24 小时 / 7 天）。点击「查询」重新加载当前页签数据，「重置」恢复全部过滤参数为默认值；切换页签后过滤条件保持。
- **页签**：分为**总览**与**明细**两个页签。

### 总览页签

总览页签展示选定时间窗口内的 **5 项指标卡**与 **6 张时序图表**（2 行 × 3 列网格布局），覆盖请求量、错误、Token、延迟、TTFT/TPOT、成本等维度。时序数据的采样间隔由服务端按窗口自动计算，客户端无需传参；成本数据按币种分开展示，总览卡主值取首个币种（金额 + 币种小字），其余币种以副行列示，无成本数据时显示「-」。

> 延迟分位数仅 Doris 后端返回；MySQL 轻量形态无原生分位数，对应总览指标不展示。

### 明细页签

明细页签展示**服务端分页的日志表格**：

- 默认每页 **20** 条，最大 **100** 条，按时间倒序排列（最新请求优先）；
- 支持行展开，查看完整字段与 JSON 数据；
- 页签内另有独立过滤条件（请求模型、只看错误、错误消息关键字），仅影响日志表格查询。

### 接口层视图

报表页签背后的接口仍按以下五类视图组织，便于自动化与二次开发：

| 视图 | 内容 | 对应接口 |
|------|------|----------|
| 总览 | 请求总量、错误率、Token 合计、平均/分位延迟、TTFT/TPOT、成本（按币种）、限流命中与鉴权拒绝 | `GET /report/overview` |
| 时序趋势 | QPS、Token 吞吐、延迟、TTFT/TPOT、成本增速的桶化曲线，时间桶随窗口自适应（≤6h 为 1 分钟桶） | `GET /report/timeseries` |
| 排行 | 模型、提供商、API Key、主机、状态码等维度的 TopN 请求量排行 | `GET /report/rankings` |
| 分布 | 状态码、协议、模式、流式占比的饼图 | `GET /report/distribution` |
| 日志明细 | 单条请求明细（模型、Token、耗时、成本、路由标签等），支持只看错误与错误关键字模糊搜索，按时间倒序分页 | `GET /report/logs` |

使用要点：

- **时间窗**：所有视图共用时间选择器，窗口最长 7 天（与明细保留期对齐），超过 7 天请在过滤栏分段查询。
- **过滤**：可按模型、API Key、提供商、主机、状态码、是否流式组合过滤；日志明细还支持 `requested_models`（请求模型）、只看错误（`err_only`）与错误消息关键字。
- **权限**：报表为管理员功能，需要 `FeatureReport + ActionReadAll` 权限（当前控制台用户均为 System scope，满足要求）。
- **形态差异**：MySQL 轻量形态不提供 P50/P90/P99 分位数延迟，对应卡片自动隐藏；Doris 标准形态展示完整分位数。如有分位数需求，建议升级到 Doris 后端。
- **口径一致**：报表指标与既有 Grafana Dashboard 面板同源，同一数据源下两套视图可对账。

## 操作日志与审计

操作日志模块（侧栏「操作日志」）提供平台写操作的**只读审计查询**。日志由系统在配置变更时自动写入，控制台不提供手工录入入口；当前仅系统管理员可访问。

### 列表页

页面分为**时间筛选栏**与**分页表格**两部分：

- **时间范围筛选**：设置起始时间、结束时间后点击「查询」，按闭区间过滤日志；未设置时间时展示全部（受分页限制）。
- **表格列**：操作人（模糊搜索，执行操作的控制台用户或 Token 名称）、操作动作（下拉筛选，`create` / `update` / `delete` / `reset` / `import` / `bind` / `unbind` 等，以 Tag 展示）、资源类型（下拉筛选）、资源名称（模糊搜索）、结果（成功 / 失败）、操作时间（格式 `YYYY-MM-DD HH:mm:ss`），操作列为「详情」按钮。
- **分页**：本模块采用服务端分页（默认每页 20 条，可切换页码与每页条数），与部分模块的前端分页不同。

### 详情抽屉

点击行操作「详情」，右侧滑出详情抽屉，展示操作者类型（用户 / Token）、操作人、操作动作、资源类型、资源 ID、资源名称、父级资源 ID、结果、错误信息（仅失败时展示）、变更摘要、请求路径与请求方式、客户端 IP、User-Agent、操作时间。其中**变更摘要**可展开查看 `before` / `after` JSON 对比与差异字段，敏感字段已脱敏。

### 日志来源与权限

以下写操作成功或失败后均会自动产生操作日志：Entity / Entity 类型、API-Key、模型服务商、业务集群、路由表、证书、配额计划、模型定价、控制台用户与 Token。典型用法是在其他模块完成配置变更后，回到操作日志按资源名称或时间范围检索，核对变更内容与操作者。

无读权限时接口返回 `Feature Access Deny`（402），列表无法加载；若看不到「操作日志」菜单，请联系管理员确认账号角色。

## 操作注意事项

在使用 Dashboard 进行日常运维时，应注意以下事项：

1. **默认账号安全**：`admin/admin` 仅用于首次登录，生产环境必须立即修改密码并限制访问来源。
2. **权限模型现状**：当前版本控制台用户固定为系统管理员角色；Token 分为 `System`（完整管理权限）与 `Support`（只读导出权限）。为 Conf Agent 或自动化脚本创建 Token 时，优先使用 `Support` Scope。
3. **Provider 变更的级联影响**：修改 Provider 的实例池、Keys 或模型列表会触发引用它的 Cluster 同步更新；删除 Provider 前必须确保无 Cluster 引用，否则将返回 `409 Conflict`。
4. **Cluster 删除的引用检查**：删除 Cluster 前，系统会检查其是否被 Global / Entity / API-Key 路由规则引用；若被引用，需先解除引用或修改规则。
5. **路由规则编辑模式**：路由表配置需先「进入编辑模式」，本地保存后再「提交并生效」。未提交的修改不影响线上流量，提交后才会生成新版本并触发 Conf Agent 拉取。
6. **配置生效延迟**：Dashboard 中的修改写入 MySQL 后，需经 Conf Agent 拉取并触发 BFE 热加载才能生效。不要期望配置保存后立即在数据面生效。
7. **版本号变化才代表真实变更**：即使多次点击保存，只要配置内容的 MD5 签名未变，`config_versions` 中的版本号就不会增加，数据面也不会重新加载。
8. **API-Key 明文脱敏展示**：API Key 列表中 Key 值脱敏显示（前 8 位 + `****` + 后 4 位），点击可查看完整值并复制；但 Key 值创建后不可修改，请创建成功后立即妥善保管。
9. **操作日志只读审计**：操作日志由系统在写操作时自动记录，仅支持查询，不提供修改与删除入口；如需核对历史变更，可按资源名称或时间范围在操作日志中检索。

---

## 本章小结

本章介绍了壬远 AI 网关 Dashboard 的基础操作。主要内容包括：

- Dashboard 默认通过 `http://api-server:8183/login` 访问，初始账号为 `admin/admin`；
- 控制台导航为「左侧导航 + 右侧内容区」，覆盖资源管理（模型服务商、AI 业务集群、模型定价、EPP调度）、消费者管理、路由管理、用户管理、数据报表与操作日志等模块；
- 模型服务商、AI 业务集群、模型定价、Entity 组织、API Key、路由表是控制台中的核心概念，理解它们的职责与引用关系是正确配置的前提；
- 控制台采用抽屉表单、列表页、编辑模式等通用交互，路由规则需先本地保存再提交生效；
- 用户、Session Key、Token 构成 Dashboard 与 API 的鉴权体系，Token 分为 `System` 与 `Support` 两种 Scope；
- 配置版本基于 MD5 签名与 `YYYYMMDDHHMMSS` 版本号实现增量同步，主要暴露给 InnerAPI 与 Conf Agent；
- 首次配置应按照“模型服务商 → AI 业务集群 → Entity（可选） → API Key → 路由规则 → curl 验证”的顺序进行；
- 数据报表模块提供总览（5 项指标卡 + 6 张图表）与明细（服务端分页日志表格）两个页签，覆盖用量、延迟、成本与排障场景，配置 `[Report]` 并部署报表形态后方可使用，MySQL 轻量形态不提供延迟分位数；
- 操作日志对平台写操作进行只读审计，支持时间范围筛选与含变更摘要（before / after JSON，敏感字段脱敏）的详情抽屉，便于核对历史变更；
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
- `ai-gateway-web/docs/zh-cn/01-login-and-user.md`
- `ai-gateway-web/docs/zh-cn/02-overview.md`
- `ai-gateway-web/docs/zh-cn/11-operation-logs.md`
- `ai-gateway-web/docs/zh-cn/12-report.md`
- `ai-gateway-web/docs/zh-cn/13-scenarios.md`
- `ai-gateway-web/docs/zh-cn/14-appendix.md`
- [第六章 控制面核心设计：AI Gateway API](../design/chapter06-control-plane-design.md)
- [第十一章 Provider 与 Cluster 设计](../design/chapter10-provider-and-cluster.md)
- [第十四章 配置导出与版本控制设计](../design/chapter14-config-export-and-version-control.md)
