# 壬远AI网关：原理、设计与实现

本书面向**壬远AI网关的使用者和开发者**，系统介绍壬远AI网关的技术背景、核心原理、系统设计、操作方法、代码实现与扩展开发。

## 全书目录

### 背景篇
- [第一章 壬远AI网关简介](./background/chapter01-what-is-rainway-ai-gateway.md)

### 原理篇
- [第二章 AI网关技术概述](./principle/chapter02-ai-gateway-overview.md)
- [第三章 大模型服务接入的挑战](./principle/chapter03-llm-access-challenges.md)
- [第四章 AI网关的路由与调度原理](./principle/chapter04-routing-and-scheduling.md)

### 设计篇
- [第五章 壬远AI网关架构设计](./design/chapter05-system-architecture.md)
- [第六章 控制面核心设计：AI Gateway API](./design/chapter06-control-plane-design.md)
- [第七章 数据面转发设计：BFE](./design/chapter07-data-plane-design.md)
- [第八章 API-Key与认证授权设计](./design/chapter08-auth-and-apikey.md)
- [第九章 Provider与Cluster设计](./design/chapter09-provider-and-cluster.md)
- [第十章 AI路由规则设计](./design/chapter10-ai-route-rules.md)
- [第十一章 配额与限流设计](./design/chapter11-quota-and-rate-limit.md)
- [第十二章 模型定价与成本核算设计](./design/chapter12-model-pricing.md)
- [第十三章 配置导出与版本控制设计](./design/chapter13-config-export-and-version-control.md)
- [第十四章 可观测性设计](./design/chapter14-observability.md)
- [第十五章 安全设计](./design/chapter15-security-design.md)

### 操作篇
- [第十六章 安装部署](./operation/chapter16-installation-and-deployment.md)
- [第十七章 控制台基础操作](./operation/chapter17-dashboard-basics.md)
- [第十八章 Provider与模型配置](./operation/chapter18-provider-and-model-config.md)
- [第十九章 Cluster与路由配置](./operation/chapter19-cluster-and-route-config.md)
- [第二十章 API-Key与配额配置](./operation/chapter20-apikey-and-quota-config.md)
- [第二十一章 限流策略配置](./operation/chapter21-rate-limit-config.md)
- [第二十二章 域名与证书配置](./operation/chapter22-domain-and-cert-config.md)
- [第二十三章 配置热加载与升级](./operation/chapter23-hot-reload-and-upgrade.md)

### 实现篇
- [第二十四章 代码组织与启动流程](./implementation/chapter24-code-layout-and-startup.md)
- [第二十五章 接口层实现：OpenAPI与InnerAPI](./implementation/chapter25-endpoints-implementation.md)
- [第二十六章 模型层实现：Manager与Storager模式](./implementation/chapter26-model-layer-implementation.md)
- [第二十七章 存储层实现：DAO与Storage](./implementation/chapter27-storage-layer-implementation.md)
- [第二十八章 AI路由模块实现：mod_ai_route](./implementation/chapter28-mod-ai-route.md)
- [第二十九章 Token认证与配额模块实现：mod_ai_token_auth](./implementation/chapter29-mod-ai-token-auth.md)
- [第三十章 限流模块实现：mod_ai_rate_limit](./implementation/chapter30-mod-ai-rate-limit.md)
- [第三十一章 请求体处理模块实现：mod_body_process](./implementation/chapter31-mod-body-process.md)
- [第三十二章 Conf Agent实现](./implementation/chapter32-conf-agent-implementation.md)

### 开发篇
- [第三十三章 如何扩展壬远AI网关](./develop/chapter33-how-to-extend.md)
- [第三十四章 如何向壬远AI网关贡献代码](./develop/chapter34-how-to-contribute.md)

### 附录篇
- [附1 OpenAPI接口速查](./appendix/appendix-a-openapi-reference.md)
- [附2 InnerAPI配置导出格式](./appendix/appendix-b-innerapi-formats.md)
- [附3 常见错误码](./appendix/appendix-c-error-codes.md)
- [附4 术语表](./appendix/appendix-d-glossary.md)

## 写作规范

详见 [writing-guide.md](./writing-guide.md)。

## 参考项目

- `ai-gateway-api/`：控制面核心组件
- `bfe/`：数据面转发引擎
- `conf-agent/`：配置代理
