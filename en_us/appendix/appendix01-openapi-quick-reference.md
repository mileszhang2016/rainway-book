# Appendix 1: OpenAPI Quick Reference

The OpenAPI v1 interface definitions of the Rainway AI Gateway Control Plane (AI Gateway API) have been split into independent documents by module and are continuously maintained on GitHub. This book does not repeat the full interface list; readers can view the authoritative definitions for the corresponding version via the links below.

## v0.0.11 API Documentation

- [OpenAPI Interface Definitions Directory](https://github.com/rainway-ai-gateway/ai-gateway-api/tree/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义)
- [Common Conventions](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/00-common.md): URL format, authentication method, response format, Method conventions, common Query parameters
- [Key Business Workflows](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/workflows.md)
- [Object Relations Diagram](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/object-relations.md)

## API Entry Points by Module

| Module | Documentation Link |
|------|----------|
| `/api-keys` | [api-keys.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/api-keys.md) |
| `/entity-types` | [entity-types.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/entity-types.md) |
| `/entities` | [entities.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/entities.md) |
| `/global-route-rules` | [global-route-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/global-route-rules.md) |
| `/route-tables` | [route-tables.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/route-tables.md) |
| `/providers` | [providers.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/providers.md) (includes the `instance_source` / `k8s_pool_name` instance-source fields) |
| `/clusters` | [clusters.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/clusters.md) |
| `/epp-assignments` | [epp-assignments.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/epp-assignments.md) |
| `/epp-pool` | [epp-pool.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/epp-pool.md) |
| `/model-prices` | [model-prices.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/model-prices.md) |
| `/certificates` | [certificates.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/certificates.md) |
| `/auth` | [auth.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/auth.md) |
| `/ai-cache-rules` | [ai-cache-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/ai-cache-rules.md) (AI cache rule collection, full replacement) |
| `/ai-cache-semantic-settings` | [ai-cache-semantic-settings.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/ai-cache-semantic-settings.md) (semantic cache global settings, singleton) |
| `/ai-context-rules` | [ai-context-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/ai-context-rules.md) (context compression rule collection, full replacement) |
| `/ai-context-settings` | [ai-context-settings.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/ai-context-settings.md) (context compression global settings, singleton) |
| `/traffic-mirror-rules` | [traffic-mirror-rules.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/traffic-mirror-rules.md) (traffic mirror rule collection, full replacement) |
| `/intent-config` | [intent-config.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/intent-config.md) (intent configuration, singleton) |
| `/expression/verify` | [expression-verify.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/expression-verify.md) |
| `/operation-logs` | [operation-logs.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/operation-logs.md) |
| `/report/overview`, `/report/timeseries`, `/report/rankings`, `/report/distribution`, `/report/logs` | [report.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/heads/v0.0.11-dev/design-docs/api-define/OpenAPI接口定义/report.md) (supports cache/mirror/intent dimension queries; endpoints return 404 when `[Report]` is not configured) |

## Notes

- The links above point to a snapshot of the `v0.0.11-dev` branch, suitable for cross-referencing with the content of this book.
- To view the latest API changes, visit the [ai-gateway-api main branch design docs](https://github.com/rainway-ai-gateway/ai-gateway-api/tree/develop/design-docs/api-define/OpenAPI接口定义).
