# Appendix 3: Common Error Codes

This book involves two categories of error codes: the OpenAI-compatible error codes returned by the BFE Data Plane in AI gateway scenarios, and the `ErrNum` business error codes returned by the AI Gateway API Control Plane OpenAPI. Both are maintained by the official BFE documentation and the Control Plane interface definition documents respectively; this book does not repeat them, but highlights the key semantics of both categories in AI gateway scenarios.

## BFE Data Plane Error Codes (v1.8.7)

- [BFE AI Gateway Error Codes](https://github.com/bfenetworks/bfe/blob/refs/tags/v1.8.7/docs/zh_cn/sys_design/ai_error_codes.md)

That document covers the following:

- Authentication and admission layer error codes (e.g. `NO_API_KEY`, `INVALID_API_KEY`, `KEY_DISABLED`, `SUBNET_NOT_ALLOWED`, `MODEL_NOT_ALLOWED`)
- Rate limit check layer error codes (e.g. `RPM_LIMIT_EXCEEDED`, `TPM_LIMIT_EXCEEDED`, `CONCURRENCY_LIMIT_EXCEEDED`)
- Quota deduction layer error codes (e.g. `QUOTA_EXHAUSTED`, `QUOTA_EXPIRED`, `INTERNAL_QUOTA_ERROR`)
- Forwarding and protocol adaptation layer error codes (e.g. `PROVIDER_PROTOCOL_MISMATCH`)
- Standard error response body structure and field descriptions
- List of reserved error codes
- Correlation between error codes and access log fields
- Troubleshooting advice by HTTP status code

429 `QUOTA_EXHAUSTED` covers two exhaustion scenarios: the `total_token` quota of a Quota Plan may be set to 0, meaning a "no-balance plan" — requests bound to such a plan are rejected directly with 429 `QUOTA_EXHAUSTED` (see `quotaPlanCheck()` in `bfe_modules/mod_ai_token_auth/token_rule_load.go`). In addition, a missing Redis balance key (e.g. a zero quota that was never synced to Redis) is treated as an exhausted balance rather than an internal error (see `HasBalance()` in `bfe_modules/mod_ai_token_auth/token.go`).

## AI Gateway API Control Plane Error Codes

The Control Plane OpenAPI uniformly returns a `{ErrNum, Data, ErrMsg}` structure; see [00-common.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/develop/design-docs/api-define/OpenAPI接口定义/00-common.md) for the general conventions. The `ErrNum` values are defined as follows:

| ErrNum | Meaning |
|--------|------|
| 200 | Success |
| 401 | Authentication failure |
| 402 | Insufficient call permission |
| 404 | The object being queried/updated/deleted does not exist |
| 409 | Resource dependency conflict |
| 422 | Invalid parameters |
| 500 | Other business logic errors (fallback code for unknown errors) |
| 555 | Duplicate object creation |
| 556 | Duplicate data |

Among them, 409 Conflict is dedicated to resource dependency conflict scenarios, while 500 remains the fallback code for unknown errors. Typical scenarios that return 409 include:

- Deleting a BFE cluster referenced by the AI cluster scheduler (`endpoints/openapi_v1/bfe_cluster/delete.go`);
- Deleting an instance pool referenced by a BFECluster or SubCluster (`model/icluster_conf/pool.go`);
- Deleting a certificate referenced by a Product (`model/iprotocol/certificate.go`);
- Deleting a product cluster referenced by route rules, or updating its models while it is referenced by route rules (`model/iroute_conf/route_rule.go`, `model/route_rules/route_rules.go`, which check rule target/fallback references);
- Deleting an Entity that still has child entities (`model/entity/entity_manager.go`).

Troubleshooting advice: when receiving a 409, first resolve the dependency identified by the referencing party name in `ErrMsg` (e.g. adjust route rules, rebind certificates, migrate child entities) and then retry the operation. When receiving a 500 whose `ErrMsg` contains no specific dependency information, treat it as an unknown error, collect logs, and contact the maintainers.

## Related Chapters in This Book

- [Chapter 16: Security Design](../design/chapter16-security-design.md) introduces the error response body and security audit log fields.
- [Chapter 18: Dashboard Basics](../operation/chapter18-dashboard-basics.md) introduces how to interpret common Dashboard error messages.
- [Chapter 21: API-Key and Quota Configuration](../operation/chapter21-apikey-and-quota-config.md) introduces troubleshooting methods for issues such as quota exhaustion and rate limit triggers.
