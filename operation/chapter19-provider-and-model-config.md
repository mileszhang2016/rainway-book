# 第十九章 Provider 与模型配置

## 本章目标

通过本章学习，读者将掌握 Provider 在壬远 AI 网关中的定位与作用，熟练使用 Dashboard 与 OpenAPI 创建和维护 Provider，正确配置模型端点、模型列表、Provider Keys 与后端实例池，使用模型发现工具自动探测模型，理解支持的模型协议差异，掌握通过 model-list.yaml 批量导入模型定价的方法，并厘清 Provider 与 Cluster 的关联关系及变更影响。

## Provider 的概念与作用

在壬远 AI 网关的控制面（Control Plane）中，**Provider** 用于描述一个下游模型提供方：它提供哪些模型、通过什么协议接入、后端实例在哪里、使用哪些 API Key。可以将其理解为“我能访问谁”这一层抽象。与之对应，**Cluster** 负责“如何把请求转发过去”，包括路由匹配、模型映射、Key 权重、重试策略、超时等。

Provider 与 Cluster 分离后，二者职责更加清晰：

- Provider = “我是谁、我能访问哪些模型、我的后端和密钥是什么”。
- Cluster = “我如何转发、用哪些模型、Key 权重如何分配”。

这种分离带来了多方面好处：同一 Provider 被多个 Cluster 引用时，实例池和密钥只需维护一份，避免重复配置；Cluster 不再存储 API Key 明文，只通过 name 引用 Provider 中的 Key，提升了安全性；Provider 可独立创建、更新、删除，Cluster 通过引用获取后端能力，生命周期更独立；新增协议时只需扩展 Provider 的 model_protocols，不会导致 Cluster 模型持续膨胀。

Provider 的核心字段包括：全局唯一的 `name`、可选的 `description`、模型发现端点 `model_endpoint`、支持的模型列表 `models`、API Key 列表 `keys`、后端实例池 `instance_pool`、支持的协议 `model_protocols`、时区 `time_zone` 与分时段模板 `tiers`。其中 `instance_pool` 必填且至少包含一个权重大于 0 的实例，`model_protocols` 必填且至少包含一个协议。

`time_zone` 默认值为 `Asia/Shanghai`，用于计算当前时间属于哪个 tier。`tiers` 初期只支持 `name=peak`，每个 tier 包含若干 `time_ranges`，采用左闭右开语义。通过 `PUT /v1/providers/{provider_name}/pricing-tiers` 可单独维护时区与 tier，无需在创建 Provider 时传入。

## 创建 Provider 的步骤

### 通过 Dashboard 创建

Dashboard 是面向运维人员的可视化控制台，创建 Provider 的常规流程如下：

1. 登录 Dashboard，进入 **Provider 管理** 页面，点击 **新建 Provider**。
2. 填写基本信息：`name` 要求全局唯一，建议使用小写字母与连字符，如 `deepseek`、`openai-official`；`description` 可选，用于团队识别用途。
3. 配置 **实例池**：填写后端地址 `addr`、端口 `port` 与权重 `weight`。同一 Provider 内 `(addr, port)` 不能重复，且至少有一个实例的 `weight > 0`。
4. 配置 **模型端点**：默认协议为 `https`，默认 URI 为 `/v1/models`。大多数 OpenAI 兼容平台无需修改；Claude 官方接口通常也使用 `/v1/models`。
5. 选择 **模型协议**：首期支持 `openai` 与 `anthropic`，聚合平台可同时勾选多种协议。
6. 添加 **Provider Keys**：每个 Key 需要一个名称 `name` 和实际密钥值 `key`。名称在 Provider 内唯一，后续 Cluster 通过该名称引用 Key。
7. 保存并提交，系统校验字段合法性，成功后返回包含 `create_time` 与 `update_time` 的完整 Provider 记录。若校验失败，Dashboard 会提示具体字段错误，例如实例池重复、协议不在枚举范围内、Key 名称不合法或 `models` 元素重复等。

### 通过 OpenAPI 创建

OpenAPI 适合自动化脚本、CI/CD 或第三方系统集成。相比 Dashboard，OpenAPI 更适合批量创建、版本化管理以及与内部平台对接。创建 Provider 的端点为：

```http
POST /v1/providers
Content-Type: application/json
```

请求体示例：

```json
{
    "name": "deepseek",
    "description": "DeepSeek 官方 API",
    "model_endpoint": { "schema": "https", "uri": "/v1/models" },
    "models": ["deepseek-chat", "deepseek-coder"],
    "keys": [
        { "name": "key-primary", "key": "sk-aaaaaaaaaaaa" },
        { "name": "key-secondary", "key": "sk-bbbbbbbbbbbb" }
    ],
    "instance_pool": [
        { "addr": "api.deepseek.com", "weight": 100, "port": 443 }
    ],
    "model_protocols": ["openai"]
}
```

若请求合法，接口返回 `ErrNum=200`，并在 `Data` 中携带完整记录。若 `model_endpoint`、`keys`、`time_zone` 未传，系统会按文档默认值填充。后续可通过 `PATCH /v1/providers/{provider_name}` 对部分字段进行更新。

## 配置模型端点、模型列表、Provider Keys

### 模型端点

`model_endpoint` 用于调用第三方平台的模型列表接口，包含 `schema` 与 `uri` 两个字段。`schema` 默认值为 `https`，可选 `http`；`uri` 默认值为 `/v1/models`，非空且必须以 `/` 开头。该端点主要供模型发现工具使用，不直接影响 BFE 转发目标地址。系统不再允许在 `model_endpoint` 中配置 `headers.Authorization`；调用模型发现接口时，认证头风格由 `model_protocols` 自动决定：`openai` 使用 `Authorization: Bearer {apikey}`，`anthropic` 使用 `x-api-key: {apikey}`。

### 模型列表

`models` 字段表示该 Provider 支持的模型名称列表，例如 `["deepseek-chat", "deepseek-coder"]`。模型名在创建时可直接填写，也可在创建后通过模型发现工具自动回填。更新 Provider 时，`models` 按全量替换处理；如果删除某个 model 时仍有 Cluster 引用该 model，系统会返回 `409 Conflict`，防止误删导致路由失效。

### Provider Keys

`keys` 字段存储 Provider 可用的 API Key 明文，每个元素包含 `name` 与 `key`。`name` 在 Provider 内唯一，长度 1-128 字符，是 Cluster 引用 Key 的纽带。Cluster 的 `llm_config.keys` 只保留 `name` 与 `weight`，例如：

```json
{
    "keys": [
        { "name": "key-primary", "weight": 70 },
        { "name": "key-secondary", "weight": 30 }
    ]
}
```

更新 Provider 的 `keys` 时，系统会校验是否有 Cluster 仍引用被删除或重命名的 Key；若存在引用，同样返回 `409 Conflict`。这一机制避免了 Key 变更导致正在运行的 Cluster 无法认证。

## 模型发现工具的使用

模型发现工具用于自动探测第三方平台当前支持的模型列表，避免人工维护模型名。端点为：

```http
POST /v1/providers/tools/discover-models
```

请求体示例：

```json
{
    "model_protocol": "openai",
    "schema": "https",
    "addr": "api.deepseek.com",
    "port": 443,
    "uri": "/v1/models",
    "apikey": "sk-aaaaaaaaaaaa"
}
```

执行时，若 `uri` 为空则默认使用 `/v1/models`；系统根据 `model_protocol` 生成对应认证头，调用 `{schema}://{addr}:{port}{uri}`，并使用对应协议解析器提取模型名列表。返回结果为一个字符串数组，可直接复制到 Provider 的 `models` 字段中。该接口为无状态工具，不直接修改 Provider；若需回填，需再调用 `PATCH /v1/providers/{provider_name}` 将 `models` 写入。

## 支持的模型协议

Provider 通过 `model_protocols` 字段声明支持的模型访问协议。首期枚举值包括：

| 枚举值 | 说明 |
|--------|------|
| `openai` | OpenAI 兼容协议，包括大多数国产兼容平台 |
| `anthropic` | Anthropic Claude Messages API |

一个 Provider 可同时支持多种协议，例如聚合平台可配置 `["openai", "anthropic"]`，但至少包含一个协议。

`model_protocols` 会影响 BFE 数据面的转发行为。认证头注入方面，`openai` 使用 `Authorization: Bearer`，`anthropic` 使用 `x-api-key`；Claude 请求还需要额外注入 `anthropic-version`。Usage 解析会按协议风格处理不同响应格式，例如 OpenAI 风格的 `usage` 字段与 Claude 风格的 `usage` 字段结构不同。协议匹配校验会检查请求协议风格是否在目标 Cluster 对应 Provider 的 `model_protocols` 中，若不匹配则直接拒绝。控制面生成 BFE 配置时，会将 `provider.model_protocols` 透传到 `AIConf.ModelProtocols`，供数据面使用。

## 模型定价导入

模型定价用于成本核算与配额扣减，存储在 `/model-prices` 资源中。为便于批量维护，系统支持通过 `model-list.yaml` 文件整表导入。接口为：

```http
POST /v1/model-prices/import
Content-Type: multipart/form-data
```

表单参数包括 `file`（YAML 文件，必填）与 `mode`（导入模式，可选 `replace` 默认全量替换，或 `merge` 增量合并）。

`model-list.yaml` 示例：

```yaml
version: v1.0
default_currency: "RMB"

models:
  - provider: "deepseek"
    model: "deepseek-v3"
    base_model: "deepseek-v3"
    mode: "chat"
    capabilities: ["chat", "reasoning", "tools"]
    supported_parameters: ["temperature", "max_tokens"]
    limits:
      context_window: 128000
      max_input_tokens: 128000
      max_output_tokens: 8192
    prices:
      input_cost_per_token: 0.000002
      output_cost_per_token: 0.000008
    tier_prices:
      peak:
        input_cost_per_token: 0.000004
        output_cost_per_token: 0.000016
    metadata:
      source: "https://platform.deepseek.com/pricing"
      notes: "DeepSeek V3"
```

导入时系统会校验版本、币种、`provider/model/mode` 唯一性、必填字段与价格非负性等。`prices` 至少包含一个价格字段，所有价格字段必须为非负数，支持 8 位及以上小数精度。若记录包含 `tier_prices`，初期 tier name 只支持 `peak`。

`replace` 模式先清空 `model_prices` 表再写入新数据，适合全量刷新官方价目表；`merge` 模式对已有 `(provider, model, mode)` 记录更新、新增记录插入，适合增量补丁。导入接口仅允许管理员调用。

> 说明：`/model-prices` 的 `provider` 字段仅作为价格归集标识，不再强制引用已存在的 `/providers`。因此可以先导入价格，再创建 Provider，二者独立维护。

## Provider 与 Cluster 的关联

Provider 与 Cluster 通过 `cluster.llm_config.provider` 建立强引用关系。推荐配置顺序为：

```text
/providers → /model-prices → /clusters → 路由规则
```

一个 Cluster 只能引用一个 Provider，但一个 Provider 可被多个 Cluster 引用。Cluster 的 `llm_config` 示例：

```json
{
    "name": "my-cluster",
    "llm_config": {
        "provider": "deepseek",
        "models": ["deepseek-chat", "deepseek-coder"],
        "keys": [
            { "name": "key-primary", "weight": 70 },
            { "name": "key-secondary", "weight": 30 }
        ],
        "key_policy": { "strategy": "weighted_random", "max_retries": 3 },
        "match_prefix": "deepseek/",
        "strip_prefix": true
    }
}
```

关键约束包括：`llm_config.provider` 必填且必须引用已存在的 Provider；`llm_config.models` 必须是 Provider `models` 的子集；`llm_config.keys` 中的 `name` 必须对应 Provider 中存在的 Key。Provider 的 `instance_pool` 变更会自动同步到引用它的所有 Cluster，实现一处修改、全局生效。删除 Provider 前，必须确保没有 Cluster 引用，否则返回 `409 Conflict`。

BFE 最终接收到的配置由控制面自动合并生成：`AIConf.Keys` 通过 `name` join Provider 的 Key 明文与 Cluster 的 Key 权重；`AIConf.ModelProtocols` 透传 Provider 的协议列表；`AIConf.ModelTable` 由控制面根据 Provider 查询 `model-prices` 自动填充。因此 Cluster 无需关心密钥明文与价格数据，只需维护转发策略。

## 常见问题与排查

### 1. 创建 Provider 时报“instance_pool 不合法”

检查是否至少填写了一个实例；`(addr, port)` 是否重复；是否至少有一个实例的 `weight > 0`。

### 2. 更新 Provider 的 Keys 或 Models 时返回 409

说明当前仍有 Cluster 引用被删除/重命名的 Key，或被移除的 Model。解决步骤：先修改对应 Cluster 的 `llm_config.keys` 或 `models`，解除引用后再更新 Provider。

### 3. 删除 Provider 时返回 409

该 Provider 仍被至少一个 Cluster 引用。需要先删除或修改引用它的 Cluster，再删除 Provider。

### 4. 模型发现返回空列表或报错

检查 `model_protocol` 是否与实际平台匹配；`addr`、`port`、`uri` 是否正确；`apikey` 是否有效。对于 Claude 平台，确认 `model_protocol` 选择 `anthropic`。

### 5. Cluster 引用 Provider 后转发失败

检查 Cluster 的 `llm_config.models` 是否为 Provider `models` 的子集；`llm_config.keys` 中的 `name` 是否存在于 Provider 的 `keys` 中；请求协议风格是否在 Provider 的 `model_protocols` 中。

### 6. 模型价格未生效

检查 `/model-prices` 中是否存在对应的 `(provider, model, mode)` 记录；`model-list.yaml` 导入是否成功，关注 `errors` 列表；`price_currency` 是否为 `RMB`； prices 与 tier_prices 中价格字段是否为非负数。

## 配置示例

### 完整 Provider JSON 配置

```json
{
    "name": "anthropic",
    "description": "Anthropic Claude 官方 API",
    "model_endpoint": { "schema": "https", "uri": "/v1/models" },
    "models": ["claude-3-5-sonnet-20241022", "claude-3-opus-20240229"],
    "keys": [
        { "name": "key-prod", "key": "sk-ant-api03-xxxxxxxx" }
    ],
    "instance_pool": [
        { "addr": "api.anthropic.com", "weight": 100, "port": 443 }
    ],
    "model_protocols": ["anthropic"],
    "time_zone": "America/New_York",
    "tiers": [
        {
            "name": "peak",
            "time_ranges": [
                { "weekdays": [1, 2, 3, 4, 5], "start": "09:00", "end": "18:00" }
            ]
        }
    ]
}
```

### 模型发现请求示例

```bash
curl -X POST https://control-plane.example.com/v1/providers/tools/discover-models \
  -H "Content-Type: application/json" \
  -d '{
    "model_protocol": "openai",
    "schema": "https",
    "addr": "api.deepseek.com",
    "port": 443,
    "apikey": "sk-aaaaaaaaaaaa"
  }'
```

### Cluster TOML 语义示意

```toml
[cluster.my-cluster.llm_config]
provider = "deepseek"
models = ["deepseek-chat", "deepseek-coder"]
match_prefix = "deepseek/"
strip_prefix = true

[[cluster.my-cluster.llm_config.keys]]
name = "key-primary"
weight = 70

[[cluster.my-cluster.llm_config.keys]]
name = "key-secondary"
weight = 30

[cluster.my-cluster.llm_config.key_policy]
strategy = "weighted_random"
max_retries = 3
```

Cluster 不声明 `instance_pool`、`model_endpoint` 或 `provider_type`，这些信息全部来自 Provider。

## 本章小结

Provider 是壬远 AI 网关控制面中描述下游模型提供方的核心资源。Provider 与 Cluster 职责分离后，Cluster 专注转发策略，Provider 专注接入信息，提升了配置复用性、安全性与可维护性。

本章重点包括：Provider 的数据模型与字段含义；通过 Dashboard 与 OpenAPI 创建、更新 Provider 的流程；模型端点、模型列表、Provider Keys 的配置方法与约束；`/providers/tools/discover-models` 无状态模型发现工具的使用；`openai` 与 `anthropic` 协议对认证头、版本头、Usage 解析与协议匹配的影响；通过 `model-list.yaml` 批量导入模型定价的流程与注意事项；Provider 与 Cluster 的强引用关系以及变更时的同步与冲突处理；常见问题的排查思路与配置示例。

合理规划 Provider 与 Cluster 的拆分，是后续路由规则、API-Key 配额、限流策略生效的重要前提。建议在生产环境中先统一维护 Provider 与模型价格，再按需创建不同业务线的 Cluster。定期对比 `/providers` 与 `/model-prices/actions/get-providers` 返回的 provider 列表，可及时发现并补录价格记录与实际 Provider 脱节的问题，确保成本核算准确。

## 参考文档

- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/providers.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/model-prices.md`
- `ai-gateway-api/design-docs/sys-design/details/provider与cluster概念分离.md`
- `ai-gateway-api/design-docs/sys-design/details/Claude协议转发支持.md`
