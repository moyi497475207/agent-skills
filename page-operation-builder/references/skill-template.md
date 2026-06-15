# 页面操作 Skill 模板

以下是 SKILL.md 的完整模板，`<占位符>` 部分根据实际需求替换。

---

---
name: <skill-name>
description: "<一句话功能描述>。使用 chrome-devtools CLI 实现浏览器自动化。当用户提到'<触发词1>'、'<触发词2>'、'<触发词3>'时使用此技能。即使用户没有明确说'<关键词>'，只要涉及<业务场景>，都应触发此技能。"
---

# <Skill 显示名> - <功能简述>

通过 chrome-devtools CLI 自动化浏览器操作，<1-2 句话说明功能>。

## 前置条件

- 已全局安装 `chrome-devtools-mcp`（`npm i chrome-devtools-mcp@latest -g`）
- Chrome 浏览器已启动
- `chrome-devtools status` 显示服务正在运行

如果服务未运行，先执行 `chrome-devtools start` 启动。

## 配置参数

用户可以在调用时通过参数覆盖默认值。以下是所有可配置项：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `targetUrl` | `https://<默认地址>` | 目标页面 URL |
| `<其他参数>` | <默认值> | <说明> |

支持的文件格式/操作类型：<说明>

**参数传递方式**：用户在指令中直接说明，例如：
> "<示例指令 1>"
> "<示例指令 2>"

## 工作流

按照以下步骤严格执行。每一步通过 `chrome-devtools` CLI 命令完成。

### 步骤 1：检查服务状态并获取页面

```bash
chrome-devtools list_pages
```

<根据页面状态判断后续步骤>

### 步骤 2：导航到目标页面

```bash
chrome-devtools navigate_page --url "<targetUrl>" --timeout 30000
```

### 步骤 3：处理异常（如 SSL 证书、登录等）

<根据实际流程编写条件分支>

### 步骤 4-N：核心业务操作

<每一步遵循"快照 → 判断 → 操作"三段式>

### 步骤 N+1：确认结果并收尾

<检查最终状态、关闭对话框等>

## 常见问题与解决

遇到问题时，参考 [references/troubleshooting.md](references/troubleshooting.md)。

## 注意事项

1. <约束 1>
2. <约束 2>
