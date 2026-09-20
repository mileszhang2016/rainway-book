# Chapter 15: Observability Design

## Chapter Goals

An AI Gateway carries a large volume of large-model call traffic, and the request path involves multiple stages: authentication, routing, rate limiting, quota deduction, upstream forwarding, and streaming responses. An anomaly in any of these stages can affect downstream business and cause billing disputes. This chapter introduces the observability system of the Rainway AI Gateway and helps readers understand the following questions:

- How the three pillars of observability (logs, metrics, and traces) are implemented in the AI Gateway;
- Which AI-specific fields are included in the BFE Data Plane access logs, and how these fields are collected;
- Which key monitoring metrics should be tracked, covering route hits, quota hits, and rate limit triggers;
- How to integrate BFE with monitoring systems such as Prometheus and Zabbix;
- The classification of the AI Gateway error code system and troubleshooting approaches;
- Recommended alert configurations based on logs and metrics;
- The two deployment forms of the reporting system (lightweight form and standard form) and the differences in their data pipelines;
- The capabilities and key design of the built-in report query API (`/report/*`): query-layer abstraction, minute-level aggregation, partition management, and dual-backend consistency.

After reading this chapter, readers should be able to independently plan, configure, and troubleshoot the observability solution of the AI Gateway.

## The Three Pillars of Observability

The industry typically divides observability into three pillars: Logs, Metrics, and Traces. The Rainway AI Gateway has made targeted designs for all three pillars in both the Data Plane (BFE) and the Control Plane (AI Gateway API).

### Logs

Logs are used to record the complete lifecycle of every request. The BFE Data Plane outputs key information about each stage of a request — authentication, routing, forwarding, and billing — through the Access Log, enabling downstream troubleshooting, billing reconciliation, and security auditing. AI-specific fields uniformly occupy field numbers 701-900 of the `bfe-access-pb` protocol; 29 fields are currently defined. For details, see [BFE AI Access Log Observability Fields Design](../../../bfe/docs/zh_cn/sys_design/ai_access_log_fields.md).

### Metrics

Metrics are used to quantify system operation status and support trend analysis and threshold-based alerting. Each BFE AI module exposes counter-style monitoring items, such as `REQ_TOTAL` (total requests), `REQ_HIT_APIKEY` (requests hitting apikey routes), and `REQ_HIT_ENTITY` (requests hitting entity routes). These metrics can be collected by systems such as Prometheus and Zabbix through BFE's built-in monitoring interface.

### Traces

Traces are used to depict the complete call path of a request in a distributed system. Although the current BFE access log already records fields such as `ai_cluster_key_names` to reflect the clusters and keys a request has tried, for scenarios that require cross-service latency bottleneck analysis, it is recommended to extend with distributed tracing frameworks such as OpenTelemetry. Traces complement logs and metrics, forming a three-dimensional observability capability.

```mermaid
graph LR
    A[Client Request] --> B[BFE Data Plane]
    B --> C[Access Log]
    B --> D[Module Metrics]
    B --> E[Distributed Tracing]
    C --> F[Billing/Troubleshooting/Audit]
    D --> G[Prometheus / Zabbix]
    E --> H[Trace Analysis]
```

## BFE Access Log Fields and AI-Specific Fields

When BFE serves as an AI Gateway, the access log no longer contains only traditional HTTP fields (such as status code, response size, and latency); it also appends a series of AI-specific fields. These fields are carried by the AI context in `bfe_basic.Request` and are ultimately assembled and output by `mod_access_pb3`.

### Field Number Planning

AI observability fields uniformly occupy field numbers 701-900 of `bfe-access-pb`, divided into multiple sub-ranges by purpose:

| Number Range | Purpose |
|----------|------|
| 701 - 713 | Fields in use, such as API Key identifier, model name, token counts, rate limit hits |
| 714 - 760 | Model and request basic information, such as provider, protocol, stream, retry, cache |
| 761 - 800 | Token and cost metering, including regular token/cost and cache/audio/image/video sub-items |
| 801 - 840 | Routing, transformation, and plugins, such as route rule hits, cluster/key attempt lists |
| 841 - 880 | Security, compliance, and privacy, such as hit/rejected Quota Plan IDs |
| 881 - 900 | Vendor extensions and reserved |

### Core AI Access Log Fields

The following table lists the most commonly used fields in daily troubleshooting and billing reconciliation:

| Field Name | Number | Type | Description | Collection Module |
|--------|------|------|------|----------|
| `ai_apikey_id` | 701 | string | Internal identifier of the API Key (`key_id`); the raw key value is not recorded | `mod_ai_token_auth` |
| `ai_apikeytags` | 702 | repeated | Entity hierarchy tags associated with the API Key | `mod_ai_token_auth` |
| `ai_requested_model` | 703 | string | Original model name requested by the client | `bfe_server/http_conn.go` |
| `ai_target_model` | 704 | string | Target model name after gateway routing/mapping | `bfe_server/reverseproxy.go` |
| `ai_stream` | 705 | bool | Whether it is a streaming response | `bfe_basic.Request.IsSse` |
| `ai_input_tokens` | 706 | int64 | Number of input tokens | `mod_ai_token_auth` / `mod_body_process` |
| `ai_output_tokens` | 707 | int64 | Number of output tokens | `mod_ai_token_auth` / `mod_body_process` |
| `ai_total_tokens` | 708 | int64 | Total token consumption | `mod_ai_token_auth` |
| `ai_ttft_us` | 709 | int64 | Time to first token (microseconds), streaming only | `mod_body_process` |
| `ai_tpot_us` | 710 | int64 | Average output token latency (microseconds), streaming only | `mod_body_process` |
| `ai_rate_limit_hits` | 711 | repeated | List of triggered rate limit policies | `mod_ai_rate_limit` |
| `ai_auth_reject_reason` | 712 | string | Authentication rejection reason | `mod_ai_token_auth` |
| `ai_auth_reject_quota_plans` | 713 | repeated | List of Quota Plan IDs with insufficient balance at rejection | `mod_ai_token_auth` |
| `ai_provider` | 714 | string | Upstream model provider identifier | `bfe_server/reverseproxy.go` |
| `ai_retry_count` | 715 | uint32 | Number of key-level retries at the model invocation layer | `bfe_server/reverseproxy.go` |
| `ai_mode` | 716 | string | AI request mode, such as `chat`, `image_generation` | `bfe_server/http_conn.go` |
| `ai_protocol` | 717 | string | AI protocol/auth style, such as `openai`, `anthropic` | `bfe_basic.GetApiKey` / `bfe_server/reverseproxy.go` |
| `ai_cost_value` | 761 | int64 | Estimated cost (fixed-point integer, RMB precision is 1e-8 yuan) | `mod_ai_token_auth` |
| `ai_cost_currency` | 762 | string | Cost currency, such as `RMB` / `USD` | `bfe_server/reverseproxy.go` |
| `ai_route_rule_hits` | 801 | repeated | List of hit AI routing rules | `mod_ai_route` |
| `ai_cluster_key_names` | 802 | repeated | List of (cluster, key) pairs tried during request processing | `bfe_server/reverseproxy.go` |
| `ai_auth_hit_quota_plans` | 841 | repeated | List of Quota Plan IDs hit by normal requests | `mod_ai_token_auth` |

It is worth emphasizing that `ai_apikey_id` in the access log only records the internal identifier of the API Key and never the raw key value, thereby avoiding leakage of sensitive information. The raw key is still kept in memory for injection into upstream requests, but it is never written to the logs.

In addition to the core fields listed above, the 781-790 sub-range of the 761-800 range is used for metering sub-items, covering cache, audio, image, and video generation scenarios:

| Field Name | Number | Type | Description |
|--------|------|------|------|
| `ai_cache_read_tokens` | 781 | int64 | Tokens read from cache (already included in `ai_input_tokens`) |
| `ai_cache_write_tokens` | 782 | int64 | Tokens written to cache (independent add-on item) |
| `ai_audio_input_tokens` | 783 | int64 | Audio input tokens (already included in `ai_input_tokens`) |
| `ai_audio_output_tokens` | 784 | int64 | Audio output tokens (already included in `ai_output_tokens`) |
| `ai_image_count` | 785 | int64 | Number of images generated (image_generation mode) |
| `ai_image_input_tokens` | 786 | int64 | Image input tokens (already included in `ai_input_tokens`) |
| `ai_video_count` | 787 | int64 | Number of videos generated (video_generation mode) |

`ai_image_input_tokens` and `ai_video_count` depend on `bfe-access-pb` v0.3.5. Their collection logic is located in `reqAiInfoGen()` of `bfe_modules/mod_access_pb3/request_log.go`; the values come from `ImageInputTokens` and `VideoCount` of `bfe_basic.TokenUsage`, which are filled by `mod_ai_token_auth` when parsing the usage in the response phase.

### Control Plane Operation Logs (Audit Data Source)

The access log captures the request lifecycle of the Data Plane, while every configuration change in the Control Plane (AI Gateway API) is recorded by the Operation Log module, forming an audit observability data source. The Control Plane records every configuration change via the `operation_logs` module and exposes the query interface `GET /open-api/v1/operation-logs` (see [Appendix 1: OpenAPI Quick Reference](../appendix/appendix01-openapi-quick-reference.md) for the interface definition).

Write operations on each resource (covering create/update/delete on modules such as `/entities`, `/api-keys`, `/providers`, `/clusters`, `/routes`, `/global-route-rules`, `/certificates`, `/quota-plans`, `/model-prices`, and `/auth`) automatically produce an operation log entry upon success or failure; no write interface is exposed at the API layer. Each log entry records:

- Operator information: `operator_type` (user/token), `operator_id`, `operator_name`;
- Resource information: `action` (create/update/delete/reset/import/bind/unbind), `resource_type`, `resource_id`, `resource_name`, `resource_parent_id`;
- Execution result: `status` (1 success / 2 failed), with `error_msg` attached on failure;
- Change summary: `change_summary`, including the content before and after the change (`before`/`after`) and the list of changed fields (`diff_keys`); sensitive fields such as API-Key tokens, passwords, and certificate private keys are masked;
- Request context: `request_path`, `request_method`, `client_ip`, `user_agent`, and `created_at`.

Operation logs are stored in the `operation_logs` table of the Control Plane database. The `log_id` is consistent with the LogID in the BFE access log, so audit scenarios can query by operator, resource, time range, and other criteria with pagination, enabling two-way tracing between "configuration changes" and "request behavior".

## Key Monitoring Metrics

Each BFE AI module collects and exposes key metrics at runtime. The following tables summarize the main monitoring items of `mod_ai_route` and `mod_ai_rate_limit`.

### Routing Module Monitoring Items

`mod_ai_route` routes requests to different backend clusters and models based on AI routing rules. Its monitoring items are as follows:

| Monitoring Item | Description |
|--------|------|
| `REQ_TOTAL` | Total number of requests |
| `REQ_HIT_APIKEY` | Number of requests hitting apikey routes |
| `REQ_HIT_ENTITY` | Number of requests hitting entity routes |
| `REQ_HIT_GLOBAL` | Number of requests hitting global routes |
| `REQ_MISS` | Number of requests that hit no route |
| `REQ_FALLBACK` | Number of requests hitting fallback |

By examining the ratio of `REQ_HIT_*` to `REQ_MISS`, operators can quickly determine whether the routing rule configuration is reasonable; `REQ_FALLBACK` reflects the trigger frequency of the fallback route, and an excessively high value indicates that upstream clusters or rule priorities need adjustment.

### Rate Limit and Quota Monitoring Items

`mod_ai_rate_limit` supports Redis-based distributed rate limiting, with TPM, RPM, and max concurrency limits configurable by dimensions such as product and apikey. Although the module documentation does not list monitoring item names one by one, combining the `ai_rate_limit_hits` log field with the BFE module framework, the following metrics deserve attention:

| Metric Category | Description |
|----------|------|
| Rate limit trigger count | Total number of rate limit triggers per policy and per dimension (RPM/TPM/concurrency) |
| Rate limit rejection ratio | Ratio of requests returning 429 to total requests |
| Quota hit count | Number of Quota Plans hit by normal requests, corresponding to `ai_auth_hit_quota_plans` |
| Quota rejection count | Number of requests rejected due to insufficient balance or expiration, corresponding to `ai_auth_reject_quota_plans` |
| Token consumption rate | Rate of `ai_total_tokens` aggregated by model, provider, and apikey |
| TTFT / TPOT percentiles | Time to first token and average output token latency of streaming responses |

It is recommended to aggregate the above metrics by dimensions such as `product`, `apikey_id`, `model`, and `provider`, so that problematic tenants or models can be quickly located in multi-tenant scenarios.

## Prometheus / Zabbix Integration

### BFE Monitoring Interface

The BFE Data Plane has a built-in monitoring interface that exposes module monitoring data through configuration by default. Prometheus can collect these metrics via HTTP pull; Zabbix can collect data via HTTP agent type monitoring items or custom scripts.

### Prometheus Integration Example

Add a scrape configuration for BFE in `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'bfe-ai-gateway'
    static_configs:
      - targets: ['bfe-node-1:8421', 'bfe-node-2:8421']
    metrics_path: /monitor/metrics
    scrape_interval: 15s
```

After collection, the following alert rules can be defined in Prometheus:

```yaml
groups:
  - name: ai_gateway_alerts
    rules:
      - alert: AIRouteMissRateHigh
        expr: rate(bfe_mod_ai_route_req_miss[5m]) / rate(bfe_mod_ai_route_req_total[5m]) > 0.05
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "AI route miss rate too high"

      - alert: AIRateLimitTriggered
        expr: rate(bfe_mod_ai_rate_limit_rejected[5m]) > 0
        for: 1m
        labels:
          severity: info
        annotations:
          summary: "AI rate limit policy triggered"
```

### Zabbix Integration Example

In Zabbix, you can create a monitoring item of type `HTTP agent`:

| Configuration Item | Example Value |
|--------|--------|
| Name | BFE AI REQ_TOTAL |
| Type | HTTP agent |
| Key | bfe.req_total |
| URL | `http://{HOST.IP}:8421/monitor/metrics` |
| Update interval | 30s |
| Preprocessing | Regex match `bfe_mod_ai_route_REQ_TOTAL\s+(\d+)` |

For complex metric parsing, you can also use Zabbix UserParameter to call a local script that converts the BFE monitoring interface output into a format acceptable to Zabbix Sender.

## Reporting System: Lightweight and Standard Forms

Logs and metrics solve "data collection"; reporting solves "data consumption". In the traditional pipeline, access logs only become readable dashboards after passing through three external components: Kafka, Doris, and Grafana — too heavy for small-scale or private deployments. Therefore, the Rainway AI Gateway provides two switchable deployment forms for reporting, sharing the same query API and console pages:

| Item | Lightweight Form | Standard Form |
|----|----------|----------|
| Data pipeline | BFE access logs → log-reader `mod_log_mysql` plugin → MySQL | BFE access logs → log-reader `mod_kafka` → Kafka → Doris Routine Load → Doris |
| Detail table | MySQL `bfe_ai_request_log` (89 columns, **same name and columns** as Doris) | Doris `bfe_ai_request_log` |
| Pre-aggregation | Built-in minute-level aggregation JOB in AI Gateway API | Doris-side INSERT JOB |
| Presentation | ai-gateway-web report pages (calling `/report/*`) | Grafana Dashboard (`/report/*` can also query Doris directly) |
| External dependencies | MySQL only | Kafka + Doris (+ Grafana) |
| Applicable scenarios | Small-scale / private deployments, recommended for up to ~1M log entries per day | Large-scale deployments where MySQL capacity is exceeded |

Key design trade-offs of the two pipelines:

- **The plugin writes directly to the database, without routing through the Control Plane**. log-reader is deployed with BFE in the Data Plane, and the availability of the log pipeline must not depend on the release and load of the Control Plane (AI Gateway API). Access logs are the highest-throughput data stream of the Data Plane; relaying them through the API would create a convergence bottleneck. This is consistent with the existing convention of `mod_kafka` writing directly to Kafka.
- **Same-name, same-column table design**. The MySQL detail table has the same name and columns as the Doris one; `ARRAY<STRUCT>` columns are carried as JSON columns on the MySQL side (for detail display only, never used as filter conditions). The query API returns identically structured responses for both backends, differing only in the presence of optional fields.
- **Idempotent writes**. The unique key `(hostid, log_time, ai_apikey_id, ai_requested_model)` matches the Doris UNIQUE KEY. Writes use `INSERT ... ON DUPLICATE KEY UPDATE` overwrite semantics, so log-reader redeliveries or `-b` re-reads produce no duplicates.
- **One form per cluster**. If both forms are enabled for the same cluster, the two reports become inconsistent due to their respective data-gap windows; deployments should choose exactly one.
- **Schema-first**. The DDL of the two report tables (detail table + minute aggregation table) is released with the ai-gateway-api repository (`db_ddl_report_mysql.sql`). The deployment pipeline creates the tables before enabling the log-reader plugin; the plugin itself provides no auto table creation.
- **Least-privilege accounts**. The log-reader write account holds only INSERT/UPDATE on the target table (overwrite needs UPDATE), with no DDL/DELETE. The AI Gateway API query/aggregation account holds SELECT on detail/aggregation tables, INSERT/DELETE on the aggregation table, DELETE on the detail table (non-partitioned fallback cleanup), and ALTER TABLE (partition management).

## Report Query API Design

The report query API is provided since AI Gateway API v0.0.10, mounted uniformly under `/open-api/v1/report/*`. Its metric definitions are aligned with the existing Grafana Dashboard panels (QPS, error rate, token throughput, average/percentile latency, cost, etc.). The same API and console pages work regardless of whether MySQL or Doris is the backend.

### Endpoint Overview

| Endpoint | Function | Description |
|------|------|------|
| `GET /report/overview` | Overview metric cards | Total requests, error rate, token totals, average/percentile latency, TTFT/TPOT, cost (by currency), rate limit hits, auth rejections |
| `GET /report/timeseries` | Time series | Bucketed series per `metric` (`qps`/`tokens`/`latency`/`ttft`/`tpot`/`cost`); buckets computed server-side by window (≤6h→1min, ≤3d→5min, ≤7d→30min) |
| `GET /report/rankings` | TopN rankings | Rankings by dimension (model/requested model/provider/API Key/host/status code/protocol/mode), default 10, up to 50 |
| `GET /report/distribution` | Ratio distribution | Pie chart data for status code, protocol, mode, and stream flag; empty values normalized to `unknown` |
| `GET /report/logs` | Log detail pagination | Detail-row projection (field names identical to table column names), supporting `err_only` and `keyword` fuzzy match, ordered by `log_time` descending |

Common conventions: `start`/`end` are Unix seconds with a maximum 7-day window; filters such as `models`, `apikey_ids`, `providers`, `hosts`, `stream`, and `status_codes` can be combined; all endpoints require `FeatureReport + ActionReadAll` (System scope grants it to administrators; Product scope reserves `Read` for tenant self-service usage reports).

### Query-Layer Abstraction and Dual Backends

The query layer defines a `ReportStorager` interface (`Overview`/`TimeSeries`/`Rankings`/`Distribution`/`Logs`), with one implementation per backend (`storage/mysqlreport`, `storage/dorisreport`), switched by `[Report].Backend`. `ReportManager` (`model/ireport`) only performs parameter binding validation (go-playground/validator convention), window and enum validation, bucket calculation, and metric definition constants (error rate = `error_count/request_count`; TTFT/TPOT aggregated over streaming requests only); it contains no SQL. SQL dialect differences (time-bucket functions, percentile functions) are encapsulated in the respective implementations:

- **Percentile degradation**: MySQL has no native percentile functions; `latency_p50/p90/p99` fields are always absent, and the frontend degrades accordingly. The Doris backend returns full percentiles.
- **Timezone-neutral rendering**: DATETIME → Unix seconds uniformly uses `TIMESTAMPDIFF(SECOND, '1970-01-01 00:00:00', col)` arithmetic (no timezone interpretation), avoiding offsets caused by session timezone — log-reader stores UTC wall-clock, and both backends use the same convention.

### Background JOBs of the MySQL Form (Lightweight Form Only)

MySQL lacks Doris's scheduled INSERT aggregation, so pre-aggregation is performed by JOBs embedded in AI Gateway API (the Doris form does not start these JOBs):

- **Minute aggregation JOB**: on each `AggregateIntervalSec` cycle (default 60s), it writes the previous full minute's detail rows into the aggregation table `bfe_ai_metrics_1m` (37 dimensions + 24 metrics, same dimension set as the Doris aggregation table) via `GROUP BY` over 37 dimensions. The write uses a "DELETE time window + INSERT SELECT" single transaction, making window replays idempotent. Multi-replica deployments use MySQL `GET_LOCK('report_agg_job', 0)` to elect a single runner. Process restarts do not backfill historical windows (accepting a ≤1-minute gap, symmetric with the Doris INSERT JOB semantics).
- **Partition management JOB**: inspects every 6 hours, pre-creates partitions 3 days ahead, and DROPs partitions older than `RetentionDays` (default 7 days); it automatically falls back to batched `DELETE ... LIMIT` when the target table is non-partitioned. Partitions must exist before data arrives (MySQL has no dynamic partitioning; writes to a missing partition fail outright), so the JOB pre-creates them immediately at startup.
- **DDL ownership**: the DDL of both tables is released from the ai-gateway-api repository, because it has the widest schema dependency surface (the aggregation JOB reads by column, partition management executes ALTER, and detail queries) and is the only repository in the codebase with a MySQL DDL management tradition.

### Module Assembly and Release Order

If the `[Report]` configuration is absent, the report module is not assembled at all and the `/report/*` routes are not registered (404) — a purely incremental capability. The release order is: release the API version containing the DDL and the report module → the deployment pipeline executes the DDL (schema-first, with the initial partition covering 3 days after table creation) → enable the log-reader `mod_log_mysql` plugin → configure `[Report].Backend = "mysql"` (or `"doris"` to connect an existing Doris) → release the ai-gateway-web report pages in sync.

## Error Code System

In the AI Gateway scenario, the BFE Data Plane returns unified OpenAI-compatible format error responses. The error code definitions are located in `bfe_basic/request_ai_basic.go` and are mainly produced by `mod_ai_token_auth`, `mod_ai_rate_limit`, and `bfe_server/reverseproxy.go`.

### Error Response Body Structure

```json
{
  "error": {
    "code": "QUOTA_EXHAUSTED",
    "type": "quota_error",
    "message": "Quota plan qplan-0001 exhausted.",
    "param": null,
    "details": {
      "api_key": "ak-2v8x9k3m7p",
      "key_id": "key-001",
      "quota_plan_id": "qplan-0001",
      "limit_type": "api_key_quota",
      "model": "gpt-4",
      "retry_after_seconds": 0
    }
  }
}
```

### Error Code Classification

| Layer | Main Error Codes | HTTP Status Code | Trigger Scenarios |
|------|------------|-------------|----------|
| Authentication and admission | `INVALID_REQUEST`, `NO_API_KEY`, `INVALID_API_KEY`, `KEY_DISABLED`, `KEY_EXPIRED`, `SUBNET_NOT_ALLOWED`, `MODEL_NOT_ALLOWED` | 400 / 401 / 403 | Product line does not exist, missing API Key, key invalid/disabled/expired, IP not in allowlist, model not in allowlist |
| Rate limit check | `RPM_LIMIT_EXCEEDED`, `TPM_LIMIT_EXCEEDED`, `CONCURRENCY_LIMIT_EXCEEDED`, `RATE_LIMIT_REDIS_ERROR` | 429 / 500 | RPM/TPM/concurrency rate limit triggered, Redis rate limit access failure |
| Quota deduction | `QUOTA_EXHAUSTED`, `QUOTA_EXPIRED`, `INTERNAL_QUOTA_ERROR` | 429 / 500 | Quota Plan insufficient balance, expired, Redis quota query anomaly |
| Forwarding and protocol adaptation | `PROVIDER_PROTOCOL_MISMATCH` | 400 | The requested AuthStyle is not within the `AIConf.ModelProtocols` supported range of the target cluster |

There is a direct correspondence between error codes and access log fields: `ai_auth_reject_reason` records the error code at authentication/quota rejection, `ai_auth_reject_quota_plans` records the Quota Plan IDs with insufficient balance, and `ai_rate_limit_hits` records the triggered rate limit policies.

## Alerting Recommendations

Based on logs and metrics, the following alert rules are recommended:

| Alert Name | Trigger Condition | Severity | Suggested Action |
|----------|----------|------|----------|
| Error rate surge | 5xx/4xx ratio exceeds threshold within 5 minutes | critical | Check backend model service health, Redis connectivity, quota configuration |
| Frequent rate limit triggers | RPM/TPM/concurrency rate limit count stays above 0 | warning | Evaluate whether rate limit thresholds are reasonable; raise quotas or scale out if necessary |
| Insufficient quota balance | `QUOTA_EXHAUSTED` errors appear | warning | Notify users to top up or adjust the Quota Plan |
| High route miss rate | `REQ_MISS` ratio exceeds 5% | warning | Check whether routing rules cover newly added models or API Keys |
| High time to first token | `ai_ttft_us` P99 exceeds business SLA | warning | Investigate network latency, backend model load, cold start hits |
| High average output token latency | `ai_tpot_us` P99 exceeds threshold | warning | Check model instance load, streaming response bandwidth |
| High retry count | `ai_retry_count` average rises abnormally | info | Check upstream key availability, model service stability |

Alert notifications should be split by product line or tenant to avoid global alerts drowning out critical information. Meanwhile, for billing-related alerts (such as insufficient quota), the business party should be notified first rather than only the operations team.

## Log and Metric Configuration Examples

### BFE Access Log Configuration

Enable the `mod_access_pb3` module in the BFE configuration and specify the access log output path and format:

```toml
[Server]
# ...

[Modules]
mod_access_pb3 = true
mod_ai_token_auth = true
mod_ai_route = true
mod_ai_rate_limit = true
mod_body_process = true

[Log]
AccessLogPrefix = "./log/access"
```

The access log is output in the `bfe-access-pb` protocol format, and downstream systems can connect it to a log platform for parsing, archiving, and alerting.

### AI Gateway API Log Configuration

The log configuration of the AI Gateway API Control Plane is located in `ai_gateway_api.toml`, as shown below:

```toml
[Server]
ServerPort          = 8183
GracefulTimeoutInMs = 5000
MonitorPort         = 8284

[Loggers.access]
LogName     = "access"
LogLevel    = "INFO"
RotateWhen  = "MIDNIGHT"
BackupCount = 7
Format      = "[%D %T] [%L] [%S] %M"
StdOut      = false
```

The Control Plane's `MonitorPort` is used to expose its own health checks and runtime metrics. Operators can integrate Prometheus or custom health probes through this port.

### Log Platform Parsing Tips

Since BFE access logs are encoded with Protocol Buffers, the log platform needs to load the corresponding `.proto` files for decoding. After decoding, real-time dashboards and alerts can be built based on fields such as `ai_apikey_id`, `ai_route_rule_hits`, and `ai_rate_limit_hits`.

## Chapter Summary

This chapter introduced the observability design of the Rainway AI Gateway. The key points are as follows:

- Observability consists of three pillars: logs, metrics, and traces. The BFE Data Plane currently has deep customization in logs and metrics;
- The AI access log contains 29 AI-specific fields covering the full lifecycle information of authentication, routing, rate limiting, token metering (including image/video generation sub-items), and cost estimation, and it does not record the raw API Key;
- Control Plane operation logs record the operator, resource, change summary (with diff_keys), and request context of every configuration change, with sensitive fields masked, and can be queried for audit via `GET /open-api/v1/operation-logs`;
- Key monitoring metrics include `REQ_TOTAL`, route hit/miss/fallback, rate limit triggers, quota hits and rejections, token consumption rate, TTFT/TPOT, etc.;
- Prometheus can collect BFE metrics via pull, and Zabbix can integrate via HTTP agent or custom scripts;
- The reporting system offers a lightweight form (log-reader writes directly to MySQL, no Kafka/Doris/Grafana required) and a standard form (Kafka → Doris → Grafana); only one form should be enabled per cluster. Both forms share the same detail table schema and the same set of `/report/*` query APIs and console report pages;
- The report query API shields MySQL/Doris dialect differences behind the `ReportStorager` interface; the MySQL form relies on built-in minute-aggregation and partition-management JOBs, and the module stays unassembled (endpoints 404) when `[Report]` is absent;
- The error code system is divided into four layers: authentication and admission, rate limit check, quota deduction, and forwarding and protocol adaptation, with a clear correspondence to access log fields;
- Alerts should cover error rate, rate limiting, quota, route hit rate, latency, and retries, and notifications should be split by product line or tenant.

Observability is not a one-time effort; it evolves continuously as business scale and model variety grow. It is recommended to plan the log retention period, metric aggregation dimensions, and alert severity strategy early in the launch phase, laying a solid foundation for subsequent capacity planning, cost optimization, and fault localization.

## References

- `bfe/docs/zh_cn/sys_design/ai_access_log_fields.md` — BFE AI Access Log Observability Fields Design
- `bfe/docs/zh_cn/sys_design/ai_error_codes.md` — BFE AI Gateway Error Codes
- `bfe/docs/zh_cn/modules/mod_ai_route/mod_ai_route.md` — AI Routing Module Documentation
- `bfe/docs/zh_cn/modules/mod_ai_rate_limit/mod_ai_rate_limit.md` — AI Rate Limit Module Documentation
- `ai-gateway-api/docs/zh_cn/config_param.md` — AI Gateway API Configuration File Documentation
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/operation-logs.md` — Operation Log Interface Definition
- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/report.md` — Report Query Interface Definition
- `ai-gateway-api/design-docs/modifications/2026-09-15-report-query-api/change-summary.md` — Report Query API Change Summary
- `ai-gateway-api/db_ddl_report_mysql.sql` — Report Detail and Aggregation Table DDL (MySQL Form)
- `log-reader/doc/modules/mod_log_mysql/mod_log_mysql.md` — mod_log_mysql Plugin Documentation
- `bfe_basic/request_ai_basic.go` — AI Context and Error Code Definitions in Go
- `bfe_modules/mod_access_pb3/` — Access Log Output Module
