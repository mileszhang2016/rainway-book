# 附4 术语表

## A

**AI 业务集群（Cluster）**
：引用 Provider 并配置转发策略的逻辑后端单元，决定流量如何转发、使用哪些模型、Key 权重如何分配。

**AI 网关实例池**
：控制台中登记的数据面 BFE 引擎地址清单，用于控制面向指定 BFE 节点下发配置。

**API Key**
：调用方访问壬远 AI 网关的凭证，请求头中携带（不需要 Bearer 前缀）。可挂载到 Entity 继承配额、限流与路由策略。

## B

**BFE**
：百度开源的七层负载均衡与流量网关，在壬远 AI 网关中作为数据面转发引擎，负责 AI 请求的鉴权、限流、配额、路由与转发。

**Balancer**
：BFE 的负载均衡模块，负责在 Cluster 内部多个后端实例之间分发请求。

## C

**Cluster（集群）**
：见“AI 业务集群”。

**Conf Agent（配置代理）**
：部署在 BFE 侧的组件，周期性地从 AI Gateway API 拉取最新配置，写入本地版本目录并触发 BFE 热加载。

**Condition（条件表达式）**
：BFE 提供的表达式语言，AI 路由规则通过 `Cond` 字段描述命中条件，如 `req_body_json_in("model", "gpt-4", false)`。

## D

**Dashboard（控制台）**
：壬远 AI 网关的 Web 管理界面，面向运维人员提供资源管理、消费者管理、路由管理、用户管理等功能。

## E

**Entity（组织）**
：调用方分组，可表达部门、团队或项目。每个 Entity 可挂载配额计划、限流策略、模型黑白名单与路由规则。

**Entity Type（组织类型）**
：Entity 的层级分类定义，通过 `level` 字段区分层级高低，数值越小层级越高。

## F

**Fallback（降级）**
：AI 路由规则中 `fallbacks` 指定的备用目标。当首选 `targets` 全部失败时，按顺序尝试 fallback 目标。

## G

**Global 路由表**
：AI 路由规则三级体系中的最底层兜底规则，所有 API-Key 最终都会绑定并查找。

## I

**InnerAPI**
：AI Gateway API 提供的内部接口，主要供 Conf Agent 与 BFE 拉取配置、完成版本同步。

**Instance Pool（实例池）**
：在 Provider 中定义的后端 AI 服务真实地址、端口与权重集合，供 Cluster 引用。

## K

**Key Affinity（Key 亲和性）**
：基于 Redis 实现会话级 Key 亲和，同一 `ClientKeyId` 在一定时间内持续命中同一 Provider Key。

## M

**Model Mapping（模型映射）**
：Cluster 中将用户请求的模型名映射为后端实际使用的模型名的机制。

**Model Price（模型定价）**
：维护模型在不同 Provider 与时段下的价格，用于 RMB 配额成本核算。

**Model Protocol（模型协议）**
：Provider 支持的上游协议类型，如 `openai`、`anthropic` 等，用于请求体与响应体的协议适配。

## O

**OpenAPI**
：AI Gateway API 提供的对外管理接口，供 Dashboard 与外部程序调用，完成资源创建、查询、更新与删除。

## P

**Provider（模型服务商）**
：持有后端实例池、模型协议、模型列表与服务鉴权 Key 明文的资源；Cluster 通过“所属服务商”引用 Provider。

**Product（产品线）**
：BFE 中的顶层资源隔离单位，AI 网关模式下主要用于产品线识别与配置上下文加载。

## Q

**Quota Plan（配额计划）**
：为 API-Key 或 Entity 分配的 Token 总量或 RMB 预算，支持 `total_token` 与 `RMB` 两种单位及周期重置。

## R

**RateLimitPolicy（限流策略）**
：控制 API-Key 或 Entity 对后端 AI 模型访问速率的策略，支持 TPM、RPM 与最大并发数限制。

**Redis 唯一真实来源**
：配额余额、限流计数等运行时状态直接读写 Redis，管理面查询余额时不再维护数据库冷副本。

**Route Rule（路由规则）**
：AI 路由表中的单条规则，包含 `Cond` 命中条件、`targets` 目标列表与可选的 `fallbacks` 降级列表。

**Route Table（路由表）**
：Global / Entity / API-Key 三级 AI 路由规则集合，按 `apikey > entity > global` 优先级依次匹配。

## S

**Session Key**
：Dashboard 登录后由 `/auth/session-keys` 生成的会话凭证，格式为 `Authorization: Session {session_key}`。

**Sticky Session（会话保持）**
：将同一客户端请求长期绑定到同一后端实例的机制，AI 场景通常无需开启。

**Sub Cluster（子集群）**
：Cluster 生成 BFE 配置时自动创建的子集群，绑定到 Cluster 对应的实例池。

## T

**Tier（时段层级）**
：模型定价中按时间维度划分的价格层级，如 `peak`（高峰）、`off-peak`（空闲），用于分时段计费。

**Token**
：由 `/auth/tokens` 创建的程序访问凭证，分为 `System`（完整管理权限）与 `Support`（只读导出权限）两种 Scope。

**TPM（Tokens Per Minute）**
：每分钟 Token 消耗上限，采用滑动窗口计数。

**RPM（Requests Per Minute）**
：每分钟请求次数上限，采用固定窗口计数。

## V

**VersionControlManager**
：AI Gateway API 中负责配置导出、MD5 签名与版本号管理的组件。

**Visitor**
：AI Gateway API 认证授权模块中对用户与 Token 的统一抽象。

## W

**加权随机（Weighted Random）**
：AI 路由规则在 `targets` 之间按权重随机选择目标 Cluster 的算法。

## 参考

- `ai-gateway-web/docs/zh-cn/12-appendix.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/README.md`
- `bfe/docs/zh_cn/sys_design/ai_error_codes.md`
