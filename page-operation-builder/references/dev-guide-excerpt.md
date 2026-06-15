# 页面操作 Skill 开发方法论精要

## 核心原则

页面操作 Skill 基于 `chrome-devtools-mcp` CLI 命令驱动浏览器完成自动化操作。核心特征：

- 基于 CLI 命令，在 Bash 中直接调用，天然适配 Skill 的执行模型
- 基于 Accessibility Tree 的元素定位，比 CSS 选择器更稳定
- 工作流指令写在 SKILL.md 中，由 AI 按步骤执行

## 开发流程：探索 → 设计 → 编写 → 测试

### 探索阶段的关键发现清单

每次探索完成后，整理以下内容：

1. **前置条件**：需要哪些环境准备（已登录、特定页面等）
2. **必要步骤**：哪些步骤是核心流程、哪些是条件分支
3. **异常情况**：SSL 证书、登录过期、元素加载延迟等
4. **配置项**：哪些参数可能因用户或环境而变化（URL、凭据、文件路径等）
5. **异步点**：哪些操作需要等待服务端响应

### 设计决策清单

| 决策项 | 选项 | 判断依据 |
|--------|------|----------|
| 是否需要 `scripts/` 目录 | 需要 / 不需要 | 交互式流程（每步依赖上一步的动态 UID）不需要 |
| 配置参数放哪里 | SKILL.md 内 / 配置文件 | 参数少（<10 个）放 SKILL.md |
| 是否需要 `references/` | 需要 / 不需要 | 常见问题 >3 个时单独存放 |
| 工作流步骤数 | 建议 5-15 步 | 太少太粗略，太多需拆分子流程 |

### 编写要点

1. **frontmatter 触发词设计**：包含多种同义表达和兜底描述
2. **配置参数表**：所有可变参数列出默认值和说明
3. **工作流设计原则**：
   - 每一步都通过快照确认状态
   - UID 不硬编码
   - 错误分支清晰
   - 异步操作有等待
   - `--includeSnapshot true` 减少快照调用

### 测试场景

| 测试类型 | 测试内容 |
|----------|----------|
| 默认参数 | 不传任何参数，验证默认流程能走通 |
| 覆盖参数 | 传入自定义参数，验证参数提取正确 |
| 异常恢复 | 模拟 SSL 错误、登录失败等异常分支 |
| 边界情况 | 空文件列表、超大文件、网络超时 |
| 重复执行 | 连续执行两次，验证第二次的页面状态处理 |

## UID 动态获取最佳实践

- 每次操作前获取新快照：UID 在每次 `take_snapshot` 后重新编号
- 通过语义定位元素：通过元素的角色（button、link、textbox）和文本内容来识别目标
- 善用 `--includeSnapshot true`：在 click/fill/upload_file 后自动返回新快照
- 不要缓存 UID：任何操作都可能导致 DOM 变化，使旧 UID 失效

## chrome-devtools CLI 常用命令速查

```bash
# 服务管理
chrome-devtools start
chrome-devtools status

# 页面操作
chrome-devtools list_pages
chrome-devtools navigate_page --url "<url>" --timeout 30000
chrome-devtools take_snapshot
chrome-devtools click "<uid>"
chrome-devtools fill "<uid>" "<text>"
chrome-devtools upload_file "<uid>" "<path>"
chrome-devtools press_key "Enter"

# 调试
chrome-devtools take_screenshot
chrome-devtools list_console_messages --types error
chrome-devtools evaluate_script "() => document.title"

# 有用的参数
--includeSnapshot true    # 操作后自动返回新快照
--output-format json      # JSON 格式输出
```
