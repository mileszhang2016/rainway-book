# 第二十四章 配置热加载与升级

## 本章目标

通过本章，读者将掌握：

- 一条配置从 AI Gateway API 管理面下发到 BFE 数据面生效的完整链路；
- Conf Agent（配置代理）如何轮询 InnerAPI、比对版本并持久化配置；
- BFE 热加载的触发方式与监控端口机制；
- 如何查看当前生效的配置版本；
- 配置异常时的版本回滚操作；
- AI Gateway API、Dashboard、Conf Agent 等组件升级时的注意事项；
- 灰度发布与分批升级的建议做法。

---

## 配置生效的完整流程

壬远AI网关采用**控制面生成、数据面拉取**的架构。管理员在 Dashboard 或通过 OpenAPI 修改配置后，配置不会立即推送到 BFE，而是先由 AI Gateway API 持久化到数据库，再由部署在 BFE 侧的 Conf Agent 定期拉取并触发热加载。完整流程如下：

```mermaid
flowchart LR
    A[管理员修改配置] --> B[AI Gateway API 写入数据库]
    B --> C[VersionControlManager 计算 MD5 签名]
    C --> D[config_versions 生成新版本]
    D --> E[Conf Agent prober 轮询 InnerAPI]
    E --> F{版本是否变化}
    F -- 否 --> G[跳过本次更新]
    F -- 是 --> H[file_store 写入新版本目录]
    H --> I[切换 symlink mod_xxx]
    I --> J[trigger 调用 BFE /reload/{module}]
    J --> K[BFE 加载新配置]
```

各环节说明：

1. **配置写入**：管理员通过 Dashboard 或 OpenAPI 修改 Provider、Cluster、API-Key、限流策略等资源配置，AI Gateway API 将变更写入 MySQL/SQLite。
2. **配置导出与版本控制**：当 Conf Agent 请求 InnerAPI 时，AI Gateway API 调用 `VersionControlManager.ExportConfig` 生成配置，并将版本号字段置零后计算 MD5 签名。若签名与 `config_versions` 表中最新记录相同，则返回 `Data: null`；否则生成新的时间戳版本号并返回全量配置。详见 `ai-gateway-api/design-docs/sys-design/details/InnerAPI配置导出与版本控制.md`。
3. **Conf Agent 拉取**：Conf Agent 中的 `prober` 按配置周期访问 InnerAPI，比较本地版本号与远程版本号。
4. **本地持久化**：当检测到版本变化时，`file_store` 将新配置写入以版本号命名的目录，并原子切换 `mod_{name}` 软链接指向新目录。
5. **BFE 热加载**：`trigger` 调用 BFE 监控端口的 `/reload/{module}` 接口，BFE 重新读取软链接指向的配置文件，完成热加载。

这种拉取式设计使控制面无需感知数据面节点数量，BFE 在控制面短暂不可达时仍可继续使用本地缓存配置转发流量。

---

## Conf Agent 轮询与版本比对

Conf Agent 是配置下发链路的关键组件，其架构与职责见 `conf-agent/AGENTS.md`。每个配置主题对应一个 `Reloader`，每个 `Reloader` 包含三个子模块：

- **prober**（`conf_reload/prober/`）：负责轮询 AI Gateway API 的 InnerAPI；
- **file_store**（`conf_reload/file_store/`）：负责将配置写入版本目录并管理软链接；
- **trigger**（`conf_reload/trigger/`）：负责调用 BFE 监控端口触发重载。

### 轮询配置

Conf Agent 的配置文件 `conf/conf-agent.toml` 中定义了轮询间隔、BFE 配置目录、监控端口以及各 `Reloader` 的映射关系。关键配置项如下：

```toml
[Basic]
BFECluster              = "bfe-cluster-01"
BFEConfDir              = "/home/work/bfe/conf"
BFEMonitorPort          = 8421
BFEReloadTimeoutMs      = 1500
ReloadIntervalMs        = 5000
ConfServer              = "http://127.0.0.1:8183"
ConfTaskHeaders         = {"Authorization" = "Token {Token}"}
ConfTaskTimeoutMs       = 1500
```

`ReloadIntervalMs` 决定 prober 的轮询周期，默认 5000 毫秒。每个 `Reloader` 会按此周期向对应的 InnerAPI 发起 GET 请求，并携带本地缓存的版本号：

```http
GET /inner-api/v1/configs/mod-api-key?version=20260101120000 HTTP/1.1
Authorization: Token <token>
```

### 版本比对

AI Gateway API 收到请求后，会在 `config_versions` 表中查找该主题的最新签名与版本号。若请求的 `version` 与最新版本一致，则返回：

```json
{
  "ErrNum": 200,
  "ErrMsg": "success",
  "Data": null
}
```

Conf Agent 收到空响应后跳过写入与热加载。若版本已变化，则返回包含新 `version` 的全量配置数据，Conf Agent 进入持久化与触发流程。

### 本地版本目录与软链接

`file_store` 为新版本配置分配独立目录（目录名通常包含版本号），写入完成后再原子地修改 `mod_{name}` 软链接指向该目录。例如 `mod_ai_token_auth` 主题对应的软链接为 `mod_ai_token_auth`，BFE 始终通过该软链接读取 `token_rule.data` 等文件。这种机制避免了 BFE 读取到半截文件，也保留了若干历史版本目录，便于回滚。

---

## BFE 热加载触发机制

BFE 在启动时会监听一个监控端口（默认 8421），用于接收管理类请求，其中就包括配置热加载接口 `/reload/{module}`。Conf Agent 的 `trigger` 模块在 `file_store` 完成软链接切换后，会构造如下请求：

```bash
curl -X POST http://127.0.0.1:8421/reload/mod_ai_token_auth
```

不同配置主题对应不同的 BFE 重载接口，典型映射如下：

| 配置主题 | Conf Agent Reloader | BFE 热加载接口 |
|----------|---------------------|----------------|
| `route_rule` | `server_data_conf` | `/reload/server_data_conf` |
| `cluster_table` / `gslb.<cluster>` | `cluster_conf` | `/reload/gslb_data_conf` |
| `certificate` | `tls_conf` | `/reload/tls_conf` |
| `mod_api_key_rule` | `mod_ai_token_auth` | `/reload/mod_ai_token_auth` |
| `mod_body_process` | `mod_body_process` | `/reload/mod_body_process` |
| `mod_ai_rate_limit` | `mod_ai_rate_limit` | `/reload/mod_ai_rate_limit` |
| `ai_route` | `mod_ai_route` | `/reload/mod_ai_route` |

这些映射在 `conf-agent/conf/conf-agent.toml` 的 `[Reloaders.xxx]` 段中通过 `BFEReloadAPI` 字段显式配置：

```toml
[Reloaders.mod_ai_token_auth]
BFEReloadAPI    = "/reload/mod_ai_token_auth"
ReloadFile      = "token_rule.data"
CopyFiles       = ["token_rule.data", "mod_ai_token_auth.conf"]
[[Reloaders.mod_ai_token_auth.NormalFileTasks]]
ConfAPI         = "/inner-api/v1/configs/mod-api-key"
ConfFileName    = "token_rule.data"
```

BFE 收到重载请求后，会重新读取对应模块的配置文件并更新内存中的规则表。由于重载过程不涉及进程重启，因此不会中断现有连接。

---

## 如何查看当前生效配置版本

运维中经常需要确认 BFE 当前加载的是哪一版配置。可通过以下三种方式查看：

### 1. 查看 Conf Agent 日志

Conf Agent 在每次成功写入新版本并触发热加载后都会打印日志，路径由 `conf-agent.toml` 中的 `[Logger]` 段指定。日志中通常包含版本号、模块名与重载结果。

### 2. 查看软链接指向

直接查看 BFE 配置目录中各模块软链接指向的版本目录：

```bash
ls -l /home/work/bfe/conf/mod_ai_token_auth
# 输出示例：
# mod_ai_token_auth -> mod_ai_token_auth_20260102120000
```

软链接目标目录名中的时间戳即为当前生效版本。

### 3. 查看配置文件中的 version 字段

多数导出配置文件的 JSON 根节点包含 `version` 字段，可直接读取：

```bash
cat /home/work/bfe/conf/mod_ai_token_auth_20260102120000/token_rule.data | head -c 200
```

输出中应包含 `"version": "20260102120000"` 字样。

---

## 版本回滚操作

当新配置导致 BFE 行为异常时，可快速回滚到上一版本。由于 Conf Agent 保留了若干历史版本目录（数量由 `VersionKeepCount` 控制，默认通常保留最近几个版本），回滚无需重新从控制面拉取。

### 手动回滚步骤

1. 确认要回滚到的历史版本目录，例如 `mod_ai_token_auth_20260101120000`。
2. 停止 Conf Agent 或临时调大轮询间隔，避免它立即将软链接再次切回新版本。
3. 手动修改软链接指向目标版本目录：

```bash
cd /home/work/bfe/conf
ln -sfn mod_ai_token_auth_20260101120000 mod_ai_token_auth
```

4. 调用 BFE 监控端口重新加载该模块：

```bash
curl -X POST http://127.0.0.1:8421/reload/mod_ai_token_auth
```

5. 验证业务恢复后，在 Dashboard 修正配置并重新发布。

### 回滚注意事项

- 回滚仅影响数据面本地配置，不会回写数据库。若控制面配置本身有误，仍需在 Dashboard 中修正。
- 对 `server_data_conf`、`cluster_conf`、`tls_conf` 等基础模块回滚时，需确认各模块版本之间的兼容性，避免路由与后端实例表版本不一致。
- 生产环境建议先在小范围节点验证再全量切换，详见下一节灰度发布建议。

---

## 升级注意事项

AI Gateway 涉及控制面（AI Gateway API）、Dashboard、Conf Agent、BFE 以及 MySQL 等多个组件。升级时需按正确顺序执行，并关注数据库迁移与配置兼容性。以下以 `ai-gateway-api/docs/zh_cn/upgrade.md` 中 v0.0.2 的升级路径为例说明。

### 数据库迁移

AI Gateway API 升级时常伴随数据库表结构变更。升级前应备份数据库，然后按升级文档执行 DDL。例如 v0.0.2 需要对 `users` 表做以下调整：

```sql
ALTER TABLE users ADD COLUMN `type` tinyint(1) NOT NULL DEFAULT '0' AFTER name;
ALTER TABLE users ADD COLUMN `scopes` varchar(2048) NOT NULL DEFAULT '' AFTER `type`;

UPDATE users SET type = 0, scopes = 'System' WHERE roles = 'admin';
UPDATE users SET type = 1, scopes = 'Support' WHERE roles = 'inner';

ALTER TABLE users CHANGE COLUMN `session_key` `ticket` varchar(20) NOT NULL DEFAULT '';
ALTER TABLE users CHANGE COLUMN `session_key_created_at` `ticket_created_at` datetime NOT NULL DEFAULT '0000-01-01 00:00:00';

ALTER TABLE users DROP COLUMN `roles`;

ALTER TABLE users DROP INDEX name_uni;
ALTER TABLE users ADD UNIQUE KEY `name_uni` (`name`, `type`);
```

升级前务必：

- 对生产库做完整备份；
- 在测试环境先验证 DDL 与数据迁移脚本；
- 确认升级文档中列出的起始版本是否包含当前版本。

### 配置兼容性

新版本可能引入新的配置字段或变更鉴权头格式。例如 v0.0.2 中，Conf Agent 请求头从 `Session {Token}` 调整为 `Token {Token}`：

```toml
# 旧版本
ConfTaskHeaders = {"Authorization" = "Session {Token}"}

# v0.0.2 及以后
ConfTaskHeaders = {"Authorization" = "Token {Token}"}
```

升级后应检查：

- `conf-agent.toml` 中的 `ConfTaskHeaders`、`ExtraFileTaskHeaders` 是否与 AI Gateway API 的鉴权方式一致；
- 新增模块的 `Reloader` 是否已正确配置；
- BFE 是否支持新模块的热加载接口。

### 组件版本配套

根据 `upgrade.md`，v0.0.2 升级需要：

- AI Gateway API 升级为 v0.0.2；
- Dashboard 升级为 v0.0.2；
- Conf Agent 保持 v0.0.1 或更新版本。

建议的升级顺序为：

1. 备份数据库；
2. 执行数据库迁移脚本；
3. 替换 AI Gateway API 可执行程序；
4. 升级 Dashboard；
5. 按需升级 Conf Agent 并检查配置；
6. 观察 Conf Agent 日志与 BFE 监控指标，确认配置正常同步。

---

## 灰度发布建议

配置变更或版本升级不应一次性全量下发，否则一旦新配置存在问题，会影响全部流量。建议采用以下灰度策略：

### 1. 按 BFE 集群灰度

`gslb.<bfe_cluster>` 主题天然按 BFE 集群名区分版本线。可先将变更应用到部分 BFE 集群，观察错误率、延迟、配额消耗等指标，确认无误后再推广到其他集群。

### 2. 按节点分批升级

对于 AI Gateway API 或 Conf Agent 的程序升级，可按节点分批进行：

- 先升级 1 台 Conf Agent 所在节点，观察日志与热加载结果；
- 确认无误后，以 10%、30%、100% 的节奏逐步扩大范围；
- 每批升级后执行关键接口探测，例如：

```bash
curl -H "Authorization: Token <token>" \
     "http://127.0.0.1:8183/inner-api/v1/configs/mod-api-key"
```

### 3. 配置变更后的观察窗口

Conf Agent 的轮询周期默认为 5 秒，意味着配置下发到 BFE 生效通常有数秒延迟。建议：

- 在 Dashboard 保存配置后等待至少 2-3 个轮询周期；
- 通过 BFE 监控端口或 Conf Agent 日志确认版本号已更新；
- 使用测试流量验证关键路由、API-Key 认证、限流规则是否按预期工作。

### 4. 回滚预案

灰度期间应预先确定回滚触发条件，例如：

- 错误率较基线上升超过阈值；
- 特定模型或 API-Key 请求失败率异常；
- 配额或限流行为与预期不符。

一旦触发，立即停止扩大灰度范围，并按 23.5 节步骤回滚异常节点。

---

## 完整操作示例

以下演示从 Dashboard 修改 API-Key 配额到 BFE 热加载生效，再到回滚的完整操作。

### 场景

管理员将 `team-a` 的日 Token 配额从 100 万调整为 200 万。

### 步骤 1：在 Dashboard 修改配额

保存后，AI Gateway API 将新配额写入数据库，但尚未下发到 BFE。

### 步骤 2：等待 Conf Agent 轮询

约 5 秒内，Conf Agent 的 `mod_ai_token_auth` Reloader 会检测到 `mod-api-key` 主题版本变化：

```bash
# Conf Agent 日志示例（路径以实际配置为准）
tail -f /home/work/conf-agent/log/conf_agent.log
# [2026-01-02 12:00:05] [INFO] [prober.go:xxx] mod_ai_token_auth version changed: 20260101120000 -> 20260102120000
# [2026-01-02 12:00:05] [INFO] [file_store.go:xxx] write mod_ai_token_auth_20260102120000
# [2026-01-02 12:00:05] [INFO] [trigger.go:xxx] reload mod_ai_token_auth success
```

### 步骤 3：查看当前生效版本

```bash
ls -l /home/work/bfe/conf/mod_ai_token_auth
# mod_ai_token_auth -> mod_ai_token_auth_20260102120000

curl -X POST http://127.0.0.1:8421/reload/mod_ai_token_auth
# {"code":0,"message":"reload success"}
```

### 步骤 4：验证配额生效

通过测试请求携带 `team-a` 的 API-Key 发起调用，确认限额已变为 200 万。

### 步骤 5：回滚（如需要）

若发现配额调整后业务异常，可快速回滚：

```bash
cd /home/work/bfe/conf
ln -sfn mod_ai_token_auth_20260101120000 mod_ai_token_auth
curl -X POST http://127.0.0.1:8421/reload/mod_ai_token_auth
ls -l mod_ai_token_auth
# mod_ai_token_auth -> mod_ai_token_auth_20260101120000
```

---

## 本章小结

- 壬远AI网关采用控制面生成、数据面拉取的配置同步模式：管理员变更先写入数据库，Conf Agent 再按周期从 InnerAPI 拉取并触发 BFE 热加载。
- `VersionControlManager` 基于 MD5 签名与 `config_versions` 表实现增量同步；配置未变化时返回 `Data: null`，避免无意义的配置下发。
- Conf Agent 的每个 `Reloader` 由 prober、file_store、trigger 三部分组成，分别负责拉取、持久化与触发 BFE 热加载。
- BFE 通过监控端口的 `/reload/{module}` 接口完成热加载，各模块对应独立的热加载路径。
- 生效版本可通过 Conf Agent 日志、软链接指向或配置文件中的 `version` 字段查看。
- 版本回滚可利用 Conf Agent 保留的历史版本目录手动切换软链接并重新加载，无需重启 BFE。
- 升级时需按顺序执行数据库迁移、AI Gateway API 替换、Dashboard 升级与 Conf Agent 配置检查，并注意鉴权头与配置字段的兼容性。
- 灰度发布建议按 BFE 集群或节点分批进行，配合观察窗口与回滚预案，降低变更风险。

---

## 参考文档

- `ai-gateway-api/docs/zh_cn/upgrade.md`
- `conf-agent/AGENTS.md`
- `ai-gateway-api/design-docs/sys-design/details/InnerAPI配置导出与版本控制.md`
- `ai-gateway-api/design-docs/api-define/InnerAPI接口定义/00-overview.md`
- [第二十一章 配置导出与版本控制设计](../design/chapter14-config-export-and-version-control.md)
