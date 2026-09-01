# 第七章 数据面转发设计：BFE

## 本章目标

本章聚焦壬远 AI 网关的数据面组件 BFE（Beyond Front End），介绍 BFE 如何接收、处理并转发 AI 请求。通过阅读本章，读者将能够理解：

- BFE 在壬远 AI 网关中承担的角色及其与控制面（AI Gateway API）的关系；
- BFE 的请求处理生命周期，尤其是 AI 网关模式下的独立转发路径；
- `mod_ai_route`、`mod_ai_token_auth`、`mod_ai_rate_limit`、`mod_body_process` 四个 AI 相关模块的执行顺序与协作方式；
- BFE 模块框架 `bfe_module` 的回调机制与模块注册方式；
- AI 相关配置文件的加载、校验与热加载机制；
- 关键配置示例与转发降级行为。

## BFE 在壬远 AI 网关中的角色

壬远 AI 网关采用控制面与数据面分离的架构。控制面由 AI Gateway API 负责，完成 Provider、Cluster、API-Key、配额、限流策略等元数据的管理；数据面由 BFE Server 承担，负责把终端用户请求转发到实际的大模型后端。

BFE 是一个开源的七层负载均衡器，起源于百度，现为 CNCF Sandbox 项目。壬远 AI 网关在其基础上扩展了 AI 相关的模块与转发路径，使其具备以下能力：

- 基于 API-Key、Entity、Global 三级优先级进行 AI 路由；
- 对 API-Key 进行鉴权、配额校验与使用量的最终扣除；
- 按产品、API-Key 等维度进行分布式限流；
- 解析流式响应（SSE）中的 token 使用量，用于配额统计；
- 在目标集群失败时按 `fallbacks` 顺序降级，支持多 API-Key 轮换与重试。

数据面需要具备高并发、低延迟、可观测和可热更新等特性。BFE 通过事件驱动的连接处理、模块化的回调框架以及原子化的配置切换，满足这些要求。控制面与数据面的交互方式详见 `bfe/AGENTS.md`：控制面通过配置分发机制将路由、鉴权、限流等数据下发到 BFE；BFE 在运行期加载这些配置并按规则转发流量。数据面不直接访问控制面的数据库，只消费由控制面生成的配置文件，这种解耦使得数据面可以独立扩展和升级。

## BFE 请求处理生命周期

BFE 在启动后监听 HTTP/HTTPS/HTTP2/WebSocket 等连接。每个请求进入 BFE 后，会依次经过连接接入、协议解析、租户识别、模块回调、后端转发、响应发送等阶段。在 AI 网关模式下，BFE 会进入独立的 `ServeHTTPForAI()` 转发路径。

连接接入阶段由 `bfe_server/` 中的监听器完成，负责 TLS 握手、会话管理和协议协商。HTTP 请求解析由 `bfe_http/`、`bfe_http2/` 等协议实现负责，生成 `bfe_basic.Request` 对象。随后 BFE 进入模块回调阶段，按固定顺序调用已注册模块的回调函数。AI 相关模块主要在 `HandleFoundProduct` 阶段介入，而响应阶段则由 `HandleReadResponse` 与 `HandleRequestFinish` 处理。

### 传统路径与 AI 网关路径的分发

在 `bfe_server/http_conn.go` 的 `conn.serveRequest()` 中，BFE 根据 `EnableAiGateway` 配置决定进入哪条路径：

```go
var ret1 int
if c.server.Config.Server.EnableAiGateway {
    ret1 = c.server.ReverseProxy.ServeHTTPForAI(w, request)
} else {
    ret1 = c.server.ReverseProxy.ServeHTTP(w, request)
}
```

当 `ai_gateway_enabled = false` 时，请求沿用 BFE 原有的 `ServeHTTP()` 路径；当 `ai_gateway_enabled = true` 时，请求进入新增的 `ServeHTTPForAI()` 路径。两条路径共享连接管理、超时、响应发送等基础设施，但 AI 路径不再使用原有租户内集群路由，而是使用 `mod_ai_route` 计算出的 `AiRouteResult` 进行转发。

### AI 网关路径处理流程

```
┌─────────────────────────────────────┐
│  接收 HTTP/HTTPS/HTTP2 请求          │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  setClientAddr()                    │
│  设置原始客户端地址                  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  HandleBeforeLocation               │
│  mod_trust_clientip / mod_logid 等  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  findProduct()                      │
│  识别 BFE 租户（AI 网关场景兼容保留） │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  HandleFoundProduct                 │
│  mod_ai_token_auth                  │
│  mod_ai_route                       │
│  mod_ai_rate_limit                  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  AiRouteResult 检查                 │
│  未命中 → 返回 404 Not Found        │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  HandleAfterLocation                │
│  mod_body_process 等                │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  SelectTarget() 加权选择 target      │
│  构造 [target] + fallbacks 尝试列表  │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  aiClusterInvoke() 循环转发          │
│  失败时按 fallbacks 顺序降级         │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  HandleReadResponse / 发送响应       │
└─────────────────────────────────────┘
```

从图中可以看出，AI 网关路径在 `HandleFoundProduct` 阶段聚合了所有 AI 相关模块的处理结果，并在 `ServeHTTPForAI()` 中完成 target 选择、模型覆盖与 fallback 降级。

## AI 相关模块的执行顺序与协作

BFE 的模块通过 `bfe_module` 框架注册到固定回调点。AI 相关模块的注册顺序在 `bfe/bfe_modules/bfe_modules.go` 的 `moduleList` 中显式指定：

```go
var moduleList = []bfe_module.BfeModule{
    // ... 其他模块 ...

    // mod_ai_token_auth
    mod_ai_token_auth.NewModuleAITokenAuth(),

    // mod_ai_route
    // 要求：排在 mod_ai_token_auth 之后（需要 ClientApiKey），在 mod_body_process 之前
    mod_ai_route.NewModuleAiRoute(),

    // mod_body_process
    mod_body_process.NewModuleBodyProcess(),

    // 依赖 token 计算
    mod_ai_rate_limit.NewModuleAiRateLimit(),

    // ...
}
```

该顺序决定了模块在 `HandleFoundProduct` 回调中的执行顺序：`mod_ai_token_auth` → `mod_ai_route` → `mod_ai_rate_limit`。三个模块通过 `AiBasicInfo` 与 `Request.Context` 共享状态，例如 `ClientApiKey`、`ClientModel`、`TargetModel`、`AiRouteResult` 等。

执行顺序的约束由数据依赖决定：

- `mod_ai_token_auth` 最早执行，识别调用方身份并设置 `ClientApiKey`；
- `mod_ai_route` 紧随其后，依赖 `ClientApiKey` 完成路由查找，生成 `AiRouteResult`；
- `mod_ai_rate_limit` 最后执行，依赖 `ClientApiKey`、目标模型等信息执行 TPM/RPM/并发限流。

`mod_body_process` 主要在 `HandleReadResponse` 阶段执行，负责解析流式响应中的 Token 用量，其计算结果会供 `mod_ai_token_auth` 在 `HandleRequestFinish` 阶段进行最终配额扣减。任何顺序调整都会破坏这一依赖链，导致路由、限流或配额扣减行为异常。因此 `bfe_modules/bfe_modules.go` 中的注册位置与注释需要同步维护。

### 模块协作关系

```
                    ┌─────────────────┐
                    │   HTTP Request  │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
    ┌─────────────────┐ ┌─────────────┐ ┌─────────────────┐
    │ mod_ai_token_auth│ │ mod_ai_route │ │ mod_ai_rate_limit│
    │ API-Key 鉴权     │ │  路由查找    │ │  限流控制       │
    └────────┬────────┘ └──────┬──────┘ └────────┬────────┘
             │                 │                 │
             ▼                 ▼                 ▼
    ┌─────────────────────────────────────────────────────┐
    │          AiBasicInfo / Request.Context              │
    │  - ClientApiKey                                     │
    │  - ClientModel / TargetModel                        │
    │  - QuotaPlan / TokenUsage                           │
    │  - AiRouteResult (targets / fallbacks)              │
    └────────────────────────┬────────────────────────────┘
                             │
                             ▼
                 ┌───────────────────────┐
                 │ ReverseProxy.ServeHTTPForAI │
                 └───────────────────────┘
```

### mod_ai_route：AI 路由模块

`mod_ai_route` 根据 AI 路由规则将请求路由到不同后端集群与模型。它支持按 `apikey → entity → global` 三级优先级组织的规则表，每张表由 `ApikeyRouteTableBindings` 与 API-Key 绑定。命中后返回 `targets` 列表（含 `ClusterName`、`Model`、`Weight`）与 `fallbacks` 列表。

路由查找在 `HandleFoundProduct` 回调中完成。模块从 `AiBasicInfo.ClientApiKey` 获取 API-Key，按绑定顺序查找路由表，命中后将 `AiRouteResult` 写入请求上下文。`AiRouteResult` 的定义位于 `bfe_basic/request_ai_route.go`：

```go
type AiRouteResult struct {
    RouteType string   // apikey / entity / global
    Owner     string   // 路由表属主
    RuleName  string   // 命中规则名
    Targets   []AiRouteTarget
    Fallbacks []AiRouteFallback
}
```

`mod_ai_route` 本身不执行加权选择，也不负责实际转发，只负责把路由结果写入上下文，供 `ServeHTTPForAI()` 使用。

### mod_ai_token_auth：API-Key 鉴权模块

`mod_ai_token_auth` 负责验证请求携带的 API-Key。请求需在 `Authorization` Header 中按如下格式携带 token：

```
Authorization: Bearer <api-key>
```

模块的鉴权流程包括：

1. 从请求中提取 API-Key；
2. 验证 API-Key 的有效性与状态；
3. 检查请求访问的模型是否在允许列表中；
4. 校验来源 IP 是否命中允许子网；
5. 检查关联配额计划是否有足够配额；
6. 请求完成后，从响应体中提取 token 使用量并扣除配额。

其中前 5 步在 `HandleFoundProduct` 阶段完成；第 6 步在 `HandleRequestFinish` 阶段完成。模块会将 `ClientApiKey` 与配额计划写入 `AiBasicInfo`，供后续模块使用。

### mod_ai_rate_limit：限流模块

`mod_ai_rate_limit` 对 AI 请求执行分布式限流。它支持基于 Redis 的限流策略，可按产品、API-Key 等维度配置：

- TPM（Tokens Per Minute）：每分钟 token 数上限；
- RPM（Requests Per Minute）：每分钟请求数上限；
- 最大并发数限制。

该模块在 `HandleFoundProduct` 阶段执行，依赖 `mod_ai_token_auth` 设置的 `ClientApiKey` 等信息识别限流维度。若请求触发限流，模块会提前返回响应，阻止请求进入后续转发流程。

### mod_body_process：请求/响应体处理模块

`mod_body_process` 主要负责响应体解析，运行在 `HandleReadResponse` 回调点。对于流式响应（SSE），它负责从响应流中解析 token 使用量，并将统计结果写入 `AiBasicInfo`，供 `mod_ai_token_auth` 在 `HandleRequestFinish` 阶段进行最终配额扣除。

`mod_body_process` 的加载与顺序对 RMB 配额扣减至关重要。若修改了配额扣除逻辑，需要确保流式响应场景在 `mod_body_process` 加载时仍能正确工作。

## 模块框架与回调机制（bfe_module）

BFE 的模块框架定义在 `bfe_module/` 目录下，核心抽象是 `BfeModule` 接口与 `BfeCallbacks` 回调管理器。每个模块通过 `Init()` 方法将自身处理函数注册到指定的回调点，BFE 在请求处理过程中按顺序调用这些回调。

### 回调点

常用的回调点包括：

| 回调点 | 触发时机 | AI 相关模块 |
|--------|----------|-------------|
| `HandleBeforeLocation` | 租户识别之前 | mod_trust_clientip、mod_logid 等 |
| `HandleFoundProduct` | 租户识别之后 | mod_ai_token_auth、mod_ai_rate_limit、mod_ai_route |
| `HandleAfterLocation` | 位置/路由确定之后 | mod_body_process 等 |
| `HandleReadResponse` | 读取后端响应时 | mod_body_process |
| `HandleRequestFinish` | 请求处理完成时 | mod_ai_token_auth（配额扣除） |

### 回调返回值

模块回调函数返回一个 `int` 状态与可选的 `*bfe_http.Response`：

- `BfeHandlerGoOn`：继续执行后续回调；
- `BfeHandlerFinish`：结束请求，返回错误响应；
- `BfeHandlerResponse`：直接跳过后续处理，进入响应发送阶段；
- `BfeHandlerClose`：直接关闭连接；
- `BfeHandlerRedirect`：返回重定向响应。

AI 相关模块在 `HandleFoundProduct` 阶段通常返回 `BfeHandlerGoOn`，将状态写入上下文；若鉴权失败或触发限流，则返回 `BfeHandlerFinish` 或 `BfeHandlerResponse`。

### 模块注册

新增模块的标准步骤如下：

1. 在 `bfe_modules/mod_<name>/` 下创建模块包；
2. 实现 `BfeModule` 接口及所需回调；
3. 在 `bfe_config/` 下新增配置加载器；
4. 在 `bfe_modules/bfe_modules.go` 的 `moduleList` 中按正确顺序注册；
5. 在 `conf/` 下提供默认配置与文档。

模块顺序非常重要。例如 `mod_ai_route` 必须排在 `mod_ai_token_auth` 之后，否则 `ClientApiKey` 尚未设置，路由查找将无法进行。

## 配置加载与热加载机制

BFE 的配置体系分为服务器配置、路由配置、集群配置、TLS 配置与模块配置。AI 相关模块的配置加载遵循统一模式：

1. **模块基础配置**：`conf/mod_<name>/mod_<name>.conf`，采用 INI 格式，指定数据文件路径与日志开关；
2. **模块数据文件**：`conf/mod_<name>/<data>.data`，采用 JSON 格式，存放实际规则数据；
3. **加载入口**：模块的 `Init()` 调用 `ConfLoad()` 加载基础配置，再调用数据加载函数反序列化并校验规则；
4. **热加载**：通过 Web 监控接口触发，重新加载数据文件并原子替换内存结构。

以 `mod_ai_route` 为例，启动时 `Init()` 会：

1. 调用 `ConfLoad()` 读取 `conf/mod_ai_route/mod_ai_route.conf`；
2. 调用 `AiRouteDataLoad()` 读取 `conf/mod_ai_route/ai_route.data`；
3. 调用 `ValidateRouteTable()` 校验每张路由表的类型、规则名、条件表达式、target 权重；
4. 通过 `routeTable.Update()` 原子替换内存中的路由表。

热加载接口为：

```
GET /reload/mod_ai_route
```

该接口调用 `loadRouteRuleConf()`，先在校验阶段生成新的路由表副本，校验通过后再加锁替换，确保失败时不影响当前运行中的规则。`mod_ai_token_auth` 与 `mod_ai_rate_limit` 也提供类似的热加载接口。

热加载的安全性体现在两个层面：一是配置语法与语义校验在替换前完成，避免非法配置进入内存；二是 `AiRouteTable.Update()` 使用读写锁保护，查找操作在 `RLock` 下读取旧表引用，更新操作在 `Lock` 下替换引用，二者互不阻塞。对于路由规则，条件表达式会在加载时由 `condition.Build()` 编译为可执行的 `Condition` 对象，并在 `ValidateRouteTable()` 中校验 target 权重总和为 100，确保运行时无需重复编译。

## 关键配置示例

### 启用 AI 网关模式

在 `bfe/conf/bfe.conf` 中开启 AI 网关：

```ini
[server]
ai_gateway_enabled = true
```

### mod_ai_route 基础配置

`conf/mod_ai_route/mod_ai_route.conf`：

```ini
[basic]
RouteRulePath = ../conf/mod_ai_route/ai_route.data

[log]
OpenDebug = false
```

### mod_ai_route 路由规则

`conf/mod_ai_route/ai_route.data`：

```json
{
    "Version": "20260718131505",
    "route_rules": {
        "apikey_ak_user_a": {
            "type": "apikey",
            "owner": "ak_user_a",
            "rules": [
                {
                    "name": "user_a-deepseek",
                    "Cond": "req_host_in(\"api.example.org\")",
                    "targets": [
                        {
                            "ClusterName": "cluster_deepseek_a",
                            "Model": "deepseek-v4-pro",
                            "Weight": 70
                        },
                        {
                            "ClusterName": "cluster_deepseek_b",
                            "Model": "deepseek-v4-pro",
                            "Weight": 30
                        }
                    ],
                    "fallbacks": [
                        {
                            "ClusterName": "cluster_deepseek_c",
                            "Model": "deepseek-v3.2"
                        }
                    ]
                }
            ]
        },
        "entity_dept_ai": {
            "type": "entity",
            "owner": "dept_ai",
            "rules": [
                {
                    "name": "dept_ai-default",
                    "Cond": "default_t()",
                    "targets": [
                        {
                            "ClusterName": "cluster_dept_ai",
                            "Model": "",
                            "Weight": 100
                        }
                    ],
                    "fallbacks": []
                }
            ]
        },
        "global_default": {
            "type": "global",
            "owner": "global",
            "rules": [
                {
                    "name": "global-default",
                    "Cond": "default_t()",
                    "targets": [
                        {
                            "ClusterName": "cluster_global",
                            "Model": "",
                            "Weight": 100
                        }
                    ],
                    "fallbacks": []
                }
            ]
        }
    },
    "ApikeyRouteTableBindings": {
        "ak_user_a": [
            "apikey_ak_user_a",
            "entity_dept_ai",
            "global_default"
        ]
    }
}
```

上述配置表示：API-Key 为 `ak_user_a` 的请求，先查找 `apikey_ak_user_a` 路由表；未命中则查找 `entity_dept_ai`；仍未命中则使用 `global_default`。命中 `user_a-deepseek` 规则后，按 70:30 的权重选择 `cluster_deepseek_a` 或 `cluster_deepseek_b`，并在两者均失败时降级到 `cluster_deepseek_c`。

### fallback 触发条件

`ServeHTTPForAI()` 在目标转发失败时按 `fallbacks` 顺序降级。触发条件为：

- `clusterInvoke()` 返回错误（连接失败、超时、读写错误等）；
- 后端返回状态码 `>= 500`。

以下情况不触发 fallback：

- 后端返回 `4xx` 客户端错误；
- 请求在 `HandleFoundProduct` 阶段已被限流或鉴权失败。

每次 fallback 前，`resetRequestForRetry()` 会重置 `OutRequest`、backend 连接、retry 计数与错误信息，并通过 `rewindRequestBody()` 将请求体重置到起始位置，确保下一次转发使用干净的请求状态。

### fallback 执行流程

```
加权选择 target
    │
    ▼
构造尝试列表 [target] + fallbacks
    │
    ▼
prepareRequestBodyForRetry()
若请求体不可回退 → 禁用 fallback
    │
    ▼
┌─────────────────────────────────┐
│  循环遍历 attempts 列表          │
│                                 │
│  首次？ 否 → resetRequestForRetry()│
│  ClusterTable.Lookup(ClusterName)│
│  模型覆盖 / ModelMapping         │
│  clusterInvoke()                │
│                                 │
│  成功（err == nil 且 status < 500）│
│      → 跳出循环                  │
│  失败且非最后一次                │
│      → shouldTriggerFallback()?  │
│         是 → 关闭响应，继续下一个   │
│         否 → 跳出循环             │
└─────────────────────────────────┘
    │
    ▼
返回最终响应或 500
```

该流程确保只有在后端不可用或出现服务端错误时才尝试降级，而客户端错误不会触发无意义的 fallback，避免将错误请求扩散到备用集群。

## 本章小结

本章介绍了壬远 AI 网关数据面组件 BFE 的设计。

- BFE 负责实际转发 AI 请求，与控制面 AI Gateway API 通过配置分发协同工作。
- AI 网关模式下，请求进入独立的 `ServeHTTPForAI()` 路径，复用原有回调与转发基础设施。
- `mod_ai_token_auth`、`mod_ai_rate_limit`、`mod_ai_route` 三个模块在 `HandleFoundProduct` 阶段按固定顺序执行，通过 `AiBasicInfo` 与 `Request.Context` 共享状态。
- `mod_body_process` 在 `HandleReadResponse` 阶段解析 SSE 响应中的 token 使用量，供 `mod_ai_token_auth` 最终扣减配额。
- BFE 的 `bfe_module` 框架通过回调点与返回值机制组织模块，`bfe_modules/bfe_modules.go` 中的注册顺序直接影响行为正确性。
- 配置加载采用 INI + JSON 双层结构，支持通过 Web 接口热加载，新配置在校验完成后原子替换旧配置。
- `mod_ai_route` 支持 `apikey → entity → global` 三级路由、`targets` 加权选择与 `fallbacks` 顺序降级，是 AI 网关转发的核心。

## 参考文档

- `bfe/AGENTS.md`
- `bfe/docs/zh_cn/modules/mod_ai_route/mod_ai_route.md`
- `bfe/docs/zh_cn/modules/mod_ai_token_auth/mod_ai_token_auth.md`
- `bfe/docs/zh_cn/modules/mod_ai_rate_limit/mod_ai_rate_limit.md`
- `bfe/docs/zh_cn/sys_design/mod_ai_route.md`
- `bfe/docs/zh_cn/sys_design/mod_ai_route_bfe_changes.md`
