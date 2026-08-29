# 壬远AI网关书籍写作规范

## 一、目标读者

- **使用者**：运维工程师、平台工程师、AI应用开发者，关心如何部署、配置和使用壬远AI网关。
- **开发者**：后端工程师、网关/基础设施开发者，关心代码实现、扩展开发和贡献代码。

## 二、语言与风格

1. 全书使用简体中文撰写。
2. 技术术语首次出现时给出英文原名，后续可只写中文。例如："控制面（Control Plane）"。
3. 保持技术文档风格：清晰、准确、简洁，避免口语化表达。
4. 章节标题使用 Markdown 二级（`##`）和三级（`###`）标题组织。

## 三、章节结构

每章建议包含以下部分（可根据内容适当调整）：

```markdown
# 第X章 章节标题

## 本章目标
简要说明本章要解决的问题和读者能获得的收获。

## 核心概念/背景
介绍相关背景知识。

## 详细内容
分小节展开。

## 配置/代码示例
给出实际操作或代码示例。

## 本章小结
总结本章要点。

## 参考文档
列出相关的设计文档、代码路径或外部资料。
```

## 四、引用规范

1. 引用项目内部文档时，使用相对路径或绝对仓库路径：
   - `ai-gateway-api/design-docs/sys-design/总体设计文档.md`
   - `bfe/bfe_modules/mod_ai_route/`
2. 引用代码时，标注文件路径和关键函数/类型名，例如：
   - `ai-gateway-api/main.go` 中的 `stateful.LoadConfig`
   - `bfe/bfe_modules/mod_ai_route/mod_ai_route.go` 中的 `AiRouteConf`
3. 引用其他章节时，使用相对链接，例如：
   - [第六章 控制面核心设计：AI Gateway API](./chapter06-control-plane-design.md)

## 五、图表规范

1. 优先使用 ASCII 图或 Mermaid 文本图（便于版本管理）。
2. 如需图片，保存到 `images/` 目录，并在文中使用相对路径引用。
3. 数据流图、架构图、模块交互图是重点，建议每章至少包含 1-2 个图。

## 六、代码与配置示例

1. 代码块必须标注语言类型（`go`、`toml`、`yaml`、`json`、`bash` 等）。
2. 配置示例应来自实际项目，必要时做简化但保持语义正确。
3. 关键操作命令需给出可执行的示例。

## 七、一致性要求

1. 术语统一：
   - AI Gateway API / 控制面 / 控制平面
   - BFE / 数据面 / 数据平面
   - API-Key（注意大小写）
   - Provider / Cluster / Entity / QuotaPlan / RateLimitPolicy
2. 组件名称统一：AI Gateway API、Dashboard、BFE、Conf Agent、Service Controller。
3. 路径统一：章节文件命名遵循 `chapterXX-short-title.md`。

## 八、章节对应关系

设计篇、操作篇、实现篇存在前后对应关系，写作时注意交叉引用：

- 第五章（架构设计）→ 第十六章（安装部署）→ 第二十四章（代码组织与启动流程）
- 第六章（控制面设计）→ 第十七章（控制台基础操作）→ 第二十五章（接口层实现）
- 第九章（Provider与Cluster设计）→ 第十八、十九章（Provider/Cluster配置）→ 第二十八章（mod_ai_route实现）
- 第十一章（配额与限流设计）→ 第二十、二十一章（API-Key/限流配置）→ 第二十九、三十章（mod_ai_token_auth / mod_ai_rate_limit实现）
