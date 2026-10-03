# 附1 OpenAPI 接口速查

壬远 AI 网关控制面（AI Gateway API）的 OpenAPI v1 接口定义已按模块拆分为独立文档，并在 GitHub 上持续维护。本书不再重复罗列完整接口列表，读者可通过以下链接直接查看对应版本的权威定义。

## v0.0.11 版本接口文档

- [OpenAPI 接口定义目录](https://github.com/rainway-ai-gateway/ai-gateway-api/tree/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义)
- [通用约定](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/00-common.md)：URL 格式、鉴权方式、返回值格式、Method 约定、通用 Query 参数
- [关键业务流程](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/workflows.md)
- [对象关系图](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/object-relations.md)

## 各模块接口入口

| 模块 | 文档链接 |
|------|----------|
| `/api-keys` | [api-keys.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/api-keys.md) |
| `/entity-types` | [entity-types.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/entity-types.md) |
| `/entities` | [entities.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/entities.md) |
| `/global-route-rules` | [global-route-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/global-route-rules.md) |
| `/route-tables` | [route-tables.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/route-tables.md) |
| `/providers` | [providers.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/providers.md)（含 `instance_source` / `k8s_pool_name` 实例来源字段） |
| `/clusters` | [clusters.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/clusters.md) |
| `/epp-assignments` | [epp-assignments.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/epp-assignments.md) |
| `/epp-pool` | [epp-pool.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/epp-pool.md) |
| `/model-prices` | [model-prices.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/model-prices.md) |
| `/certificates` | [certificates.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/certificates.md) |
| `/auth` | [auth.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/auth.md) |
| `/ai-cache-rules` | [ai-cache-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/ai-cache-rules.md)（AI 缓存规则集合，全量替换） |
| `/ai-cache-semantic-settings` | [ai-cache-semantic-settings.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/ai-cache-semantic-settings.md)（语义缓存全局设置，单例） |
| `/ai-context-rules` | [ai-context-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/ai-context-rules.md)（上下文压缩规则集合，全量替换） |
| `/ai-context-settings` | [ai-context-settings.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/ai-context-settings.md)（上下文压缩全局设置，单例） |
| `/traffic-mirror-rules` | [traffic-mirror-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/traffic-mirror-rules.md)（流量镜像规则集合，全量替换） |
| `/intent-config` | [intent-config.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/intent-config.md)（意图配置，单例） |
| `/expression/verify` | [expression-verify.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/expression-verify.md) |
| `/operation-logs` | [operation-logs.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/operation-logs.md) |
| `/report/overview`、`/report/timeseries`、`/report/rankings`、`/report/distribution`、`/report/logs` | [report.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/report.md)（支持缓存/镜像/意图维度查询；`[Report]` 未配置时端点 404） |

## 说明

- 上述链接指向 `v0.0.11-dev` 分支快照，适合与本书内容对照阅读。
- 若需查看最新接口变更，请访问 [ai-gateway-api 主分支设计文档](https://github.com/rainway-ai-gateway/ai-gateway-api/tree/develop/design-docs/api-define/OpenAPI接口定义)。
