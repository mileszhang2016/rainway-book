# 附2 InnerAPI 配置导出格式

壬远 AI 网关控制面通过 InnerAPI v1 向数据面（BFE）和 Conf Agent 导出配置。各配置主题的导出格式、数据模型与接口说明已在 GitHub 上完整维护。本书不再重复展开，读者可通过以下链接查看 v0.0.8 版本的权威定义。

## v0.0.8 版本 InnerAPI 文档

- [InnerAPI 接口定义目录](https://github.com/rainway-ai-gateway/ai-gateway-api/tree/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义)
- [接口总览](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/00-overview.md)
- [通用约定](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/01-common.md)
- [接口清单](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/02-interface-list.md)
- [数据模型](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/data-models.md)
- [附录](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/appendix.md)

## 各配置主题导出格式

| 配置主题 | 文档链接 |
|----------|----------|
| Server Data Conf | [server-data-conf.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/server-data-conf.md) |
| GSLB / Cluster Table | [gslb.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/gslb.md)、[cluster-table.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/cluster-table.md) |
| Server Cert Conf | [server-cert-conf.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/server-cert-conf.md) |
| Extra Files | [extra-files.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/extra-files.md) |
| mod-ai-key | [mod-api-key.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/mod-api-key.md) |
| mod-body-process | [mod-body-process.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/mod-body-process.md) |
| rate-limit-policy | [rate-limit-policy.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/rate-limit-policy.md) |
| ai-route | [ai-route.md](https://github.com/rainway-ai-gateway/ai-gateway-api/blob/refs/tags/v0.0.8/design-docs/api-define/InnerAPI接口定义/ai-route.md) |

## 说明

- InnerAPI 返回的配置通常由 Conf Agent 拉取后写入本地版本目录，再触发热加载。
- 各主题导出的 JSON 结构与 BFE 数据面配置文件的字段命名保持一致，详细字段说明见上述链接。
