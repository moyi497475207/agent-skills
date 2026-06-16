---
name: ops-data-multistep-query
description: "通过 ManageOne AI助手（智能助手）对复杂运维问题做多步拆解式数据检索与分析：登录 ManageOne 运维门户、点击进入 AI 助手、切换到指定云平台(region)、把复杂问题拆成原子子问题逐个检索、合并结果并标注异常。使用 chrome-devtools CLI 实现浏览器自动化。当用户提到'多步数据检索'、'拆解查询'、'宿主机CPU趋势'、'TOP查询+趋势'、'复杂运维数据多维分析'、'指定region查询'、'AI助手多轮问答'、'先查TOP再查明细/趋势'时使用此技能。即使用户没明确说'拆解'，只要涉及需要在 AI 助手中连续提出多个相关原子问题并汇总结果的场景，都应触发此技能。"
---

# 运维数据多步拆解查询 - 复杂运维问题的多轮数据检索与聚合

通过 chrome-devtools CLI 自动化浏览器，登录 ManageOne 运维门户并进入 AI 助手、切换到指定云平台，把一个复杂运维问题拆解为多个原子子问题逐个检索，最后把各轮结果合并为统一结论（含图表/异常标注）。适用于 AI 助手只支持"原子问题"、无法一次回答复合问题的场景。

## 前置条件

- 已全局安装 `chrome-devtools-mcp`（`npm i chrome-devtools-mcp@latest -g`）
- Chrome 浏览器可用
- `chrome-devtools status` 能正常返回（服务可由本技能自启，见步骤 1）

## 配置参数

用户可在调用时通过参数覆盖默认值。以下是所有可配置项：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `targetUrl` | `https://oc-861vt.861vt.com:31943/momaintenancewebsite/uniportal/#/home` | ManageOne 运维门户地址（登录入口） |
| `cloudPlatform` | `region861vt` | 要切换到的云平台名称；留空则跳过切换，沿用当前平台 |
| `username` | `bss_admin` | 登录用户名 |
| `password` | `` | 登录密码（⚠️ 敏感信息，建议用户每次传入；生产环境务必改用真实账号） |
| `taskDescription` | （必填，无默认值） | 需要拆解的复杂运维问题，例如"查询今天平均CPU利用率TOP3的宿主机，列出一周趋势" |
| `maxWaitSeconds` | `180` | 单个原子问题等待 AI 回答的最大秒数 |
| `pollIntervalSeconds` | `15` | 轮询 AI 回答状态的间隔秒数 |
| `keepPageOnFinish` | `false` | 为 `true` 时保留本技能创建/登录的页面，跳过步骤 10 的 `close_page`（适用于想留下页面继续人工查看） |

**参数传递方式**：用户在指令中直接说明，例如：
> "查一下今天CPU利用率TOP3的宿主机，再看看每台最近一周的趋势，合并成图表"（taskDescription + 隐含 region861vt）
> "切换到 region861vt 平台，问AI助手二层和三层组网的区别，再问推荐配置"

## 工作流

按以下步骤严格执行。**每一步操作前都重新 `take_snapshot` 从快照中动态获取 UID，绝不硬编码 UID。**

### 步骤 1：检查服务状态并记录起始状态

```bash
chrome-devtools status          # 记录：MCP 是否已在运行 → mcpWasRunning
chrome-devtools list_pages      # 记录：起始已存在的页面索引/标题/URL → initialPages
```

记录起始状态供步骤 10 判定清理范围：**仅清理本次技能新建的资源**。

- 若 `mcpWasRunning == false`：本技能自启 MCP，且**必须带 SSL 忽略参数**（该站点多级自签名证书）：

```bash
# 自启 MCP：--headless false 便于观察；--acceptInsecureCerts true 跳过自签名证书
chrome-devtools start --acceptInsecureCerts true --headless false
```

> 若 `mcpWasRunning == true`（共享 MCP，可能未带 `--acceptInsecureCerts`）：不要重启（会打断其他流程），改用步骤 2 的点击式证书绕过。

### 步骤 2：访问 targetUrl + 处理 SSL 证书

```bash
chrome-devtools navigate_page --url "<targetUrl>" --timeout 30000
chrome-devtools take_snapshot
```

若快照含 `ERR_CERT` / `隐私设置错误` / `您的连接不是私密连接`（自启 MCP 带 acceptInsecureCerts 时通常不会出现），用**循环点击式绕过**（站点会多级重定向到 auth 域名，每次跳转都可能触发新证书错误，最多循环 5 次）：

```bash
# 循环体（最多 5 次）：
chrome-devtools click "<'高级'按钮 uid>"          # 展开详情
chrome-devtools take_snapshot
chrome-devtools click "<'继续前往...(不安全)'链接 uid>"
sleep 5
chrome-devtools take_snapshot                      # 仍含证书错误则继续循环
```

证书处理完成后页面会落到**登录页**。

### 步骤 3：登录

```bash
chrome-devtools take_snapshot
```

判断是否登录页：快照同时含 `textbox`、`用户名`、`密码`。若已是登录后状态，跳过本步。

```bash
chrome-devtools fill "<'用户名'输入框 uid>" "<username>"
chrome-devtools fill "<'密码'输入框 uid>" "<password>"
sleep 2                                   # 等表单校验，登录按钮可能短暂 disabled
chrome-devtools take_snapshot             # fill 后 DOM 可能变化，重新取登录按钮 uid
chrome-devtools click "<'登 录'按钮 uid>"
sleep 8
chrome-devtools take_snapshot             # 确认登录成功
```

确认登录成功：登录框消失，进入 **ManageOne 运维门户首页**（能看到 `ManageOne AI助手` 等应用入口）。若仍在登录页，重新取快照再点一次登录按钮（见 troubleshooting）。

### 步骤 4：点击"ManageOne AI助手"跳转到 AI 助手

```bash
chrome-devtools take_snapshot
```

在门户首页找到 `ManageOne AI助手` 应用入口（通常是 link / button / card，文本为 `ManageOne AI助手`），点击进入：

```bash
chrome-devtools click "<'ManageOne AI助手' 入口元素 uid>" --includeSnapshot true
sleep 5
chrome-devtools take_snapshot
```

确认已跳转到 AI 助手：URL 变为 AI 助手站点（如 `moaiassist.../aidoer/website/#/homepage`），页面出现 `ManageOne AI助手` 标题与 `智能助手` 入口。

> 跳转可能跨域触发新的证书错误；若快照含证书错误，按步骤 2 的循环点击式绕过处理。

### 步骤 5：切换云平台（若 cloudPlatform 非空）

```bash
chrome-devtools take_snapshot
```

在导航栏找到云平台选择器：一个含**当前平台代号**（如 `per3W`、`region861vt`，通常在 banner 区域的 generic 元素里）的可点击容器。

```bash
chrome-devtools click "<云平台选择器容器 uid>" --includeSnapshot true   # 打开下拉
```

下拉打开后会出现搜索框 + 候选列表（StaticText 形式）。点击与 `cloudPlatform` 文本一致的候选项：

```bash
chrome-devtools click "<与 cloudPlatform 文本匹配的候选 uid>" --includeSnapshot true
```

确认：选择器文本已更新为 `cloudPlatform` 的值。

### 步骤 6：打开智能助手对话框

```bash
chrome-devtools take_snapshot
chrome-devtools click "<'智能助手' StaticText uid>" --includeSnapshot true
sleep 3
chrome-devtools take_snapshot
```

确认对话框已打开：快照出现问候语 `您好，我是ManageOne AI助手` 与输入框 `您可以用"产品+问题"的形式描述问题`。若仍在加载，再等 3 秒重新取快照。

### 步骤 7：拆解复杂问题为原子子问题

AI 助手只支持**原子问题**，复合问题需拆解。由 AI（你）对 `taskDescription` 做推理拆解，常见模式：

- **TOP → 明细/趋势**：先 `查询今天平均CPU利用率TOP3的宿主机`（得到实体列表），再对每个实体 `查询宿主机{名称}最近一周的CPU使用率趋势，用表格展示`。
- **多维度并列**：分别问 CPU、内存、磁盘 IO，再横向对比。
- **先定位后下钻**：先查"哪些对象异常"，再逐个查"为什么异常"。

产出有序的子问题清单 `Q1, Q2, ... Qn`，记下每轮之间需要从前一轮结果中提取的**变量**（如实体名称）。

> **趋势/对比类问题务必追加"用表格展示"**：AI 助手对此类问题默认渲染 ECharts 图表（canvas/SVG），DOM 无 `<table>`、无法抓取原始数据。在问句末尾加"用表格展示"（如 `...最近一周的CPU使用率趋势，用表格展示`）才会输出表格，步骤 8.3 才能 `evaluate_script` 抓到逐日/逐项原始数据。详见 [references/troubleshooting.md](references/troubleshooting.md) 第 15 条。

### 步骤 8：逐个提交子问题并提取结果

对每个子问题 `Qi`（执行 n 轮，每轮如下）：

**8.1 提交问题**（⚠️ 多行输入框用"点击→输入→单独回车"，`--submitKey` 在该控件上不可靠）：

```bash
chrome-devtools take_snapshot
chrome-devtools click "<输入框 uid>"        # 必须先点击获取焦点（每轮都点）
chrome-devtools type_text "<Qi 全文>"
chrome-devtools take_snapshot               # 确认输入框 value 已写入、字数计数>0
chrome-devtools press_key "Enter"           # 单独按回车提交
```

**8.2 轮询等待回答**（数据检索类问题耗时 30s ~ 3min）：

```bash
sleep <pollIntervalSeconds>
chrome-devtools take_snapshot
```

回答经过若干阶段：`分析中/深度思考中` → `正在筛选可用数据集` → `数据来源: ...` → `正在生成查询语句` → `正在执行查询` → `分析完成`（出现结果）。每轮 `take_snapshot` 判断：

- 仍含 `分析中` / `深度思考中` / `正在生成` / `正在执行` → 继续等待（累计时长 < `maxWaitSeconds`）。
- 出现 `分析完成` + 结果文本/表格 → 本轮完成，进入 8.3 提取。
- 出现 `抱歉，助手暂时无法回答您的问题` → **重新提交一次** Qi（最多重试 1 次）。
- 出现 `数据检索失败，请联系系统管理员处理` → **不要判定为失败**：系统通常会自动用另一个数据集（如 `SYS.SERVER物理服务器性能分析数据集`）重试并在下方追加新的分析块，继续轮询，直到出现 `分析完成` 的成功结果或累计超时。

**8.3 提取结果**：结果文本出现在 `分析完成` 之后的 StaticText 行（常含一段总结句 + 可选表格）。结构化数据用 `evaluate_script` 抓表格更稳（前提是提问时带了"用表格展示"，见步骤 7）：

```bash
chrome-devtools evaluate_script "() => {
  const rows = document.querySelectorAll('table tbody tr');
  return Array.from(rows).map(r => r.innerText);
}"
```

从本轮结果中提取**下一轮所需的变量**（如宿主机名称），回填到 `Q(i+1)` 模板。

### 步骤 9：合并结果并产出最终结论

把 n 轮提取的数据按 `taskDescription` 的目标聚合：

- 趋势/明细合并为统一图表（如 HTML/Chart.js 区间图）或对照表；
- 标注异常时段（峰值/谷值对应日期）与可能原因（基于负载形态推断，注明"AI 未返回根因，建议结合日志核查"）；
- 说明数据粒度局限（如 AI 只返回周 max/min 而非逐日序列，需如实呈现不编造）。

把最终结论返回给用户，必要时落盘为文件（如 `<task>.html`）。

### 步骤 10：资源清理与收尾

**无论前面步骤成功或失败，都执行本步骤。** 仅清理本次技能自己启动的资源：

```bash
chrome-devtools list_pages          # 重新获取索引（运行中可能变化），按 URL/标题匹配
# 仅当工作页面不在步骤 1 的 initialPages 中、且 keepPageOnFinish != true 时：
chrome-devtools close_page <createdPageIndex>

# 仅当步骤 1 的 mcpWasRunning == false（本技能自启了 MCP）时：
chrome-devtools stop

# 校验
chrome-devtools status              # 若执行了 stop，预期返回未运行
chrome-devtools list_pages          # 若执行了 close_page，预期新建页面已消失
```

**判定表**：

| 起始状态 | 结束动作 |
|----------|----------|
| 复用已有页面 + MCP 已在运行 | 不关闭页面、不停止 MCP |
| 新建/登录页面 + MCP 已在运行 | `close_page` 该页面；不停止 MCP |
| 复用页面 + 本技能启动了 MCP | 不关闭页面；`stop` |
| 新建/登录页面 + 本技能启动了 MCP | `close_page` + `stop` |

> 复用的已登录页面与共享 MCP 属于会话/用户，关闭它们会丢弃登录态并打断其他流程，**严禁清理**。stop 后该 MCP 启动的浏览器（临时配置）会随之回收；不要误杀用户日常 Chrome（用命令行 user-data-dir 区分）。

## 常见问题与解决

遇到问题时，参考 [references/troubleshooting.md](references/troubleshooting.md)。

## 注意事项

1. **绝不硬编码 UID**：每步操作前重新 `take_snapshot`，用语义文本（如"找'高级'按钮"）定位。
2. **提交用单独回车**：该输入框是多行控件，`type_text --submitKey Enter` 不可靠，必须 `click` → `type_text` → `press_key Enter` 三步，且每轮重新点击获取焦点。
3. **"数据检索失败"非终点**：站点数据检索首轮常失败但会自动换数据集重试，持续轮询直到 `分析完成` 或累计超时。
4. **趋势/对比问题带"用表格展示"**：否则 AI 渲染 ECharts 图表，抓不到原始数据（见步骤 7 与 troubleshooting 第 15 条）。
5. **拆解由 AI 推理完成**：本技能提供登录/进入助手/切换/提交/提取/清理的执行框架，子问题拆解与变量回填由 AI 按步骤 7 策略完成。
6. **如实呈现数据粒度**：AI 返回什么粒度（如周 max/min）就用什么粒度，不编造逐日序列。
7. **密码安全**：默认密码仅限测试环境，生产环境由用户每次传入真实凭据。
