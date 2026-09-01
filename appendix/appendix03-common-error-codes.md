# 附3 常见错误码

BFE 数据面在 AI 网关场景下返回的错误码、响应格式、触发条件及排查建议，已由 BFE 官方文档完整维护。本书不再重复罗列，读者可通过以下链接查看 v1.8.6 版本的权威定义。

## v1.8.6 版本错误码文档

- [BFE AI 网关错误码说明](https://github.com/bfenetworks/bfe/blob/refs/tags/v1.8.6/docs/zh_cn/sys_design/ai_error_codes.md)

该文档覆盖以下内容：

- 认证与准入层错误码（如 `NO_API_KEY`、`INVALID_API_KEY`、`KEY_DISABLED`、`SUBNET_NOT_ALLOWED`、`MODEL_NOT_ALLOWED`）
- 限流检查层错误码（如 `RPM_LIMIT_EXCEEDED`、`TPM_LIMIT_EXCEEDED`、`CONCURRENCY_LIMIT_EXCEEDED`）
- 配额扣减层错误码（如 `QUOTA_EXHAUSTED`、`QUOTA_EXPIRED`、`INTERNAL_QUOTA_ERROR`）
- 转发与协议适配层错误码（如 `PROVIDER_PROTOCOL_MISMATCH`）
- 标准错误响应体结构与字段说明
- 预留错误码列表
- 错误码与访问日志字段的关联
- 按 HTTP 状态码的排查建议

## 本书相关章节

- [第十六章 安全设计](../design/chapter16-security-design.md) 中介绍了错误响应体与安全审计日志字段。
- [第十八章 控制台基础操作](../operation/chapter18-dashboard-basics.md) 中介绍了常见控制台错误提示的对照思路。
- [第二十一章 API-Key 与配额配置](../operation/chapter21-apikey-and-quota-config.md) 中介绍了配额耗尽、限流触发等问题的排查方法。
