# 第二十二章 限流策略配置

## 本章目标

本章介绍如何在壬远AI网关控制面（AI Gateway API）中创建、绑定并验证限流策略（RateLimitPolicy）。阅读本章后，读者将能够：

- 理解 `RateLimitPolicy` 的数据模型与三种限流维度；
- 掌握 TPM、RPM 与最大并发限制的配置方法；
- 将限流策略绑定到 API-Key 或 Entity；
- 理解 Entity 层级合并对最终限流效果的影响；
- 通过 BFE 配置、Redis 计数器与响应行为验证限流是否生效。

## RateLimitPolicy 的数据模型

限流策略用于控制 API-Key 或 Entity 对后端 AI 模型的访问速率，避免突发流量压垮上游服务。控制面在 MySQL 中通过 `rate_limit_policies` 表持久化策略，核心字段见 `ai-gateway-api/design-docs/sys-design/details/限流策略与导出.md`。

```json
{
  "id": 1,
  "name": "ratelimitX",
  "description": "默认限流策略",
  "product_name": "AI_product",
  "enabled": true,
  "max_concurrency": 50,
  "tpm_configs": [
    {
      "name": "tpm_default",
      "model": "*",
      "window_minutes": 1,
      "max_tokens": 100000,
      "step_minutes": 1
    }
  ],
  "rpm_configs": [
    {
      "name": "rpm_default",
      "model": "*",
      "window_minutes": 1,
      "max_requests": 1000
    }
  ]
}
```

字段说明如下：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int64 | 策略在数据库中的唯一标识，导出后对应 `rlp-<id>` |
| `name` | string | 策略名称，仅在控制面展示使用 |
| `description` | string | 策略描述，便于运维识别用途 |
| `product_name` | string | 产品线名称，当前默认填充为 `AI_product` |
| `enabled` | bool | 是否启用；禁用后不会导出到 BFE |
| `max_concurrency` | int | 最大并发请求数；`-1` 表示不限制 |
| `tpm_configs` | array | TPM（Tokens Per Minute）规则列表 |
| `rpm_configs` | array | RPM（Requests Per Minute）规则列表 |

在业务层，控制面使用 `RateLimitPolicyParam` 承载创建与更新参数，其中 `TpmConfigs` 与 `RpmConfigs` 分别为 `TPMConfig` 和 `RPMConfig` 切片。导出到 BFE 时，这些结构会被转换为 `BfeRateLimitPolicy`，并附带控制面生成的稳定 `redis_key`。

## TPM、RPM、并发限制的配置方法

壬远AI网关的限流策略支持三类限制：TPM（每分钟 Token 数）、RPM（每分钟请求数）与最大并发数。三类限制可以单独使用，也可以组合使用，以满足不同业务场景对成本和稳定性的双重诉求。

### TPM 配置

TPM 限制单位时间窗口内消耗的 Token 总量，适用于按 Token 计费或后端 GPU 显存敏感的场景。每条规则包含：

| 字段 | 说明 |
|------|------|
| `name` | 规则名，同一策略内唯一，创建后不可修改 |
| `model` | 适用模型，`"*"` 表示默认限制 |
| `window_minutes` | 时间窗口长度（分钟） |
| `max_tokens` | 窗口内允许的最大 Token 数 |
| `step_minutes` | 滑动窗口步长（分钟） |

示例：对 `gpt-4` 设置 1 分钟 10 万 Token 限制。

```json
{
  "name": "tpm_gpt4",
  "model": "gpt-4",
  "window_minutes": 1,
  "max_tokens": 100000,
  "step_minutes": 1
}
```

TPM 规则支持按模型细粒度控制。当请求模型同时命中具体模型规则与 `"*"` 默认规则时，BFE 优先匹配具体模型名；未命中具体模型时，再回落到默认规则。

### RPM 配置

RPM 限制单位时间窗口内的请求次数，适用于控制调用频率、防止脚本滥用的场景。每条规则包含：

| 字段 | 说明 |
|------|------|
| `name` | 规则名，同一策略内唯一 |
| `model` | 适用模型，`"*"` 表示默认限制 |
| `window_minutes` | 时间窗口长度（分钟） |
| `max_requests` | 窗口内允许的最大请求数 |

示例：对默认模型设置 1 分钟 1000 次请求限制。

```json
{
  "name": "rpm_default",
  "model": "*",
  "window_minutes": 1,
  "max_requests": 1000
}
```

RPM 与 TPM 的窗口可以独立设置。例如，可配置 1 分钟 RPM 限制应对突发流量，同时配置 60 分钟 TPM 限制控制总体 Token 预算。

### 并发限制

`max_concurrency` 控制同一时刻正在处理的请求数，防止后端因大量长连接请求而堆积。该值应依据后端模型服务的并发承载能力设定；设为 `-1` 表示不限制。对于流式（SSE）场景，并发限制尤为重要，因为流式请求会持续占用连接资源。

## 创建限流策略

在 AI Gateway API 中，限流策略通常通过 OpenAPI 的 `/api-keys` 或 `/entities` 接口随 API-Key / Entity 一并创建，也可通过独立的限流策略管理接口维护。创建时需要满足以下校验规则，详见 `ai-gateway-api/design-docs/sys-design/details/限流策略与导出.md`：

- `name` 必填、非空、长度 1-128 字符，字符集限定为 `[a-zA-Z0-9_-]`；
- `name` 在同一 `RateLimitPolicy` 内唯一，创建后不可修改；
- `model` 不能为空；
- `max_tokens`、`max_requests` 等 limit 字段必须 ≥ 0；
- `max_concurrency` 必须 ≥ 0（`0` 的语义需与 BFE 实现对齐）。

以下示例展示了在创建 API-Key 时同时启用限流策略的请求体：

```json
{
  "description": "测试Key",
  "rate_limit_policy": {
    "enabled": true,
    "rules": {
      "tpm": [
        {"name": "tpm_1min", "model": "*", "window_minutes": 1, "max_tokens": 10000, "step_minutes": 1}
      ],
      "rpm": [
        {"name": "rpm_1min", "model": "*", "window_minutes": 1, "max_requests": 100}
      ],
      "max_concurrency": 50
    }
  }
}
```

创建 Entity 时的结构与上述一致，字段定义可参考 `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/entities.md`。若未传入 `rate_limit_policy`，系统将使用默认值：`enabled = false`，`rules` 为空，即不对该对象执行限流。

## 将限流策略绑定到 API-Key 或 Entity

限流策略通过外键关联到 API-Key 或 Entity，对应数据库字段为：

- `api_keys.rate_limit_policy_id`
- `entities.rate_limit_policy_id`

绑定方式有两种：

1. **创建时绑定**：在创建 API-Key 或 Entity 的请求体中直接传入 `rate_limit_policy`，系统会同时创建策略并建立引用。
2. **更新时绑定**：通过 `PUT /api-keys/{id}` 或 `PUT /entities/{id}` 全量更新，也可使用 `PATCH` 部分更新 `rate_limit_policy` 字段。

当策略的 `enabled` 为 `false` 时，即使已经被 API-Key 或 Entity 引用，导出到 BFE 时也会跳过该策略及对应绑定。这一设计使得运维人员可以临时下线某个策略，而不必解除所有引用关系。

需要注意的是，删除 API-Key 或 Entity 时，会级联删除其专属的策略底层资源，但仅限于这些资源不被其他 API-Key 或 Entity 引用的情况。

## 层级合并效果说明

壬远AI网关支持 Entity 层级结构，常见的层级类型包括组织（org）、部门（dep）、团队（team）与个人（person）。当 API-Key 挂载到某个 Entity 时，限流策略会按层级向上合并，最终形成该 API-Key 需要遵守的全部策略集合。

合并流程如下：

1. 取 API-Key 自身的 `rate_limit_policy_id`；
2. 递归向上收集该 Entity 及其所有祖先 Entity 的 `rate_limit_policy_id`；
3. 跳过 `enabled=false` 的策略；
4. 为每个启用的策略生成一条绑定记录。

下图展示了一个典型的层级合并场景：

```text
root (ent-root, policy: rlp-0002)
  └─ dep (ent-dep, policy: rlp-0003)
       └─ team (ent-team, 未绑定策略)
            └─ API-Key ak-test (policy: rlp-0001)
```

对于 `ak-test`，最终导出的绑定顺序为 `["rlp-0001", "rlp-0003", "rlp-0002"]`，从自身向上排列。BFE 在请求到达时会依次检查这些策略中的规则，任一规则触发都会导致限流。

导出后的策略名称统一为 `rlp-<policy_id>`，避免命名冲突并便于 BFE 索引。控制面同时为每条 TPM/RPM 规则生成稳定的 Redis Key，例如 `RL_TPM_rlp-1_tpm_1min`。Key 基于 `(policy_id, rule_name)` 生成，因此修改 `model` 不会重置计数器，而删除规则会清理对应 Key，新增规则会生成新的 Key。

如果同一策略在层级中多次出现，导出时会生成多条绑定记录；BFE 会按命中顺序处理或去重。建议在 Entity 设计时避免重复绑定同一策略，以减少配置量和排查复杂度。

## 验证限流是否生效

策略生效需要经历“控制面导出 → Conf Agent 下发 → BFE 热加载”三个环节。验证步骤如下：

1. 在 AI Gateway API 中确认 `rate_limit_policy.enabled = true`，并且规则参数符合预期；
2. 查询导出的 `rate_limit_policies.json` 与 `api_key_rl_policy_bindings.json`，确认策略与绑定已生成；
3. 在 BFE 中确认 `mod_ai_rate_limit` 已加载，并读取到 `ai_rate_limit.data`；
4. 使用压测工具以超过 RPM/TPM 阈值的速率发起请求；
5. 观察响应状态码与 Redis 计数器变化。

可通过 Redis 查看对应计数器是否被写入，例如：

```bash
redis-cli GET RL_RPM_rlp-1_rpm_1min
```

如果计数器持续增加并在阈值附近触发 429 响应，说明限流已生效。建议在验证时使用单一 API-Key 和固定模型，避免多 Key、多模型并发导致结果难以分析。

## 限流触发后的响应

当请求触发任一限流规则或超过最大并发数时，BFE 会返回 `429 Too Many Requests`。客户端收到该响应后，应实施退避（backoff）重试，避免继续冲击网关。推荐的客户端行为包括：

- 首次失败后等待随机时间（如 100ms ~ 1s）再重试；
- 多次失败后采用指数退避；
- 在重试次数达到上限后放弃请求并记录告警。

> 当前 BFE 模块行为以 `bfe/docs/zh_cn/configuration/mod_ai_rate_limit/ai_rate_limit.data.md` 中的配置为准；响应头是否携带 `Retry-After` 取决于具体 BFE 版本实现。

## 完整配置示例

以下是一个导出到 BFE 的完整限流配置示例，文件路径通常为 `conf/mod_ai_rate_limit/ai_rate_limit.data`：

```json
{
  "Version": "1.0",
  "Config": {
    "AI_product": [
      {
        "cond": "default_t()",
        "hit_action": {
          "cmd": "PASS"
        }
      }
    ]
  },
  "RateLimitPolicies": {
    "rlp-0001": {
      "name": "ratelimitX",
      "enabled": true,
      "rules": {
        "tpm": [
          {
            "name": "tpm_default",
            "window_minutes": 1,
            "max_tokens": 10000,
            "step_minutes": 1,
            "models": ["*"],
            "redis_key": "RL_TPM_rlp-0001_tpm_default"
          }
        ],
        "rpm": [
          {
            "name": "rpm_default",
            "window_minutes": 1,
            "max_requests": 100,
            "burst": 1,
            "models": ["*"],
            "redis_key": "RL_RPM_rlp-0001_rpm_default"
          }
        ],
        "max_concurrency": 50
      }
    }
  },
  "ApikeyRateLimitPolicyBindings": {
    "ak-2v8x9k3m7p": ["rlp-0001"]
  }
}
```

在该示例中，API-Key `ak-2v8x9k3m7p` 绑定了策略 `rlp-0001`，受到 1 分钟 10000 Token、1 分钟 100 次请求以及最大 50 并发的三重限制。`Config` 段中的 `default_t()` 条件表示该限流规则默认对所有请求生效，`hit_action.cmd` 为 `PASS` 表示命中后放行到后续模块处理。

## 本章小结

本章介绍了壬远AI网关限流策略的完整配置方法：

- `RateLimitPolicy` 通过 `tpm_configs`、`rpm_configs` 与 `max_concurrency` 三个维度控制访问速率；
- 策略可随 API-Key 或 Entity 创建，也可通过更新接口重新绑定；
- Entity 层级会向上合并所有启用的策略，最终生成多绑定关系；
- 控制面导出 `rate_limit_policies.json` 与 `api_key_rl_policy_bindings.json` 供 BFE 消费，并为每条规则生成稳定 Redis Key；
- 限流触发后 BFE 返回 429，客户端应配合退避逻辑重试。

理解并正确配置限流策略，是保障后端 AI 模型服务稳定性、控制成本支出的关键操作之一。

## 参考文档

- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/api-keys.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/entities.md`
- `ai-gateway-api/design-docs/sys-design/details/限流策略与导出.md`
- `bfe/docs/zh_cn/configuration/mod_ai_rate_limit/ai_rate_limit.data.md`
