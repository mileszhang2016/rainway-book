# 第十三章 模型定价与成本核算设计

## 本章目标

通过本章，读者将理解：

- 模型定价数据在 AI 网关成本核算、配额扣减和成本分摊中的作用；
- `ModelPrice` 数据模型的字段含义与校验规则；
- `model-list.yaml` 批量导入格式及其使用方式；
- RMB 配额分时段定价（Tiered Pricing）的设计思路与实现机制；
- BFE 数据面如何基于定价数据完成运行时成本计算；
- 模型定价与 Provider、Cluster 概念分离后的关系与配置归属。

---

## 模型定价数据的作用

在壬远AI网关中，模型定价数据（Model Price）是连接"资源使用"与"成本核算"的桥梁。当业务系统通过 API-Key 调用模型服务时，BFE 数据面会解析请求与响应中的 Token 用量，并依据控制面下发的定价数据计算本次调用产生的成本。

模型定价数据主要服务于以下场景：

- **配额扣减**：API-Key 或 Entity 绑定的 RMB 配额，需要按实际调用成本精确扣减。如果定价缺失或不准确，会导致配额消耗与实际成本偏离，影响预算控制；
- **成本核算**：为运营者提供按模型、按 Provider、按 API-Key 乃至按 Entity 的调用成本统计，支撑成本洞察与优化决策；
- **成本分摊**：在多租户或多业务共享网关时，按实际消耗价格进行成本分摊与账单生成，实现内部结算；
- **高峰/闲时定价**：部分模型提供商采用分时段报价，网关需要按请求发生时刻匹配不同价格，从而真实反映供应商计费规则；
- **缓存计费**：支持对提示缓存（prompt caching）命中与未命中的 Token 分别计价，鼓励缓存复用并准确反映成本结构。

模型定价数据由 AI Gateway API 控制面统一维护，通过 InnerAPI 导出后下发给 BFE。BFE 在转发请求时完成实时成本计算，并将结果用于配额扣减和日志审计。控制面与数据面的价格版本保持一致，避免了多源数据带来的核算差异。

---

## ModelPrice 数据模型

`ModelPrice` 是控制面维护的单条模型定价记录，主键为 `(provider, model, mode)` 三元组。OpenAPI 路径 `/model-prices` 提供了完整的 CRUD 与批量导入能力。

### 核心字段

```json
{
  "id": 1,
  "provider": "deepseek",
  "model": "deepseek-v3",
  "base_model": "deepseek-v3",
  "mode": "chat",
  "capabilities": ["chat", "reasoning", "tools"],
  "supported_parameters": ["temperature", "max_tokens"],
  "limits": {
    "context_window": 128000,
    "max_input_tokens": 128000,
    "max_output_tokens": 8192
  },
  "prices": {
    "input_cost_per_token": 0.000002,
    "output_cost_per_token": 0.000008,
    "cache_read_input_token_cost": 0.0000005
  },
  "tier_prices": {
    "peak": {
      "input_cost_per_token": 0.000004,
      "output_cost_per_token": 0.000016,
      "cache_read_input_token_cost": 0.000001
    }
  },
  "price_currency": "RMB",
  "metadata": {
    "source": "https://platform.deepseek.com/pricing",
    "notes": "DeepSeek V3"
  },
  "create_time": 1716883200,
  "update_time": 1716883200
}
```

各字段说明如下：

| 字段 | 说明 |
|------|------|
| `provider` | Provider / Cluster 标识，仅作为价格归集标识，不强制要求在 `/providers` 中存在 |
| `model` | 模型名，如 `deepseek-v3` |
| `base_model` | 归一化模型名，用于模型映射后的价格归一 |
| `mode` | 请求模式，如 `chat`、`completion`、`embedding`、`image_generation` 等 |
| `capabilities` | 模型支持的能力列表，如 `vision`、`reasoning`、`tools` |
| `supported_parameters` | 支持的请求参数列表，如 `temperature`、`max_tokens` |
| `limits` | 限制对象，包括上下文窗口、最大输入/输出 Token 数等 |
| `prices` | 默认价格表，未命中 tier 时作为 fallback |
| `tier_prices` | 分时段价格表，键为 tier name（初期只支持 `peak`） |
| `price_currency` | 价格货币，当前固定为 `RMB` |
| `metadata` | 元数据，包括价格来源、备注等 |

### 价格字段枚举

`prices` 与 `tier_prices.<tier>` 中可使用的键名包括：

| 键名 | 说明 |
|------|------|
| `input_cost_per_token` | 每 Token 输入成本 |
| `output_cost_per_token` | 每 Token 输出成本 |
| `cache_read_input_token_cost` | 缓存读取输入 Token 成本 |
| `cache_creation_input_token_cost` | 缓存创建输入 Token 成本 |
| `input_cost_per_token_above_200k_tokens` | 超过 200k Token 的输入成本 |
| `output_cost_per_token_above_200k_tokens` | 超过 200k Token 的输出成本 |
| `output_cost_per_image` | 每张输出图像成本 |
| `output_cost_per_pixel` | 每像素输出成本 |
| `input_cost_per_audio_per_second` | 每秒音频输入成本 |
| `input_cost_per_video_per_second` | 每秒视频输入成本 |
| `output_cost_per_second` | 每秒输出成本 |
| `ocr_cost_per_page` | 每页 OCR 成本 |
| `output_cost_per_character` | 每字符输出成本 |

当前版本主要使用按 Token 计费的价格项，其余字段为后续多模态计费预留。

### 价格精度

`prices` 与 `tier_prices` 中的价格字段为浮点数，业务上支持 8 位及更多小数精度，例如 `0.0000015`、`0.00000075`。为避免默认 JSON encoder 将极小数值输出为科学计数法（如 `1.5e-6`），AI Gateway API 与 BFE 两侧对 `PriceMap` / `TierPriceMap` 均实现了自定义 `MarshalJSON`，强制使用十进制表示法。该表示方式仅影响配置文本的可读性，不改变 `float64` 数值语义，也不影响 BFE 内部的定点整数扣减逻辑。

---

## model-list.yaml 导入格式

当需要批量维护大量模型定价时，可通过 `/v1/model-prices/import` 接口导入 `model-list.yaml` 文件。该文件是 `model_prices` 表的权威数据源之一。

### 顶层结构

```yaml
version: v1.0                    # 格式版本，必填
default_currency: "RMB"          # 全局默认币种，当前仅支持 RMB
models:                          # 模型列表，必填
  - ...
```

### 单条记录结构

```yaml
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
      cache_read_input_token_cost: 0.0000005
    tier_prices:
      peak:
        input_cost_per_token: 0.000004
        output_cost_per_token: 0.000016
        cache_read_input_token_cost: 0.000001
    metadata:
      source: "https://platform.deepseek.com/pricing"
      notes: "DeepSeek V3"
```

### 导入模式

`/v1/model-prices/import` 支持两种导入模式：

- **replace**（默认）：先清空 `model_prices` 表，再写入新数据，适用于全量更新；
- **merge**：对已有 `(provider, model, mode)` 记录更新，新增记录插入，适用于增量维护。

导入接口仅允许管理员调用。导入过程中会校验必填字段、`prices` 非负、`tier_prices` 键名合法性、`limits` 非负整数以及三元组唯一性。

---

## RMB 配额分时段定价机制

随着 DeepSeek 等模型提供商采用"高峰 / 空闲"分时段定价策略，RMB 配额扣减需要具备按请求发生时刻匹配不同价格的能力。

### 核心概念

| 概念 | 说明 |
|------|------|
| **Tier** | 按时间维度划分的价格层级，如 `peak`（高峰）。请求命中某个 tier 时使用对应价格，否则 fallback 到默认 `prices` |
| **TimeRange** | 一个时段定义，包含 `weekdays`（星期几，0=周日）、`start` / `end`（HH:MM，左闭右开） |
| **Provider 时段模板** | 定义在 `/providers` 上的 `time_zone` 和 `tiers`，同一 provider 下所有模型共享 |
| **Model tier 价格** | 定义在 `/model-prices` 上的 `tier_prices`，描述某个模型在某个 tier 下的价格 |

### 配置归属与下发链路

```
/provider deepseek
    ├── time_zone: Asia/Shanghai
    └── tiers: [peak]
    ├── model-prices deepseek-v3 chat
    │       ├── prices: 默认/空闲价
    │       └── tier_prices.peak: 高峰价
    └── model-prices deepseek-v4 chat
            └── ...

/cluster my-cluster
    ├── llm_config.provider: deepseek
    └── AIConf.ModelTable（导出时由控制面把 provider 的 time_zone/tiers
        与 model-prices 的 prices/tier_prices 拼接）
```

多个 cluster 引用同一个 provider 时，会各自得到一份相同的 `ModelTable` 数据；provider 的时段规则变更后，所有引用它的 cluster 在下一次配置导出时自动生效。

### Provider 时段模板

`/providers` 新增 `time_zone` 与 `tiers` 字段，用于描述共享的时段规则：

```json
{
  "name": "peak",
  "time_ranges": [
    { "weekdays": [1, 2, 3, 4, 5], "start": "09:00", "end": "12:00" },
    { "weekdays": [1, 2, 3, 4, 5], "start": "14:00", "end": "18:00" }
  ]
}
```

`time_zone` 须为合法 IANA 时区名，默认 `Asia/Shanghai`。同一 tier 内部的 `time_ranges` 不得重叠，且 `end` 必须大于 `start`；跨午夜时段需拆成两段。

### BFE ModelTable 结构

控制面导出给 BFE 的 `AIConf.ModelTable` 结构如下：

```go
type TimeRange struct {
    Weekdays []int  // 0=周日，1=周一 ... 6=周六
    Start    string // "HH:MM"
    End      string // "HH:MM"，必须 > Start
}

type PriceTier struct {
    Name       string      // 初期只支持 "peak"
    TimeRanges []TimeRange
}

type PriceMap map[string]float64
type TierPriceMap map[string]map[string]float64

type ModelPrice struct {
    Provider            string
    Model               string
    BaseModel           string
    Mode                string
    Capabilities        []string
    SupportedParameters []string
    Limits              map[string]interface{}
    Prices              PriceMap
    TierPrices          TierPriceMap
    Metadata            map[string]interface{}
}

type ModelTable struct {
    Currency string
    TimeZone string
    Tiers    []PriceTier
    Models   []ModelPrice
}
```

---

## BFE 侧成本计算逻辑

BFE 数据面在加载 `AIConf` 时完成价格数据的预处理，在请求转发结束后根据实际 Token 用量和当前时段计算成本。

### 配置加载阶段

```
AIConf.ModelTable
       │
       ▼
┌──────────────┐
│ 解析 TimeZone │─── 默认 Asia/Shanghai
│ 校验 Tiers    │─── name、时间格式、weekdays、区间重叠
└──────────────┘
       │
       ▼
┌──────────────┐
│ 价格转定点整数 │───  prices / tier_prices 均转为整数单位
└──────────────┘
```

BFE 使用定点整数存储价格，避免运行时浮点运算引入误差。所有价格字段按统一精度放大后参与扣减计算。

### 运行时时段匹配

BFE 在请求结束时根据当前时间匹配活跃 tier：

```go
func (table *ModelTable) ActiveTierName(now time.Time) string {
    if table == nil || len(table.Tiers) == 0 {
        return ""
    }
    t := now.In(table.tz)
    wd := int(t.Weekday())
    hour, min := t.Hour(), t.Minute()
    cur := hour*60 + min

    for i := range table.Tiers {
        tier := &table.Tiers[i]
        for _, tr := range tier.TimeRanges {
            if len(tr.Weekdays) > 0 && !containsInt(tr.Weekdays, wd) {
                continue
            }
            start := parseHHMM(tr.Start)
            end := parseHHMM(tr.End)
            if start <= cur && cur < end {
                return tier.Name
            }
        }
    }
    return ""
}
```

### 运行时成本计算

成本计算流程如下：

```
请求结束
   │
   ▼
解析 TokenUsage
   ├── prompt_tokens
   ├── completion_tokens
   └── cached_tokens
   │
   ▼
匹配 ActiveTierName
   │
   ├── 命中 tier ──► 取 tier_prices.<tier> 价格
   └── 未命中 tier ──► 取 prices 默认价格
   │
   ▼
分别计算
   ├── 缓存命中输入 = cached_tokens × cache_read_input_token_cost
   ├── 普通输入     = (prompt_tokens - cached_tokens) × input_cost_per_token
   └── 输出         = completion_tokens × output_cost_per_token
   │
   ▼
累加为本次请求总成本，用于 RMB 配额扣减与日志输出
```

若某个 tier 未配置特定价格键，则自动 fallback 到默认 `prices` 中的对应键。`TokenUsage` 增加 `CachedTokens` 字段后，缓存命中与未命中的输入 Token 可分别计价。

### 向后兼容

分时段定价设计保持了对固定价格的向后兼容：

- `/providers` 不填 `time_zone` / `tiers` 时，`ModelTable.TimeZone` / `ModelTable.Tiers` 为空，行为与固定价格完全一致；
- `/model-prices` 不填 `tier_prices` 时，始终按默认 `Prices` 计费；
- 命中 tier 但该 tier 未配置某个价格键时，自动 fallback 到默认 `Prices`；
- `TokenUsage.UsedCost`、Lua 扣减逻辑、Redis 定点数存储都不需要修改。

这种兼容方式使得现有部署可以平滑启用分时段能力，无需一次性全量调整配置。

---

## 定价数据与 Provider/Cluster 的关系

在 Provider 与 Cluster 概念分离后，模型定价的归属与引用关系也变得更加清晰。

### 概念分离回顾

- **Provider**：模型提供方，包含接入端点、可用模型、API Key、实例池、支持的协议以及时段模板（`time_zone`、`tiers`）。
- **Cluster**：转发集群，决定把流量按什么模型、什么权重、什么策略转发到某个 provider。
- **Model Price**：一条 `(provider, model, mode)` 的价格记录，`provider` 字段仅作为价格归集标识，不强制引用已存在的 provider。

### 配置下发关系

```
/providers
   ├── name: deepseek
   ├── time_zone: Asia/Shanghai
   └── tiers: [peak]

/model-prices
   ├── provider: deepseek, model: deepseek-v3, mode: chat
   │       ├── prices
   │       └── tier_prices.peak
   └── provider: deepseek, model: deepseek-v4, mode: chat
           ├── prices
           └── tier_prices.peak

/clusters
   └── name: my-cluster
           └── llm_config.provider: deepseek

导出 AIConf 时：
   ModelTable.TimeZone ← /providers.deepseek.time_zone
   ModelTable.Tiers    ← /providers.deepseek.tiers
   ModelTable.Models   ← /model-prices 中 provider=deepseek 的记录
```

### 弱引用带来的灵活性

`/model-prices` 的 `provider` 字段不再强制引用 `/providers`，带来了以下好处：

- 配置顺序更灵活，可以先维护价格数据，再接入 Provider；
- 删除 Provider 时，同名 `model_prices` 记录不再阻塞删除；
- 历史价格记录可以保留，即使对应 Provider 已下线，也便于成本追溯。

推荐配置顺序为 `/providers → /model-prices → /clusters → 路由规则`，但 `/model-prices` 与 `/providers` 之间为弱引用关系，实际可独立维护。

### 对成本计算的影响

弱引用关系意味着：BFE 在进行成本计算时，只关心 `AIConf.ModelTable` 中是否包含对应 `(provider, model, mode)` 的价格记录，而不关心该 provider 是否仍然存在。即使某个 Provider 已被删除，只要其价格记录保留，历史请求的成本核算依然可以正确完成。

同时，由于 `ModelTable` 在导出时按 `cluster.llm_config.provider` 查询并拼接，一个 cluster 引用的所有模型价格必须存在，否则该模型调用将无法完成成本计算，相关配额扣减也会失败。因此，在删除或重命名 Provider 时，需要同步评估其对价格数据和 cluster 引用的影响。

---

## 定价配置示例

以下示例展示如何为一个支持高峰/闲时计价的 Provider 配置模型定价。

### Provider 时段模板

```json
{
  "name": "deepseek",
  "description": "DeepSeek 官方 API",
  "model_protocols": ["openai"],
  "time_zone": "Asia/Shanghai",
  "tiers": [
    {
      "name": "peak",
      "time_ranges": [
        { "weekdays": [1, 2, 3, 4, 5], "start": "09:00", "end": "12:00" },
        { "weekdays": [1, 2, 3, 4, 5], "start": "14:00", "end": "18:00" }
      ]
    }
  ]
}
```

### model-list.yaml 片段

```yaml
version: v1.0
default_currency: "RMB"

models:
  - provider: "deepseek"
    model: "deepseek-v3"
    base_model: "deepseek-v3"
    mode: "chat"
    capabilities: ["chat", "reasoning", "tools", "structured_outputs", "prompt_caching"]
    supported_parameters: ["temperature", "top_p", "max_tokens", "tools", "tool_choice", "response_format", "reasoning"]
    limits:
      context_window: 128000
      max_input_tokens: 128000
      max_output_tokens: 8192
    prices:
      input_cost_per_token: 0.000002
      output_cost_per_token: 0.000008
      cache_read_input_token_cost: 0.0000005
      cache_creation_input_token_cost: 0.000001
    tier_prices:
      peak:
        input_cost_per_token: 0.000004
        output_cost_per_token: 0.000016
        cache_read_input_token_cost: 0.000001
    metadata:
      source: "https://platform.deepseek.com/pricing"
      notes: "DeepSeek V3 官方 API"
```

### Cluster 引用

```json
{
  "name": "deepseek-cluster",
  "llm_config": {
    "provider": "deepseek",
    "models": ["deepseek-v3"],
    "keys": [
      { "name": "key-primary", "weight": 100 }
    ]
  }
}
```

当上述配置导出到 BFE 后，工作日 09:00–12:00 与 14:00–18:00 之间对 `deepseek-v3` 的调用将按高峰价计费，其余时间按默认价格计费。

---

## 本章小结

- 模型定价数据是 AI 网关成本核算、配额扣减与成本分摊的基础，由 AI Gateway API 控制面统一维护并下发给 BFE。
- `ModelPrice` 以 `(provider, model, mode)` 为主键，包含能力、限制、默认价格和分时段价格等字段。
- `model-list.yaml` 提供批量导入能力，支持 `replace` 与 `merge` 两种模式，是当前版本维护模型价格的主要数据源格式。
- RMB 配额分时段定价通过 Provider 时段模板与 Model tier 价格配合实现，BFE 按请求发生时刻匹配 `peak` 等 tier，未命中时 fallback 到默认价格。
- BFE 数据面在加载阶段将价格转为定点整数，运行时根据 Token 用量和活跃 tier 完成纯整数成本计算，避免浮点误差。
- Provider 与 Cluster 概念分离后，`model-prices.provider` 仅作为价格归集标识，与 `/providers` 为弱引用关系，配置更灵活；`AIConf.ModelTable` 由控制面在导出时按 provider 拼接生成。

---

## 参考文档

- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/model-prices.md`
- `ai-gateway-api/design-docs/sys-design/details/RMB配额分时段定价.md`
- `ai-gateway-api/design-docs/sys-design/details/provider与cluster概念分离.md`
