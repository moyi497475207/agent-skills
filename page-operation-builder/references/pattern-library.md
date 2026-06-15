# 常见页面操作模式库

开发页面操作 Skill 时，将操作步骤归类到以下 8 种模式中，使用对应的标准写法。

## 模式 1：导航 + SSL 证书处理

**适用场景**：目标站点使用自签名证书，首次访问会出现证书错误。

**标准命令序列**：

```bash
chrome-devtools navigate_page --url "<url>" --timeout 30000
# 如果出现证书错误
chrome-devtools take_snapshot
chrome-devtools click "<高级按钮uid>"
chrome-devtools take_snapshot
chrome-devtools click "<继续前往链接uid>"
```

**关键注意点**：
- 并非所有站点都有证书问题，设计为条件分支
- 如果启动时加了 `--acceptInsecureCerts` 参数可自动跳过
- 证书处理完成后页面通常会跳转到登录页

## 模式 2：登录认证

**适用场景**：需要通过表单或 SSO 登录才能访问目标页面。

```bash
chrome-devtools take_snapshot
chrome-devtools fill "<用户名输入框uid>" "<username>"
chrome-devtools fill "<密码输入框uid>" "<password>"
chrome-devtools click "<登录按钮uid>" --includeSnapshot true
sleep 3
chrome-devtools take_snapshot
```

**关键注意点**：
- 登录后需等待页面加载（3-5 秒），通过快照确认是否成功
- 密码中的特殊字符不需要转义
- 登录按钮可能短暂 disabled，等待即可
- 如果已有已登录页面（检查 URL 和标题），可直接复用，跳过登录步骤

## 模式 3：表单填写与提交

**适用场景**：需要填写多个表单字段并提交。

```bash
chrome-devtools take_snapshot
chrome-devtools fill "<字段1 uid>" "<值1>"
chrome-devtools fill "<字段2 uid>" "<值2>"
chrome-devtools click "<提交按钮uid>" --includeSnapshot true
sleep 2
chrome-devtools take_snapshot
```

**关键注意点**：
- 下拉选择也用 `fill` 命令，传入选项文本
- 提交后检查是否出现成功/失败提示

## 模式 4：文件上传

**适用场景**：需要通过页面元素上传本地文件。

**单文件上传**：

```bash
chrome-devtools upload_file "<选择文件按钮uid>" "<filepath>" --includeSnapshot true
```

**多文件上传（逐个）**：

```bash
# 对每个文件重复 upload_file 操作
for file in /path/to/files/*.{docx,pdf}; do
  chrome-devtools upload_file "<选择文件按钮uid>" "$file"
  sleep 1
done
chrome-devtools take_snapshot
```

**关键注意点**：
- `upload_file` 一次只能上传一个文件
- 文件路径使用绝对路径
- 观察快照中文件列表的变化，确认每个文件已添加

## 模式 5：等待异步操作完成

**适用场景**：点击按钮后需要等待服务端处理。

```bash
chrome-devtools click "<操作按钮uid>" --includeSnapshot true
sleep 3
chrome-devtools take_snapshot
# 根据快照判断：成功提示 → 完成；错误提示 → 报告；按钮仍 disabled → 继续等待
```

**关键注意点**：
- 不假设操作立即完成，必须轮询确认
- sleep 时长根据操作复杂度调整（3-10 秒）
- 设置最大等待次数，避免无限等待

## 模式 6：数据抓取与导出

**适用场景**：从页面提取结构化数据。

```bash
chrome-devtools take_snapshot
# 或通过 JS 提取特定数据
chrome-devtools evaluate_script "() => {
  const rows = document.querySelectorAll('table tbody tr');
  return Array.from(rows).map(r => r.innerText);
}"
```

**关键注意点**：
- 简单数据从快照文本中解析
- 复杂数据用 `evaluate_script` 执行 JS 提取
- 分页数据需循环翻页采集

## 模式 7：多页面切换与状态复用

**适用场景**：多个标签页间切换，或复用已登录页面。

```bash
chrome-devtools list_pages
# 查找已登录的页面（URL 包含目标域名）
chrome-devtools select_page <index> --bringToFront true
# 如果没有，新建页面
chrome-devtools new_page "<url>"
```

**关键注意点**：
- 复用已登录页面可以跳过登录步骤，节省时间
- `list_pages` 返回页面索引和标题，据此判断状态

## 模式 8：错误检测与自动重试

**适用场景**：操作可能因网络或服务端原因失败。

```bash
chrome-devtools click "<按钮uid>" --includeSnapshot true
# 如果快照中出现错误提示：
#   1. 记录错误信息
#   2. 判断是否可重试（网络超时可重试，参数错误不可重试）
#   3. 可重试则 sleep 后重新执行
#   4. 不可重试则告知用户
chrome-devtools list_console_messages --types error
```

**关键注意点**：
- 区分可重试错误和不可重试错误
- 设置最大重试次数（建议 3 次）
- 控制台错误日志是重要的诊断信息来源

---

## 异步操作等待策略参考

| 操作类型 | 建议等待时间 | 判断方式 |
|----------|-------------|----------|
| 页面导航 | 3-5 秒 | 快照中 URL 变化 |
| 表单提交 | 2-3 秒 | 成功/失败提示出现 |
| 文件上传 | 1-2 秒/文件 | 文件出现在列表中 |
| 服务端处理 | 3-10 秒 | 按钮状态变化或提示出现 |
| 报表生成 | 10-30 秒 | 下载开始或进度条变化 |

## 配置参数设计模式

| 参数类型 | 特征 | 示例 | 默认值设计 |
|----------|------|------|------------|
| URL 类型 | 目标页面地址 | `targetUrl` | 开发/测试环境地址 |
| 凭据类型 | 用户名/密码 | `username`, `password` | 默认账号（注意安全提示） |
| 路径类型 | 文件或目录路径 | `fileDir`, `fileName` | 常用目录 |
| 选项类型 | 布尔开关或枚举 | `uploadAll`, `fileFormat` | 最常用的选项 |

## description 触发词设计技巧

1. **多种同义表达**：覆盖用户可能的各种表达方式
2. **兜底描述**：用自然语言描述兜底条件
3. **场景描述**：描述具体应用场景，而非只列举动词
4. **长度建议**：description 宁长勿短，50-150 字为宜
