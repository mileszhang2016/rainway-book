# 第二十三章 域名与证书配置

## 本章目标

通过本章，读者将学会：

- 在壬远AI网关中为产品（Product）绑定自定义域名；
- 上传 TLS 服务端证书与私钥，并理解控制面的校验规则；
- 设置和维护默认证书（Default Certificate），保证未知域名的 HTTPS 握手不失败；
- 跟踪证书过期时间，规划续期与替换流程；
- 验证 HTTPS 访问是否生效，并排查常见的域名与证书不匹配问题；
- 在同一网关中为多个域名配置独立的证书和路由规则。

---

## 22.1 域名绑定操作

壬远AI网关的数据面 BFE 使用 `server_data_conf` 中的 **HostTable（域名表）** 将请求域名映射到具体的产品线（Product），再由 **RouteTable（路由表）** 将流量分发到目标 Cluster。域名绑定不是直接修改 BFE 配置文件，而是在 AI Gateway API 中维护 Provider / Cluster / 路由规则后，由 InnerAPI 自动导出为 BFE 可消费的配置。

### 22.1.1 域名与产品线的映射关系

`server_data_conf` 的 `HostTable` 包含三层映射：

```
hostname（例如 api.example.com）
    │
    ▼
host-tag（例如 host-tag-1）
    │
    ▼
product（例如 AI_product）
```

- **Hosts**：`host-tag` 到 `hostname` 列表的映射，一个 host-tag 可以关联多个域名；
- **HostTags**：`product` 到 `host-tag` 列表的映射，一个 product 可以关联多组 host-tag；
- **DefaultProduct**：当请求域名无法匹配任何 Hosts 时，BFE 使用该默认产品线处理请求。

在 Dashboard 中，域名绑定通常在创建或编辑 Cluster / 路由规则时同步完成。管理员填入希望对外暴露的域名（如 `api.example.com`）后，AI Gateway API 会自动生成对应的 HostTable 与 RouteTable 条目，并通过 InnerAPI `/configs/tls_conf/server_data_conf` 导出给 Conf Agent 与 BFE。

### 22.1.2 默认产品线的作用

`DefaultProduct` 决定了未匹配任何域名时的 fallback 行为。对于 AI 网关场景，建议将主 AI 产品设置为默认产品线，这样即使业务方使用临时域名或 IP 直接访问，也能进入正确的路由逻辑。若默认产品线配置错误，可能导致请求返回 404 或被路由到非预期的 Cluster。

### 22.1.3 域名变更的生效流程

域名或路由规则变更后，并不会立即作用于 BFE，而是遵循配置导出与热加载流程：

1. AI Gateway API 收到域名或路由规则变更请求，完成校验后写入 MySQL，并生成新的配置版本号；
2. InnerAPI `/configs/tls_conf/server_data_conf` 将 HostTable、RouteTable 和 ClusterConf 导出为 BFE 可识别的 JSON 结构；
3. Conf Agent 按固定周期轮询 InnerAPI，发现版本号变化后，拉取新配置并持久化到本地版本目录；
4. Conf Agent 通过符号链接（Symlink）原子切换当前生效目录，然后调用 BFE 的 `/reload` 接口热加载；
5. BFE 在不中断已有连接的情况下使用新的域名与路由配置处理后续请求。

管理员可以通过 Dashboard 或 InnerAPI 查看当前生效的配置版本号，确认变更是否已下发到目标 BFE 节点。

---

## 22.2 TLS 证书上传

HTTPS 接入依赖服务端证书（Server Certificate）和私钥（Private Key）。壬远AI网关通过 OpenAPI `/certificates` 统一管理证书，而不是要求管理员登录 BFE 节点手动上传文件。控制面负责校验、存储、版本控制，并通过 InnerAPI 将证书路径映射下发给数据面。

### 22.2.1 证书数据模型

创建证书时需要提交以下字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `cert_name` | string | 证书名，全局唯一，长度 2-64 字符，仅允许字母、数字、`_`、`-`、`.` |
| `description` | string | 证书描述，长度 2-256 字符 |
| `is_default` | bool | 是否为默认证书，全局必须有且只有一个默认证书 |
| `cert_file_content` | string | 证书文件内容，PEM 格式 X.509 证书 |
| `key_file_content` | string | 私钥文件内容，PEM 格式私钥 |
| `expired_date` | string | 过期时间，只读，由服务端从证书内容解析 |

```json
{
  "cert_name": "cert_demo",
  "description": "api.example.com 生产证书",
  "is_default": true,
  "cert_file_content": "-----BEGIN CERTIFICATE-----\nMIIDXTCCAkWgAwIBAgIJAKoK/heBjcOuMA0GCSqGSIb3DQEBCwUAMEUxCzAJBgNV\n...\n-----END CERTIFICATE-----",
  "key_file_content": "-----BEGIN RSA PRIVATE KEY-----\nMIIEpQIBAAKCAQEA3Tz2mvC3D1tRkX7B8kYwQ8u8kYwQ8u8...\n-----END RSA PRIVATE KEY-----"
}
```

### 22.2.2 创建证书接口

- **端点**：`POST /open-api/v1/certificates`
- **必填字段**：`cert_name`、`description`、`is_default`、`cert_file_content`、`key_file_content`
- **校验规则**：
  - 证书与私钥必须为合法 PEM 格式；
  - 证书与私钥必须配对，服务端会校验二者是否匹配；
  - 若当前系统中尚无默认证书，则新证书必须设置为默认；
  - `cert_name` 不得重复，且不能命名为 `BFE_DEFAULT_CERT`。

创建成功后，接口返回证书元数据，但**不会返回**证书和私钥内容：

```json
{
  "ErrNum": 200,
  "ErrMsg": "success",
  "Data": {
    "cert_name": "cert_demo",
    "description": "api.example.com 生产证书",
    "is_default": true,
    "expired_date": "2026-08-23 16:02:31"
  }
}
```

如果校验失败，接口会返回明确的错误信息。常见失败原因包括：证书与私钥不匹配、PEM 格式非法、`cert_name` 命名不符合规范或已存在、未设置默认证书但系统中尚无默认证书等。建议在自动化脚本中捕获这些错误并记录原始返回的 `ErrMsg`，以便快速定位问题。

### 22.2.3 证书列表与详情

- `GET /open-api/v1/certificates`：返回全部证书列表，不含证书和私钥内容；
- `GET /open-api/v1/certificates/{cert_name}`：返回单个证书详情，同样不包含敏感内容。

这种设计保证了证书私钥不会被频繁通过网络传输，仅在创建和更新时由管理员显式提供。

---

## 22.3 设置默认证书

### 22.3.1 默认证书的职责

当客户端通过 HTTPS 访问网关，但请求的域名（Server Name Indication，SNI）没有匹配到任何已配置的证书时，BFE 会回退到**默认证书**完成 TLS 握手。如果未设置默认证书，或默认证书与监听端口不匹配，客户端将收到 TLS 握手失败或证书警告。

因此，默认证书必须满足：

- 全局有且只有一个；
- 始终存在于 `CertConf` 映射中；
- 不能被直接删除，必须先将其切换为非默认后再删除。

### 22.3.2 设置默认证书的方式

**方式一：创建证书时直接指定**

在请求体中将 `is_default` 设置为 `true`。如果已存在默认证书，旧的默认证书会自动变为非默认。

**方式二：更新已有证书为默认**

调用专用接口：

```
PATCH /open-api/v1/certificates/{cert_name}/default
```

该接口会将当前默认证书更新为非默认，并将目标证书设为新的默认证书。返回数据与普通证书详情一致。

### 22.3.3 删除非默认证书

调用：

```
DELETE /open-api/v1/certificates/{cert_name}
```

约束条件：

- 只能删除非默认证书；
- 删除后全局仍必须保留一个默认证书（由系统保证）。

---

## 22.4 证书过期管理

### 22.4.1 过期时间来源

`expired_date` 字段是只读的，由 AI Gateway API 在创建或更新证书时从 `cert_file_content` 中解析得到，格式固定为 `YYYY-MM-DD HH:MM:SS`。管理员无需手动填写，也无法通过 OpenAPI 修改该字段。

### 22.4.2 日常监控建议

建议通过以下方式持续跟踪证书状态：

1. **定期调用证书列表接口**：遍历 `expired_date`，对 30 天、14 天、7 天内即将过期的证书设置告警；
2. **Dashboard 可视化**：在控制台证书管理页面置顶显示即将过期或已过期证书；
3. **告警规则示例**：
   - 证书剩余有效期 ≤ 30 天：提示（Info）；
   - 证书剩余有效期 ≤ 7 天：警告（Warning）；
   - 证书已过期：严重（Critical），并考虑临时切换备用证书。

对于拥有大量域名的生产环境，建议将证书列表接口接入监控告警系统或 CI/CD 流水线，实现自动化的到期提醒。部分团队会结合 ACME 协议（如 Let's Encrypt）在证书到期前自动申请新证书，并通过 OpenAPI 自动上传到 AI Gateway API，从而将人工干预降到最低。

### 22.4.3 续期与替换流程

证书续期通常包含以下步骤：

1. 向证书颁发机构（CA）申请新证书，获取新的 PEM 证书和私钥；
2. 在 AI Gateway API 中更新目标证书（当前 OpenAPI 通过删除后重建，或后续支持 PUT 更新内容）；
3. 确认 `expired_date` 已刷新为新证书过期时间；
4. 通过 Conf Agent 等待数据面热加载完成；
5. 使用 `curl -v https://域名` 验证新证书生效；
6. 保留旧证书至少一个 TTL 周期，确认无异常后再清理。

> 注意：默认证书过期前必须优先续期，否则所有未匹配域名的 HTTPS 请求都会受到影响。

如果新证书上传后发现 HTTPS 访问异常，应立即切换回旧证书（若旧证书仍在有效期内），或临时将另一张备用证书设置为默认，避免业务长时间中断。回滚后同样需要通过 `curl -v` 或浏览器验证访问是否恢复正常。

---

## 22.5 HTTPS 访问验证

### 22.5.1 验证 TLS 握手

完成域名绑定和证书上传后，可通过命令行工具验证 HTTPS 是否正常工作：

```bash
curl -v https://api.example.com/v1/chat/completions \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek-v3","messages":[{"role":"user","content":"hello"}]}'
```

在 `-v` 输出中关注以下信息：

- `* Server certificate:` 是否正确显示目标域名；
- `*  subjectAltName:` 是否包含 `api.example.com`；
- `*  issuer:` 是否为预期的 CA；
- `*  SSL certificate verify ok.` 表示证书链校验通过。

除 `curl` 外，也可使用 `openssl s_client` 专注于 TLS 层验证：

```bash
echo | openssl s_client -connect api.example.com:443 -servername api.example.com
```

该命令会输出完整的证书链、协议版本、加密套件等信息，适合在怀疑证书链不完整或 SNI 选择异常时使用。

### 22.5.2 常见排查方向

| 现象 | 可能原因 | 排查方法 |
|------|----------|----------|
| 浏览器提示证书不安全 | 证书与访问域名不匹配 | 检查证书 SAN 是否包含目标域名；检查 HostTable 是否绑定正确域名 |
| TLS 握手失败 | 未配置默认证书 | 调用 `/certificates` 确认 `is_default` 为 true 的证书存在 |
| 请求返回 404 | 域名未绑定到正确 product | 检查 `/configs/tls_conf/server_data_conf` 的 HostTable 与 RouteTable |
| 证书链不完整 | 缺少中间证书 | 确保证书内容包含完整的证书链（服务器证书 + 中间证书） |

---

## 22.6 多域名配置

在生产环境中，AI 网关往往需要同时对外暴露多个域名，例如：

- `api.example.com`：主业务域名；
- `api-cn.example.com`：中国大陆区域域名；
- `internal.example.com`：内部测试域名。

### 22.6.1 多域名与多证书的对应关系

壬远AI网关支持以下两种模式：

**单证书覆盖多域名**

申请一张包含多个 SAN（Subject Alternative Name）的通配符或多域名证书，例如同时覆盖 `*.example.com` 和 `api.example.com`。此时只需上传一张证书，并在 HostTable 中将多个域名指向同一 product。

**多证书一一对应**

为每个域名或子域名上传独立证书，例如 `cert_api_example_com`、`cert_api_cn_example_com`。BFE 会根据客户端 SNI 自动选择匹配的证书；无匹配时回退到默认证书。

### 22.6.2 配置注意事项

- 不同证书的 `cert_name` 必须全局唯一，建议使用域名相关命名便于识别；
- 每个证书都应设置描述，标注域名、用途、颁发机构；
- 默认证书建议选择覆盖范围最广、最稳定的证书；
- 更新某张非默认证书不会影响其他域名的 HTTPS 服务。

### 22.6.3 证书选择顺序

BFE 在 TLS 握手阶段依据客户端发送的 SNI 选择证书，其选择逻辑可概括为：

1. 在 `CertConf` 中查找与 SNI 匹配的证书（按证书中的 SAN/CN 匹配）；
2. 若找到唯一匹配，则使用该证书；
3. 若未找到匹配，或客户端未发送 SNI，则使用 `Default` 指定的默认证书；
4. 若默认证书也不存在，TLS 握手失败。

因此，管理员在配置多域名时，应确保证书的 SAN 列表覆盖所有目标域名，并为未覆盖的访问场景预留一张可靠的默认证书。

---

## 22.7 完整配置示例

以下示例展示如何为一个 AI 产品完成域名绑定、证书上传和 HTTPS 接入验证。

### 22.7.1 创建证书

```bash
curl -X POST "http://ai-gateway-api:8183/open-api/v1/certificates" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${ADMIN_TOKEN}" \
  -d '{
    "cert_name": "cert_api_example_com",
    "description": "api.example.com 生产证书",
    "is_default": true,
    "cert_file_content": "-----BEGIN CERTIFICATE-----\nMIIDXTCCAkWgAwIBAgIJAKoK/heBjcOuMA0GCSqGSIb3DQEBCwUAMEUxCzAJBgNV\nBAYTAkNOMQ4wDAYDVQQIDAVDaGluYTEQMA4GA1UEBwwHQmVpamluZzEUMBIGA1UE\nCgwLZXhhbXBsZS5jb20xFDASBgNVBAMMC2V4YW1wbGUuY29tMB4XDTE0MDUyNjA3\n...\n-----END CERTIFICATE-----",
    "key_file_content": "-----BEGIN RSA PRIVATE KEY-----\nMIIEpQIBAAKCAQEA3Tz2mvC3D1tRkX7B8kYwQ8u8kYwQ8u8...\n-----END RSA PRIVATE KEY-----"
  }'
```

### 22.7.2 BFE 证书映射配置

AI Gateway API 通过 InnerAPI 导出给 BFE 的 `server_cert_conf.data` 示例如下：

```json
{
    "Version": "20250101000000",
    "Config": {
        "Default": "cert_api_example_com",
        "CertConf": {
            "cert_api_example_com": {
                "ServerCertFile": "tls_conf_20250101000000/cert_api_example_com/server.crt",
                "ServerKeyFile": "tls_conf_20250101000000/cert_api_example_com/server.key",
                "OcspResponseFile": ""
            }
        }
    }
}
```

Conf Agent 会将证书文件写入 `tls_conf_{version}/{cert_name}/` 目录下，并调用 BFE 热加载接口使其生效。

### 22.7.3 域名与路由配置

对应的 `server_data_conf` 中 HostTable 与 RouteTable 片段如下：

```json
{
    "Version": "20250101000000",
    "HostTable": {
        "Version": "20250101000000",
        "DefaultProduct": "AI_product",
        "Hosts": {
            "host-api-example": ["api.example.com", "api-cn.example.com"]
        },
        "HostTags": {
            "AI_product": ["host-api-example"]
        }
    },
    "RouteTable": {
        "Version": "20250101000000",
        "BasicRule": {},
        "ProductRule": {
            "AI_product": [
                {
                    "Cond": "req_host_in(\"api.example.com\")",
                    "ClusterName": "deepseek-cluster"
                }
            ]
        }
    },
    "ClusterConf": { ... }
}
```

### 22.7.4 验证证书已生效

配置下发并热加载后，通过以下命令验证：

```bash
curl -v https://api.example.com/v1/chat/completions \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek-v3","messages":[{"role":"user","content":"hello"}]}'
```

关注输出中的证书信息，例如：

```text
* Server certificate:
*  subject: CN=api.example.com
*  start date: Aug 23 16:02:31 2025 GMT
*  expire date: Aug 23 16:02:31 2026 GMT
*  subjectAltName: host "api.example.com" matched cert's "api.example.com"
*  issuer: C=CN; O=Example CA; CN=Example Intermediate CA
*  SSL certificate verify ok.
```

若能看到类似输出，说明域名绑定、证书上传和 HTTPS 监听均已生效。

---

## 本章小结

- 域名绑定通过 AI Gateway API 维护，最终由 InnerAPI 导出为 BFE 的 `server_data_conf` 中的 HostTable 与 RouteTable。
- TLS 证书通过 OpenAPI `/certificates` 上传，控制面会校验证书与私钥的格式、配对关系，并自动解析过期时间。
- 全局必须且只能有一张默认证书，用于 SNI 未匹配时的 TLS 回退；默认证书不能被直接删除。
- 建议建立证书过期监控机制，在证书到期前完成续期和验证，避免 HTTPS 服务中断。
- HTTPS 验证可使用 `curl -v` 检查证书链、SAN、Issuer 等信息；常见问题多与域名绑定、证书链完整性、默认证书缺失有关。
- 多域名场景可通过单证书多 SAN 或多证书独立配置实现，BFE 会依据 SNI 自动选择最佳证书。

---

## 参考文档

- `ai-gateway-api/design-docs/api-define/OpenAPI接口定义/certificates.md`
- `ai-gateway-api/design-docs/api-define/InnerAPI接口定义/server-cert-conf.md`
- `ai-gateway-api/design-docs/api-define/InnerAPI接口定义/server-data-conf.md`
- `bfe/docs/zh_cn/configuration/tls_conf/server_cert_conf.data.md`
- [第五章 壬远AI网关架构设计](../design/chapter05-system-architecture.md)
- [第七章 数据面转发设计：BFE](../design/chapter07-data-plane-design.md)
