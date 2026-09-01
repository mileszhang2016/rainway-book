# 附1 OpenAPI 接口速查

壬远 AI 网关控制面（AI Gateway API）的 OpenAPI v1 接口定义已按模块拆分为独立文档，并在 GitHub 上持续维护。本书不再重复罗列完整接口列表，读者可通过以下链接直接查看对应版本的权威定义。

## v0.0.8 版本接口文档

- [OpenAPI 接口定义目录](https://github.com/rainway-ai-gateway/ai-gateway-api/tree/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义)
- [通用约定](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/00-common.md)：URL 格式、鉴权方式、返回值格式、Method 约定、通用 Query 参数
- [关键业务流程](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/workflows.md)
- [对象关系图](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/object-relations.md)

## 各模块接口入口

| 模块 | 文档链接 |
|------|----------|
| `/api-keys` | [api-keys.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/api-keys.md) |
| `/entity-types` | [entity-types.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/entity-types.md) |
| `/entities` | [entities.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/entities.md) |
| `/global-route-rules` | [global-route-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/global-route-rules.md) |
| `/route-tables` | [route-tables.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/route-tables.md) |
| `/providers` | [providers.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/providers.md) |
| `/clusters` | [clusters.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/clusters.md) |
| `/model-prices` | [model-prices.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/model-prices.md) |
| `/certificates` | [certificates.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/certificates.md) |
| `/auth` | [auth.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/auth.md) |
| `/alb-pool` | [alb-pool.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/alb-pool.md) |
| `/expression/verify` | [expression-verify.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/OpenAPI接口定义/expression-verify.md) |

## 说明

- 上述链接指向 `v0.0.8` Tag 的静态快照，适合与本书内容对照阅读。
- 若需查看最新接口变更，请访问 [ai-gateway-api 主分支设计文档](https://github.com/rainway-ai-gateway/ai-gateway-api/tree/develop/design-docs/api-define/OpenAPI接口定义)。
