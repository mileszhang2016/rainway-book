# 第二十六章 EPP调度配置与运维

## 本章目标

通过本章，读者将掌握：

- 使用 EPP 智能调度的前提条件与部署形态；
- 如何在 Dashboard 的「EPP调度」页面配置 EPP 实例池，并创建 EPP 模式 Cluster（`balance_mode=EPP` + `epp_config`）；
- 如何查看与手工覆写 Cluster→实例组分配（EPP 调度分配页签）；
- 摘流后端、上下线 EPP 实例、EPP 故障切换、全过载兜底、关闭 EPP 调度等日常运维场景的操作方法；
- EPP 链路的观测指标与告警要点；
- `epp_config` 与实例池的校验规则速查。

EPP 调度的工作原理与设计语义见 [第十六章 EPP智能调度设计](../design/chapter16-epp-scheduling-design.md)。本章控制台操作以 Dashboard 当前界面为准，权威用户手册见 `ai-gateway-web/docs/zh-cn/06-epp-schedule.md`。

---

## 前提与部署形态

使用 EPP 智能调度前，确认以下组件已随部署安装并运行：

- **ai-gateway-epp（EPP 组件）**：以实例组为单位部署，每组恰好 1 个实例：实例池校验拒绝空组与多实例组（422 `requires exactly 1 instance`）；`instances` 保留数组形态，为未来多实例扩容预留。实例经命令行参数运行，关键参数如下：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-group-name` | 环境变量 `AI_GATEWAY_EPP_GROUP_NAME`（部分设计文档亦记作 epp-group / AI_GATEWAY_EPP_GROUP，以 ai-gateway-epp 实现为准） | 实例组标识，必须与实例池中登记的组名一致；EppDataWatcher 以它在 assignment 全量视图中匹配本实例所属组承担的 Cluster。分配粒度为实例组（每组恰好 1 个实例），因此 ai-gateway-epp、AI Gateway API 与 Dashboard 需同版本发布 |
| `-api-addr` | `http://127.0.0.1:8181/inner-api/v1` | AI Gateway API 的 InnerAPI 基础地址 |
| `-api-token` | 环境变量 `AI_GATEWAY_EPP_TOKEN` | InnerAPI 鉴权 Token |
| `-poll-interval` | `5s` | InnerAPI 轮询间隔 |
| `-poll-timeout` | `3s` | 单次轮询请求超时 |
| `-grpc-port` | `9002` | ext_proc gRPC 服务端口（BFE 通过 `EPPAddr` 访问此端口） |
| `-health-port` | `9003` | gRPC 健康检查端口（BFE `EPPCheck` 探活来源） |
| `-metrics-port` | `9090` | Prometheus 指标端口 |
| `-grpc-tls-cert` / `-grpc-tls-key` | 空（明文） | ext_proc 与 health gRPC 服务端 TLS 证书；生产环境建议开启，BFE 侧以 `EPPTLS.CAFile` 校验 |
| `-refresh-metrics-interval` | `50ms` | 后端 `/metrics` 指标抓取周期 |
| `-local-config-dir` | 环境变量 `AI_GATEWAY_EPP_LOCAL_CONFIG_DIR`（缺省空） | 非空时从本地目录的 JSON 文件（`cluster_table.json`、`epp_data_config.json`）加载集群级调度配置，不连接 InnerAPI，用于本地调试与集成测试；每个轮询周期重读文件，修改在下一个周期生效（无需重启），文件缺失或 JSON 非法时沿用上一份成功配置 |

发布二进制可通过 `ai-gateway-epp` 仓库的 `make release` 交叉编译并打包 linux/amd64 与 linux/arm64 的 tarball。

- **BFE**：经 `GslbBasic.BalanceMode=EPP` 与单元素 `EPPAddr=[实例组内实例地址]` 访问 EPP，相关配置由 AI Gateway API 经 server_data_conf 自动下发，Conf Agent 完成热加载，无需手工编辑。
- **AI Gateway API**：OpenAPI 与 InnerAPI 正常运行；EPP 实例经 InnerAPI 轮询 `epp_data` 与 `cluster_table`，鉴权与版本增量机制与其他数据面组件一致。

链路关系如下图：

```mermaid
flowchart LR
    OP[运维<br/>Dashboard EPP调度页] --> API[AI Gateway API]
    API -->|server_data_conf<br/>BalanceMode=EPP + EPPAddr| CA[Conf Agent]
    CA --> BFE[BFE]
    API -->|epp_data 调度配置+分配<br/>cluster_table 后端列表| EPP[ai-gateway-epp 实例组]
    BFE -->|ext_proc| EPP
    BFE -->|兜底 WRR| RS[推理后端]
    EPP -->|/metrics| RS
```

---

## 控制台入口：资源管理 → EPP调度

Dashboard 中 EPP 调度的入口为 **资源管理 → EPP调度**。页面分为两个页签：

- **EPP 实例池**：登记 EPP 调度器自身的部署实例（按组组织，每组 1 个实例）；
- **EPP 调度分配**：查看 / 覆写「EPP 模式集群 ↔ 实例组」的分配关系。

以下两节分别说明两个页签的操作。

---

## 配置 EPP 实例池（EPP 实例池页签）

### 查看模式

页签顶部显示统计信息：**池名称**（如 `EPP.pool`，由服务端配置提供）、**实例组数**、**实例总数**。

主体为树形表格，按「实例组」一行一组：

| 列名 | 说明 |
|------|------|
| 实例组 | 组名 |
| 实例数 | 组内实例数量，显示格式 `N / 1`（当前恒为 `1 / 1`） |
| 实例列表 | 摘要展示 `实例ID(host:port)`，灰色小字 |

点击行首箭头展开组，内嵌表格显示每个实例的**实例 ID / 主机 / 端口**。

### 编辑模式

点击右上角「编辑」进入行内编辑。「保存」全量提交——注意其为**全量替换**语义，未列出的组与实例会被删除；「取消」还原本次全部修改。

| 操作 | 说明 |
|------|------|
| 编辑组名 | 行内输入，组名非空且**池内唯一** |
| + 添加组 | 在表格下方新增空组（须至少添加 1 个实例才能保存） |
| 删除组 | 需确认「删除实例组及其所有实例」；至少保留 1 个组（仅剩 1 组时按钮禁用） |
| +添加实例 | 在展开区内添加实例行，端口默认 `9002`；组内已有 1 个实例时按钮禁用，右侧灰色提示「每组恰好 1 个实例」 |
| 删除 | 移除组内实例行 |

**校验规则**（控制台与 OpenAPI 共用同一套服务端校验）：

| 规则 | 说明 |
|------|------|
| 组 | 至少 1 个组；组名非空、池内唯一；每组恰好 1 个实例（拒绝空组与多实例组，OpenAPI 返回 422） |
| 实例 ID | 非空且**池内全局唯一**；对应实例在组内的标识（K8s StatefulSet 部署时取 Pod 名，如 `epp-0`） |
| 主机 | 合法主机名或 IPv4 / IPv6 地址 |
| 端口 | 1-65535 整数 |
| 地址唯一 | `主机:端口` 组合池内全局唯一 |

> **部署形态**：每组恰好 **1 个实例**；`instances` 保留数组形态，为未来多实例组扩容预留。

### 保存后的自动修复

实例池保存为全量替换。若既有分配**所属的实例组已被移出池**（实例随组移除），系统自动修复悬空分配：

- 对整个实例池重跑分配器重新选组（跨组重分配）；
- 池内无可分配候选组时，清除该分配（集群进入「未分配」态，见下节）。

### 附录：OpenAPI 等价操作

实例池是单例资源，提供 `GET`（详情）与 `PATCH`（全量替换）两个操作，与页签的查看 / 保存一一对应。实例组与实例列表的权威来源是部署流程，扩缩容、换机后由部署流程 reconcile 调用 `PATCH`。

以下示例登记一个实例组 `g1`，含一个实例 `epp-0`（host 为实例 IP，`port` 为 EPP 的 ext_proc gRPC 端口）：

```bash
curl -X PATCH "https://control-plane.example.com/open-api/v1/epp-pool" \
  -H "Content-Type: application/json" \
  -d '{
    "groups": [
        {
            "name": "g1",
            "instances": [
                { "id": "epp-0", "host": "10.0.0.1", "port": 9002 }
            ]
        }
    ]
}'
```

查看当前实例池：

```bash
curl -X GET "https://control-plane.example.com/open-api/v1/epp-pool"
```

---

## 创建 EPP 模式 Cluster

集群的负载均衡模式在集群向导步骤 4 的「均衡模式配置」卡片中设置（`WRR` / `EPP` 切换及 `epp_config` 参数），控制台操作流程见 [第二十一章 Cluster与路由配置](./chapter21-cluster-and-route-config.md) 与 `ai-gateway-web/docs/zh-cn/03-ai-business-cluster.md` 的「均衡模式配置」「EPP 运行时参数」小节。选择 `EPP` 后按需填写负载画像、亲和强度、前缀缓存亲和性、会话亲和性、准入水位、坏后端剔除、流控配置与 EPP 运行时参数（`epp_runtime` 未配置时由 BFE 补默认，复查页显示「未配置（BFE 补默认）」）。

等价地，经 OpenAPI 创建 Cluster 时指定 `balance_mode=EPP` 并携带 `epp_config`：

```bash
curl -X POST "https://control-plane.example.com/open-api/v1/clusters" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "llm-cluster-a",
    "basic": { "protocol": "http" },
    "llm_config": { "provider": "p1", "models": ["m1"] },
    "balance_mode": "EPP",
    "epp_config": {
        "load_profile": "balanced",
        "affinity": "medium",
        "kv_cache_utilization_max": 0.9,
        "waiting_queue_max": 32,
        "running_requests_max": 256,
        "fallback_on_empty": false,
        "metrics_staleness_threshold_ms": 200,
        "prefix_cache_affinity": true,
        "session_affinity_enabled": true,
        "session_affinity_header": "x-session-id",
        "flow_control": {
            "max_requests": 1000,
            "queue_ttl": 30,
            "no_endpoint_queue_ttl": 600
        }
    },
    "epp_runtime": {
        "check": { "check_interval": "2s", "fail_threshold": 3, "cooldown": "45s", "success_threshold": 2 },
        "timeout": { "connect": "500ms", "call": "3s" }
    }
}'
```

创建成功后，系统自动完成：校验 `epp_config` / `epp_runtime` → 贪心分配器选组 → 写入 `epp_assignments` → 下次导出时 server_data_conf 带出 `BalanceMode=EPP`、单元素 `EPPAddr` 与 `EPPCheck` / `EPPTimeout`，epp_data 带出编译后的调度配置与分配条目。

`epp_config` 字段说明：

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `load_profile` | `balanced` | 负载画像：`queue-first` / `balanced` / `kv-first` |
| `affinity` | `medium` | 亲和强度：`off` / `low` / `medium` / `high`，映射为前缀/会话亲和 scorer 权重 0 / 0.3 / 0.6 / 1.0；取 `off` 时不注入亲和 scorer |
| `prefix_cache_affinity` | `true` | 前缀缓存亲和开关（软亲和，不保证命中匹配后端） |
| `session_affinity_enabled` | `false` | 会话亲和开关 |
| `session_affinity_header` | - | session id 来源请求头；`enabled=true` 时必填，二者成对配置 |
| `kv_cache_utilization_max` | `0.9` | KV cache 利用率过滤阈值 `(0,1]` |
| `waiting_queue_max` | `0`（不启用） | 等待队列长度阈值 `≥0`，大于 0 时启用对应过滤条件 |
| `running_requests_max` | `0`（不启用） | 在跑请求数阈值 `≥0`，大于 0 时启用对应过滤条件 |
| `fallback_on_empty` | `false` | 全部端点被过滤时是否回退放行（透传 utilization-filter 的 fallbackOnEmpty） |
| `metrics_staleness_threshold_ms` | `200` | 坏后端剔除阈值（毫秒 `>0`）：指标抓取连续失败、数据过期超过该值的后端被剔除 |
| `flow_control.max_requests` | 不限 | 全局并发上限；`>0` 或 `-1`（显式不限） |
| `flow_control.queue_ttl` | EPP 默认 60 秒 | 池有端点时的排队预算（秒），超期以可重试背压错误拒绝 |
| `flow_control.no_endpoint_queue_ttl` | 跟随 `queue_ttl` | 池无端点时的排队预算（秒） |
| `flow_control.enable_eviction` | `false` | 仅接受 `false`（写入 `true` 校验拒绝 422） |

`epp_runtime` 字段说明（可选；面向 BFE 侧 EPP 调用链路的运行时参数，与 `epp_config` 调度配置分层）：

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `check.disabled` | `false` | 是否禁用 EPP 地址后台健康检查 |
| `check.check_interval` | `"2s"` | 健康检查间隔 |
| `check.fail_threshold` | `3` | 活跃地址连续失败次数阈值，超过即视为不可用 |
| `check.cooldown` | `"45s"` | 冷却期，防止状态抖动 |
| `check.success_threshold` | `2` | 冷却期后地址恢复所需连续成功次数 |
| `timeout.connect` | `"500ms"` | EPP gRPC 连接超时 |
| `timeout.call` | `"3s"` | EPP 调用超时 |

`epp_runtime` 注意事项：

- 严格字段解析：出现 `tls` / `breaker` 等未定义字段返回 422——TLS 与熔断本期不开放，导出时固定 `EPPTLS{Plaintext:true}`、`EPPBreaker{Disabled:true}`；
- duration 字段必须为正；写入时未显式携带的子字段按默认值补齐后落盘，GET 回读与落盘一致；
- 仅 `balance_mode=EPP` 且分配有效时编译导出（`EPPCheck` / `EPPTimeout`）；休眠或未分配 Cluster 不导出。

注意事项：

- `balance_mode=EPP` 时 `epp_config` 必填且须通过字段校验，否则创建返回 422；
- EPP 模式 Cluster 不创建 `Role=COMMON` 的 BFE 实例池，但池实例列表照常同步 Provider 的 `instance_pool`——这些实例经 cluster_table 供 EPP 发现后端，并作为 EPP 全挂时 BFE 兜底 WRR 的后端；
- `llm_config`（provider/models/keys）的校验与均衡模式无关，照常执行；
- `epp_config` 非空即须字段合法，与 `balance_mode` 无关：`WRR → EPP` 须同时提供合法配置，`EPP → WRR` 后配置休眠保留（不编译、不导出），再切回 `EPP` 时继续生效。

---

## 查看与覆写分配（EPP 调度分配页签）

「集群 ↔ 实例组」的分配由**分配器自动生成**：集群切入 EPP 模式时自动选择实例组；实例池变更后自动修复悬空分配。本页签提供**查询视图**与**手工覆写**（运维干预入口）。

### 统计与分配视图

页签顶部显示统计：**已分配集群** / **未分配集群**（数量大于 0 时红色告警）/ **空闲实例组**。

分配视图表格：

| 列名 | 说明 |
|------|------|
| 集群名称 | 支持搜索、排序 |
| 实例组 | 分配到的组名；未分配时显示 `-` |
| 操作 | 「编辑」打开手动覆写抽屉 |

### 未分配集群 / 空闲实例组

- **未分配集群**（红色卡片）：EPP 模式但无有效分配的集群（含分配悬空——所属组已被移出池），提示「服务降级为 WRR」。这些集群导出配置时会**降级为 WRR** 转发，属需尽快处理的异常态。
- **空闲实例组**：实例池中未承担任何集群分配的组（如扩容新组后、完成分配前）。

> 同一实例组承担多个集群是**合法**的——分配以集群为单位，不做组与集群的一对一约束。

### 手动覆写

点击行内「编辑」打开抽屉，仅需选择**实例组**（下拉选项来自当前实例池中的组）。

确认后仅覆写该集群的分配：归属实例组取所选组。覆写后 server_data_conf 版本即时 bump，BFE 下一次拉取即切换到新实例组地址。

### 附录：OpenAPI 等价操作

查看分配全量视图：

```bash
curl -X GET "https://control-plane.example.com/open-api/v1/epp-assignments"
```

返回示例：

```json
{
    "ErrNum": 200,
    "ErrMsg": "success",
    "Data": {
        "clusters": [
            {
                "cluster": "llm-cluster-a",
                "group": "g1"
            }
        ],
        "unassigned_clusters": [],
        "idle_groups": []
    }
}
```

字段含义：视图中每个元素只含 `cluster` 与 `group`（实例详情由 `/epp-pool` 按组名派生）；`unassigned_clusters` 列出 EPP 模式但无有效分配的 Cluster（含分配悬空——所属组已被移出池；属需尽快处理的异常态，导出 server_data_conf 时降级为 WRR）；`idle_groups` 列出未承担任何分配的组。也可用 `?cluster=<name>` 过滤单个 Cluster。

需要运维干预（如故障转移确认、特殊布局）时，可手工覆写某个 Cluster 的分配：

```bash
curl -X PUT "https://control-plane.example.com/open-api/v1/epp-assignments/llm-cluster-a" \
  -H "Content-Type: application/json" \
  -d '{
    "group_name": "g1"
}'
```

`group_name` 必须存在于当前实例池。

---

## 注意事项

- 手工覆写要求：实例组存在于实例池中。
- 发版约束：ai-gateway-epp、AI Gateway API（含 `epp_assignments` 表结构，其中不含 `primary_instance_id` 列）与 Dashboard 必须**同版本同步发布**；混部时 EPP 全部 no-match 显式告警，不会静默错误。
- 集群由 `EPP` 切回 `WRR` 后，分配记录**休眠保留**，再切回 `EPP` 时继续生效。
- EPP 模式生效前提：已部署 EPP 调度器并在实例池登记实例。未分配集群会降级为 WRR，请在「EPP 调度分配」页签巡检未分配告警。
- 实例池「保存」为**全量替换**语义，未列出的组与实例会被删除，请整表核对后提交。

---

## 日常运维场景

### 摘流某个后端

将 Provider 对应实例的 `weight` 置 0，实例即从所有引用该 Provider 的 Cluster（含 EPP 池）中摘除：

```bash
curl -X PATCH "https://control-plane.example.com/open-api/v1/providers/p1" \
  -H "Content-Type: application/json" \
  -d '{ "instance_pool": [ ... 目标实例 weight=0 ... ] }'
```

链路：`ProviderInstancePoolSyncer` 同步实例池 → cluster_table 导出更新 → EPP 的 cluster-table-discovery 在下一个轮询周期摘除（秒级，跳过 `Weight=0` 实例）→ BFE 兜底 WRR 同样不再选中该实例。恢复时把 weight 置回正数即可。摘流期间 EPP 的 utilization-filter 与亲和逻辑不受影响，流量自动收敛到剩余后端。

### 上下线 EPP 实例

在「EPP 实例池」页签进入编辑模式，调整组与实例后保存（**全量替换**语义，未列出的组与实例会被删除，请整表核对）；等价地，也可经 `PATCH /epp-pool` 全量替换实例列表：

- **下线实例**：实例随组移除——在编辑模式中删除该实例组并保存。其承担的分配由分配器自动修复：跨组重分配；池无可分配候选组时清除分配（该 Cluster 进入未分配态，导出降级 WRR + error 日志）。
- **上线实例**：新增实例组并登记实例。新实例以 `-group-name` 组身份在 assignment 全量视图中匹配：组被分配即承担对应 Cluster 的调度。
- **替换实例（保组名）**：保持组名不变、替换组内实例条目，分配不失效，server_data_conf 下次导出时 `EPPAddr` 平滑切换到新实例地址。
- 周期对账 reconciler（30 秒）会兜底扫描无有效分配的 EPP Cluster 并自动补分配，大部分轮次零写入。

### EPP 实例故障

EPP 实例故障时无需人工介入即可保持业务不中断：

1. BFE 的 `EPPCheck` 后台健康检查发现实例地址连续失败达到 `FailThreshold`（默认 3 次，探测间隔默认 2 秒，检测约 6~10 秒），该地址即视为不可用。
2. `EPPAddr` 恒为单元素（每组 1 实例），无下一地址可切换：EPP 路径失败，BFE **静默回退本地 WRR**，请求仍返回 200。
3. 实例恢复后，冷却期（`Cooldown`，默认 45 秒）满且连续通过 `SuccessThreshold`（默认 2 次）健康检查，BFE 自动恢复 EPP 调用，防止 flapping。

注意：当前不存在主备角色（每组恰好 1 实例），因此没有 failover 到备实例的语义。实例长期不可用时的恢复手段是控制面重分配——周期对账 reconciler 会自动把悬空分配补到健康组，也可经「EPP 调度分配」页签手工覆写（`PUT /epp-assignments/{cluster}`）把 Cluster 分配到其他实例组（覆写后 BFE 拉取新 `EPPAddr` 即切换）。

故障切换的时序如下：

```mermaid
sequenceDiagram
    participant BFE
    participant E0 as ai-gateway-epp（实例组 g1）

    BFE->>E0: ext_proc 调度请求
    E0-->>BFE: 选中端点
    Note over E0: 实例进程故障
    loop EPPCheck 健康检查（间隔 2s）
        BFE->>E0: grpc.health.v1 Check
        E0--xBFE: 无响应（连续失败 3 次）
    end
    BFE->>BFE: 地址不可用，静默回退本地 WRR，请求继续返回 200
    Note over BFE: 实例恢复后，冷却期满且连续 2 次<br/>健康检查通过即恢复 EPP 调用
```

### 全过载兜底

所有后端 KV cache 利用率都超过 `kv_cache_utilization_max` 时，EPP 的 utilization-filter 过滤全部端点（fail-closed），无候选返回，BFE 回退本地 WRR 继续服务，业务不中断。可通过 `epp_fallback_local_total` 增长确认兜底发生。指标管道有抓取周期（`RefreshMetricsInterval`），利用率恢复后 EPP 在下一周期重新放行端点。

### 关闭 EPP 调度

将 Cluster 的 `balance_mode` 改回 `WRR` 即可（控制台在集群向导「均衡模式配置」卡片切换，或经 OpenAPI 修改），无需清空 `epp_config`（休眠保留，随时可切回）：

```bash
curl -X PUT "https://control-plane.example.com/open-api/v1/clusters/llm-cluster-a" \
  -H "Content-Type: application/json" \
  -d '{ "balance_mode": "WRR" }'
```

导出语义：`BalanceMode=WRR`、`EPPAddr` 为空；epp_config 与分配记录保留但不进 epp_data 导出；分配记录休眠保留，再切回 `EPP` 时原配置原分配继续生效。

---

## 观测与告警

| 观测点 | 位置 | 关键指标/日志 | 告警建议 |
|--------|------|--------------|----------|
| 未分配告警 | Dashboard「EPP 调度分配」页签 | 顶部「未分配集群」统计大于 0 时红色告警 | 定期巡检未分配集群卡片，出现即处理 |
| 降级导出 | AI Gateway API error 日志 | EPP 模式 Cluster 无有效分配时输出 error 级日志（含 Cluster 名与原因），导出降级为该 Cluster `BalanceMode=WRR` | error 日志出现即告警；配合未分配集群巡检 |
| BFE EPP 调用 | BFE 监控端口 `/monitor/epp_metrics` | `epp_calls_total{cluster,result}`（result ∈ ok/no_pool/unknown_pool/draining/transport）、`epp_fallback_local_total{cluster}`、`epp_failover_total{cluster}`、`epp_failback_total{cluster}`、`epp_active_addr_index{cluster}` | `epp_calls_total{result!="ok"}` 突增、failover 计数增长、`epp_fallback_local_total` 持续增长均提示 EPP 链路异常 |
| EPP 实例 | EPP `/metrics`（默认 `:9090`） | `ai_epp_assignment_no_match`（本实例所属组在分配中无任何条目）、`ai_epp_poller_failures_total` / `ai_epp_poller_backoff_state`（轮询失败与退避）、`ai_epp_engine_reloads_total`（引擎热加载次数）、`ai_epp_cell_state`（Cell 状态） | `assignment_no_match=1` 说明 `-group-name` 与实例池登记不一致，部署核对；poller 失败持续非零说明控制面不可达 |
| 请求链路关联 | EPP 日志（demux） | EPP 日志携带 `x-request-id`：demux 从首个 ext_proc 消息的请求头读取该值（网关注入则透传），未携带时生成 UUID 并回写到请求头，llm-d 引擎日志使用同一 request ID | 排障时按 `x-request-id` 串联 BFE → EPP → llm-d 引擎三方日志，定位单请求全链路 |

判断请求是否真正经过 EPP 调度（而非兜底 WRR）：BFE 对 EPP 失败静默回退也返回 200，因此应以 `epp_calls_total{result="ok"}` 是否增长为准，不能只看响应码。

---

## 校验规则速查

**实例池（控制台「EPP 实例池」页签 / `PATCH /epp-pool`）**

- 至少 1 个组；组名非空、池内唯一；拒绝空组；
- 每组恰好 1 个实例，拒绝多实例组（OpenAPI 返回 422 `requires exactly 1 instance`）；
- 实例 id 非空、池内全局唯一；
- `host` 为合法主机名或 IPv4 / IPv6 地址，`port` 为 1-65535 整数；
- `(host, port)` 组合池内全局唯一；
- 保存为全量替换语义，未列出的组与实例会被删除。

**`/clusters`（balance_mode、epp_config 与 epp_runtime）**

- `balance_mode` 枚举 `WRR` / `EPP`，默认 `WRR`；
- `balance_mode=EPP` 时 `epp_config` 必填；`epp_config` 非空即须通过字段校验（与 balance_mode 无关）；
- `epp_config` / `epp_runtime` 均按严格字段解析，未知字段返回 422（`epp_runtime` 出现 `tls` / `breaker` 字段同样 422）；
- `load_profile` ∈ `queue-first` / `balanced` / `kv-first`，默认 `balanced`；
- `affinity` ∈ `off` / `low` / `medium` / `high`，默认 `medium`；
- `prefix_cache_affinity` / `session_affinity_enabled` / `fallback_on_empty` 为 bool；
- `session_affinity_enabled=true` 时 `session_affinity_header` 必填（合法 HTTP header 名），二者成对；
- `kv_cache_utilization_max` ∈ `(0, 1]`，默认 `0.9`；
- `waiting_queue_max` / `running_requests_max` ≥ 0，默认 `0`（不启用）；
- `metrics_staleness_threshold_ms` > 0，默认 `200`；
- `flow_control.max_requests` `>0` 或 `-1`（缺省不限）；
- `flow_control.queue_ttl` / `no_endpoint_queue_ttl` 为 ≥0 整数（秒）；
- `flow_control.enable_eviction` 仅接受 `false`，写入 `true` 返回 422；
- `epp_runtime.check.disabled` 为 bool，默认 `false`；`check_interval` / `cooldown` / `connect` / `call` 为正的 duration 字符串，默认值分别为 `"2s"` / `"45s"` / `"500ms"` / `"3s"`；
- `epp_runtime.check.fail_threshold` / `success_threshold` 为正整数，默认 `3` / `2`。

**手工覆写（「EPP 调度分配」页签 / `PUT /epp-assignments/{cluster}`）**

- 实例组必须存在于当前实例池。

---

## 本章小结

- EPP 调度入口为 Dashboard「资源管理 → EPP调度」，分「EPP 实例池」与「EPP 调度分配」两个页签；实例池保存为全量替换语义，组名与 EPP 的 `-group-name` 一致；EPP 组件以 `-api-addr`、`-poll-interval`、`-grpc-tls-cert` 等参数运行。
- 「EPP 实例池」页签提供树形表格的查看与行内编辑：每组恰好 1 个实例（数组形态为扩容预留），校验组名唯一、实例 id 与 `主机:端口` 全局唯一；保存时自动修复悬空分配（重跑分配器重选组 / 无可分配候选组时清除分配）。
- 「EPP 调度分配」页签展示分配器自动生成的「集群 ↔ 实例组」分配：顶部统计已分配 / 未分配（红色告警）/ 空闲组，表格展示集群与所属实例组，支持手工覆写；未分配集群导出降级为 WRR。
- 创建 EPP 模式 Cluster 即在集群向导「均衡模式配置」卡片选择 `EPP` 并填写 `epp_config`（OpenAPI：`POST /clusters` 携带 `balance_mode=EPP`）；系统自动分配实例组。
- 摘流后端 = Provider 实例 `weight=0`，cluster_table 导出与 EPP discovery 秒级摘除；上下线 EPP 实例 = 在「EPP 实例池」页签编辑保存，悬空分配自动修复。
- EPP 实例故障由 BFE `EPPCheck` 标记不可用，兜底流量由 BFE 本地 WRR 承接；恢复由控制面重分配完成（reconciler 自动 / 手工覆写），BFE 下次拉取新 `EPPAddr` 即切回 EPP 路径；全过载时 EPP 无候选、BFE 兜底，业务不中断。
- 关闭 EPP 调度只需切回 `WRR`，epp_config 与分配休眠保留，可再切回。
- 观测覆盖三层：Dashboard 未分配告警、AI Gateway API error 日志（降级导出）、BFE `/monitor/epp_metrics`（`epp_calls_total` / `epp_fallback_local_total`）、EPP `/metrics`（`ai_epp_assignment_no_match` 等）；EPP 日志携带 `x-request-id`，可按 request ID 串联 BFE → EPP → llm-d 引擎三方日志。

---

## 参考文档

- `ai-gateway-web/docs/zh-cn/06-epp-schedule.md`（EPP 调度控制台用户手册，本章控制台操作的权威依据）
- `ai-gateway-web/docs/zh-cn/03-ai-business-cluster.md`（「均衡模式配置」与「EPP 运行时参数」小节）
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/epp-pool.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/epp-assignments.md`
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/clusters.md`
- `ai-gateway-api/design-docs/api-define/InnerAPI接口定义/epp-data.md`
- `ai-gateway-api/design-docs/api-define/InnerAPI接口定义/server-data-conf.md`
- `ai-gateway-api/design-docs/sys-design/details/EPP调度对接.md`
- `ai-gateway-epp/cmd/epp/config.go`
- [第十六章 EPP智能调度设计](../design/chapter16-epp-scheduling-design.md)
- [第二十一章 Cluster与路由配置](./chapter21-cluster-and-route-config.md)
