# 附3 常见错误码

本书涉及两类错误码：BFE 数据面在 AI 网关场景下返回的 OpenAI 兼容错误码，以及 AI Gateway API 控制面 OpenAPI 返回的 `ErrNum` 业务错误码。两者分别由 BFE 官方文档与控制面接口定义文档维护，本书不重复罗列，仅说明两类错误码在 AI 网关场景下的关键语义。

## BFE 数据面错误码（v1.8.7）

- [BFE AI 网关错误码说明](https://github.com/bfenetworks/bfe/blob/refs/tags/v1.8.7/docs/zh_cn/sys_design/ai_error_codes.md)

该文档覆盖以下内容：

- 认证与准入层错误码（如 `NO_API_KEY`、`INVALID_API_KEY`、`KEY_DISABLED`、`SUBNET_NOT_ALLOWED`、`MODEL_NOT_ALLOWED`）
- 限流检查层错误码（如 `RPM_LIMIT_EXCEEDED`、`TPM_LIMIT_EXCEEDED`、`CONCURRENCY_LIMIT_EXCEEDED`）
- 配额扣减层错误码（如 `QUOTA_EXHAUSTED`、`QUOTA_EXPIRED`、`INTERNAL_QUOTA_ERROR`）
- 转发与协议适配层错误码（如 `PROVIDER_PROTOCOL_MISMATCH`）
- 标准错误响应体结构与字段说明
- 预留错误码列表
- 错误码与访问日志字段的关联
- 按 HTTP 状态码的排查建议

429 `QUOTA_EXHAUSTED` 覆盖两类耗尽场景：Quota Plan 的 `total_token` 配额允许配置为 0，表示"无余额计划"，绑定该计划的请求将直接被拒绝并返回 429 `QUOTA_EXHAUSTED`（参见 `bfe_modules/mod_ai_token_auth/token_rule_load.go` 的 `quotaPlanCheck()`）；同时 Redis 余额 key 不存在（例如配额为 0 从未同步到 Redis）也视为余额耗尽而非内部错误（参见 `bfe_modules/mod_ai_token_auth/token.go` 的 `HasBalance()`）。

## AI Gateway API 控制面错误码

控制面 OpenAPI 统一返回 `{ErrNum, Data, ErrMsg}` 结构，通用约定见 [00-common.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/develop/design-docs/api-define/OpenAPI接口定义/00-common.md)。`ErrNum` 取值约定如下：

| ErrNum | 含义 |
|--------|------|
| 200 | 调用成功 |
| 401 | 鉴权失败 |
| 402 | 没有调用权限 |
| 404 | 查询/修改/删除的对象不存在 |
| 409 | 资源依赖冲突 |
| 422 | 参数不合法 |
| 500 | 其他业务逻辑错误（未知错误的兜底返回码） |
| 555 | 创建重复对象 |
| 556 | 数据重复 |

其中 409 Conflict 专用于资源依赖冲突场景，500 保留为未知错误的兜底返回码。返回 409 的典型场景包括：

- 删除被 AI 集群调度引用的 BFE 集群（`endpoints/openapi_v1/bfe_cluster/delete.go`）；
- 删除被 BFECluster 或 SubCluster 引用的实例池（`model/icluster_conf/pool.go`）；
- 删除被 Product 引用的证书（`model/iprotocol/certificate.go`）；
- 删除被路由规则引用的产品集群，或集群被路由规则引用时更新其模型（`model/iroute_conf/route_rule.go`、`model/route_rules/route_rules.go`，规则 target/fallback 引用检查）；
- 删除仍有子节点的 Entity（`model/entity/entity_manager.go`）。

排查建议：收到 409 时应先根据 `ErrMsg` 中给出的引用方名称解除依赖关系（如调整路由规则、更换证书绑定、迁移子节点）再重试操作；收到 500 且 `ErrMsg` 不含具体依赖信息时，按未知错误收集日志并联系维护方定位。

## 本书相关章节

- [第十六章 安全设计](../design/chapter17-security-design.md) 中介绍了错误响应体与安全审计日志字段。
- [第十九章 控制台基础操作](../operation/chapter19-dashboard-basics.md) 中介绍了常见控制台错误提示的对照思路。
- [第二十二章 API-Key 与配额配置](../operation/chapter22-apikey-and-quota-config.md) 中介绍了配额耗尽、限流触发等问题的排查方法。
