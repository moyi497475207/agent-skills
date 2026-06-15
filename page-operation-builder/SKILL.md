---
name: page-operation-builder
description: "根据 AI 已手动执行的 chrome-devtools 浏览器操作步骤，自动生成可复用的页面操作 Skill。当用户提到'生成 Skill'、'创建页面操作 Skill'、'把操作记录变成 Skill'、'封装浏览器操作'、'制作自动化 Skill'，或者在完成了一轮 chrome-devtools 操作后想将其固化为可复用 Skill 时使用此技能。即使用户没有明确说'生成 Skill'，只要在对话中完成了浏览器自动化操作并希望保存为可复用流程，都应触发此技能。"
---

# Page Operation Builder - 页面操作 Skill 生成器

分析当前会话中已执行的 chrome-devtools 操作记录，按照标准模板自动生成完整的页面操作 Skill（SKILL.md + troubleshooting.md）。

## 前置条件

- 当前会话中已通过 chrome-devtools CLI 完成了一套浏览器操作
- 操作流程已验证可行（至少成功执行过一次）

## 工作流

### 步骤 1：收集操作记录

从当前会话历史中提取所有 `chrome-devtools` 命令及其执行结果。重点关注：

- 执行了哪些命令（navigate_page、click、fill、upload_file 等）
- 每步的输出/快照内容
- 遇到的异常及处理方式（SSL 证书、登录、超时等）
- 操作的先后顺序和条件分支

如果会话中操作记录不完整，主动询问用户补充关键步骤。

### 步骤 2：分析与归类

将收集到的操作记录按以下维度分析：

1. **前置条件**：哪些环境准备是必须的（已登录、特定页面状态等）
2. **核心步骤**：哪些是主流程步骤，哪些是条件分支
3. **异常处理**：遇到了哪些异常（SSL、登录过期、加载延迟等），如何处理的
4. **配置参数**：哪些值可能因用户或环境而变化（URL、凭据、文件路径等）
5. **异步等待点**：哪些操作需要等待服务端响应

将每个步骤归类到已知操作模式中。读取 [references/pattern-library.md](references/pattern-library.md) 了解 8 种常见模式及其标准写法。

### 步骤 3：确定 Skill 元信息

与用户确认以下信息（如用户已提供则直接使用）：

| 信息 | 说明 | 示例 |
|------|------|------|
| Skill 名称 | 英文短名，kebab-case | `knowledge-upload` |
| 显示名称 | 中文功能描述 | `知识管理文件上传` |
| 触发词 | 用户可能的表达方式 | `上传知识`、`知识管理上传` |
| 兜底场景 | 何时也应触发 | `涉及向 ManageOne 上传文档` |

### 步骤 4：生成 SKILL.md

按照 [references/skill-template.md](references/skill-template.md) 中的模板生成 SKILL.md 文件。

生成时的关键原则：

- **每一步都通过快照确认状态**：不假设页面状态，操作前后都获取快照
- **UID 不硬编码**：从快照中动态获取，用语义描述代替（如"找到'高级'按钮"）
- **错误分支清晰**：SSL 错误 → 处理证书；登录页 → 执行登录
- **异步操作有等待**：服务端操作需 sleep 后再检查结果
- **`--includeSnapshot true`**：常用操作加上此参数减少快照调用次数
- **配置参数表完整**：所有可变参数列出默认值和说明
- **description 触发词丰富**：包含多种同义表达和兜底描述

### 步骤 5：生成 troubleshooting.md

根据操作中遇到的问题和潜在风险，生成 troubleshooting.md。参考 [references/pattern-library.md](references/pattern-library.md) 中"错误检测与自动重试"模式。

每条问题按"现象 → 原因 → 解决"三段式组织。至少覆盖：

- chrome-devtools 服务相关（命令未找到、服务未运行）
- 页面导航相关（SSL 证书、超时）
- 元素操作相关（元素找不到、UID 失效）
- 业务逻辑相关（根据实际操作中遇到的问题）

### 步骤 6：输出文件

将生成的文件输出到当前工作目录：

```
<skill-name>/
├── SKILL.md
└── references/
    └── troubleshooting.md
```

输出后告知用户文件位置，并提示：
- 可将目录移动到 `.claude/skills/` 下以供 AI 自动触发
- 建议用默认参数测试一遍生成的 Skill
- 如需调整触发词或参数，直接编辑 SKILL.md 的 frontmatter 和配置参数表

## 注意事项

1. 生成的 Skill 中绝不硬编码 UID，每步操作前必须重新获取快照
2. 配置参数从用户自然语言指令中提取，不使用配置文件
3. description 字段是触发核心机制，宁长勿短（50-150 字），包含多种同义表达和兜底描述
4. 工作流步骤建议 5-15 步，太少太粗略，太多需拆分子流程
5. 如果操作流程包含敏感信息（密码等），在配置参数表中使用占位符并添加安全提示
