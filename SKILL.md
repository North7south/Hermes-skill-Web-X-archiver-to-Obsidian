---
name: article_cleaner_and_saver
description: 自动抓取、清洗、摘要网页文章或推文并保存到 GitHub 私有仓库 Hermes-Inbox。当用户发送 http/https 开头的链接时触发。Twitter/x.com 链接自动替换为 fixupx.com 反爬代理。
version: 2.0.0
author: North7south
license: MIT
metadata:
  hermes:
    tags: [article, cleaner, saver, github, web-scraping, summary, twitter]
    related_skills: []
trigger:
  match: https?://\S+
  match_type: regex
---

# Article Cleaner & Saver v2.0

自动处理用户发送的网页链接或推文链接，提取内容、生成摘要并保存到 GitHub 私有仓库 Hermes-Inbox。支持 Twitter/x.com 反爬代理。

## 触发条件

用户发送或转发包含 `http://` 或 `https://` 开头的网址链接时自动触发。

## URL 预处理（反爬绕过）

在调用 Jina API 之前，先判断 URL 域名：

### Twitter/x.com 链接预处理

当 URL 域名是 `twitter.com` 或 `x.com` 时：

1. **检测**: 检查 URL 中是否包含 `twitter.com` 或 `x.com`
2. **替换**: 将域名中的 `twitter.com` → `fixupx.com`，`x.com` → `fixupx.com`
3. **拼接**: 最终请求 URL = `https://r.jina.ai/https://fixupx.com/...`

**示例**:
- 原始: `https://x.com/user/status/123`
- 处理后: `https://r.jina.ai/https://fixupx.com/user/status/123`

- 原始: `https://twitter.com/user/status/456`
- 处理后: `https://r.jina.ai/https://fixupx.com/user/status/456`

### 普通链接

非 Twitter/x.com 链接直接使用原始 URL：
`https://r.jina.ai/{原始链接}`

## 工作流程

### 步骤 1：提取链接

从用户消息中提取所有 http/https 开头的 URL。支持同时处理多个链接。

### 步骤 2：URL 预处理

对每个链接判断域名：
- **Twitter/x.com 链接** → 替换为 fixupx.com 后再拼接 Jina
- **普通链接** → 直接拼接 Jina

调用命令：
```bash
curl -s "https://r.jina.ai/{处理后的URL}"
```

Jina 接口返回纯 Markdown 格式的清洗后正文内容。

### 步骤 3A：普通文章处理

阅读抓取到的 Markdown 正文，为每篇文章：

1. **提取标题** — 从内容中识别文章标题
2. **提炼 3 句核心摘要** — 用简洁的中文概括文章核心要点
3. **正文摘要（长文/结构性文章）** — 对于含多个章节的长篇文章，在核心摘要后还应为每个主要章节提炼 1-2 句要点，保留原文的结构逻辑
4. **归纳 2-3 个标签** — 用#号标签形式，如 `#AI #加密货币 #金融`

### 步骤 3B：Twitter 推文处理

当链接为 Twitter/x.com 类型时，从清洗后的内容中：

1. **提取作者** — 推文发布者用户名（如 @username）和显示名称
2. **提炼核心观点** — 用简洁的中文概括推文内容，3 句摘要
3. **标签** — 自动包含 `#Twitter` 标签，再根据内容归纳 1-2 个附加标签

### 步骤 4：保存到 GitHub 仓库

调用 GitHub API 将内容写入仓库 `Hermes-Inbox`。

#### 仓库信息
- **仓库名**: `Hermes-Inbox`（私有仓库）
- **文件路径格式**: `articles/{YYYY-MM-DD}-{slug}.md`
  - slug 由标题自动生成（小写、连字符替换空格、去除特殊字符）
- **文件内容格式**（普通文章）:

```markdown
---
title: "{文章标题}"
source: "{原始链接}"
date: {YYYY-MM-DD}
tags: [{标签1}, {标签2}, {标签3}]
---

## 核心摘要

1. {第一句摘要}
2. {第二句摘要}
3. {第三句摘要}

---

## 正文

{清洗后的 Markdown 正文内容}
```

- **文件内容格式**（Twitter 推文）:

```markdown
---
title: "推文 - {作者} - {日期}"
source: "{原始链接}"
date: {YYYY-MM-DD}
tags: [#Twitter, {标签2}, {标签3}]
---

## 作者

{作者用户名 / 显示名称}

## 核心观点

1. {第一句摘要}
2. {第二句摘要}
3. {第三句摘要}

---

## 原文

{清洗后的 Markdown 原文内容}
```

#### GitHub API 调用

使用 GitHub REST API 创建/更新文件：

```bash
# 获取用户信息（先确定 GitHub 用户名）
curl -s -H "Authorization: token {GITHUB_TOKEN}" https://api.github.com/user

# 检查文件是否存在
curl -s -H "Authorization: token {GITHUB_TOKEN}" \
  https://api.github.com/repos/{USERNAME}/Hermes-Inbox/contents/articles/{filename}

# 创建或更新文件
# 需要先获取文件的 sha（如果存在）
# PUT /repos/{USERNAME}/Hermes-Inbox/contents/articles/{filename}
```

### 步骤 5：回复用户

处理完成后，回复用户处理结果，包括：
- 文章标题 / 推文作者
- 核心摘要（3句）
- 标签
- 类型标识（文章/推文）
- GitHub 仓库中的保存路径

## 注意事项

### GitHub 认证
- 优先使用环境变量 `GITHUB_TOKEN` 或 `GH_TOKEN`
- 也可使用 `gh` CLI：`gh api ...`
- 如果未配置，回复告知用户：GitHub Token 尚未配置，技能无法写入仓库。
- 绝对不要向用户索取 Token、密码或任何凭证。

### Token 权限要求
- Token 需要 `repo` 权限（访问私有仓库）

### 错误处理
- 如果 Jina API 返回错误，重试 1 次
- 如果 Jina 返回 "Max challenge attempts exceeded" 类 WAF 拦截信息：
- **优先策略**：尝试用浏览器工具（browser_navigate）加载页面，检查是否有 PDF 下载链接
- **通用 PDF 解析模式**：通用 PDF 解析模式：如果检测到目标链接直接指向一个 PDF 文件，或者页面内包含有效的 PDF 文档直链，优先执行文件下载，并在后台自动调用 pdfminer.six 工具库（通过 pip install pdfminer.six）直接无噪提取 PDF 文本。
- **路径路由优化**：如果检测到特定多语言区域域名（如带有 en-JP 等后缀）容易触发拦截，优先尝试将其替换并重定向为标准通用英文域名（如包含 /en/ 的通用路径）后再执行抓取。
- **兜底策略**：如果无头浏览器工具也返回由于高级安全防护拦截导致的空页面或错误，则判定该网址为不可抓取状态，停止无限重试。
- 如果 GitHub API 因未授权失败，回复告知用户 Token 未配置
- 如果链接无法访问，回复用户告知

### 多个链接
如果用户一次发送多个链接，逐个处理并汇总结果。

## 首次使用设置

如果用户尚未配置 GitHub Token，在第一次触发时提示：

请自行在终端配置环境变量 `GITHUB_TOKEN` 或 `GH_TOKEN`（需要 `repo` 权限），或写入 `~/.hermes/.env` 文件。配置完成后可通过 `echo $GITHUB_TOKEN` 验证。
