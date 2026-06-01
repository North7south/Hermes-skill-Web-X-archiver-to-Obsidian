---
name: web_x_archiver_to_obsidian
description: 一个 Hermes Agent 技能。用户在 Telegram 给 Hermes Bot 发送任意网页或 Twitter/X 链接，Hermes 自动抓取、清洗、生成摘要，并通过 GitHub API 写入用户自定义的私有仓库。该仓库通过 Obsidian Git 插件按用户配置的间隔同步到本地 Obsidian 知识库。Twitter/x.com 链接自动替换为 fixupx.com 以绕过反爬。
version: 2.2.0
author: North7south
license: MIT
metadata:
  hermes:
    tags: [hermes, obsidian, telegram, article, cleaner, github, web-scraping, summary, twitter, knowledge-base]
    related_skills: []
trigger:
  match: https?://\S+
  match_type: regex
---

# Hermes 技能：Web & X 归档器 → Obsidian

一个 Hermes 原生自动化技能。

**工作链路：**

```
用户 → Telegram → Hermes Bot → r.jina.ai 清洗
                                    ↓
                              生成摘要 + 标签
                                    ↓
                              GitHub API 写入
                                    ↓
                        用户自定义私有仓库（如 my-inbox）
                                    ↓
                        Obsidian Git 插件按间隔自动拉取
                                    ↓
                              Obsidian 知识库
```

从"发链接"到"知识库"全程零复制粘贴。

## 触发条件

用户在 Telegram（或其他 Hermes 连接的平台）发送或转发任意 `http://` / `https://` 链接时自动触发。

## URL 预处理（反爬绕过）

在调用 Jina API 之前，先判断 URL 域名：

### Twitter/X 链接预处理

当 URL 域名是 `twitter.com` 或 `x.com` 时：

1. **检测**：检查 URL 中是否包含 `twitter.com` 或 `x.com`
2. **替换**：将域名中的 `twitter.com` → `fixupx.com`，`x.com` → `fixupx.com`
3. **拼接**：最终请求 URL = `https://r.jina.ai/https://fixupx.com/...`

**示例**：
- 原始: `https://x.com/user/status/123`
- 处理后: `https://r.jina.ai/https://fixupx.com/user/status/123`

### 普通链接

非 Twitter/X 链接直接使用原始 URL：
`https://r.jina.ai/{原始链接}`

## 工作流程

### 步骤 1：接收并解析消息

从 Telegram 用户消息正文中提取所有 `http://` / `https://` 开头的 URL。

**失败隔离原则**：当一条消息含多个链接时，**逐个独立处理**，单个链接失败不阻断其他链接的处理。最终回复中明确标注每条链接的成功/失败状态。

### 步骤 2：URL 预处理

对每个链接判断域名：
- **Twitter/X 链接** → 替换为 fixupx.com 后再拼接 Jina
- **普通链接** → 直接拼接 Jina

调用命令：
```bash
curl -s "https://r.jina.ai/{处理后的URL}"
```

Jina 接口返回纯 Markdown 格式的清洗后正文内容。

#### Twitter 抓取回退策略

`fixupx.com` 偶有抓取失败的情况。**触发回退的明确条件**：
- 返回内容**少于 50 字**，或
- 返回内容**仅含登录提示**（如 "Sign in to X" / "Log in to Twitter" 等关键词）

满足以上任一条件时，按顺序尝试：
1. 改用 `fxtwitter.com` 作为代理重试一次
2. 仍失败则用 `browser_navigate` 加载原始页面
3. 全部失败则将该链接标记为不可抓取，继续处理其他链接

> ⚠️ 内容**长度大于 50 字且非登录页**时不触发回退，避免误判短推文。

### 步骤 3A：普通文章处理

阅读抓取到的 Markdown 正文，为每篇文章：

1. **提取标题** — 从内容中识别文章标题
2. **核心摘要（所有文章必做）** — 用简洁中文提炼 3 句核心要点
3. **正文摘要（仅长文/结构性文章）** — 仅当原文含明显的章节划分（多个 H2/H3 标题）时执行。在核心摘要之后，为每个主要章节追加 1-2 句要点，保留原文结构逻辑。短文跳过此步。
4. **归纳 2-3 个标签** — 用 # 号标签形式，如 `#AI #生产力 #研究`

### 步骤 3B：Twitter/X 推文处理

当链接为 Twitter/X 类型时，从清洗后的内容中：

1. **提取作者** — 推文发布者用户名（如 @username）和显示名称
2. **提炼核心观点** — 用简洁中文概括推文内容，3 句摘要
3. **标签** — 自动包含 `#Twitter` 标签，再根据内容归纳 1-2 个附加标签

### 步骤 4：写入用户配置的 GitHub 仓库

调用 GitHub API 将内容写入用户在配置中指定的私有仓库（默认值 `Hermes-Inbox`，用户可在 `~/.hermes/.env` 中通过 `GITHUB_REPO` 变量自定义）。

#### 仓库信息（用户可配置）

| 变量 | 说明 | 默认值 |
|---|---|---|
| `GITHUB_REPO` | 目标仓库名 | `Hermes-Inbox` |
| `GITHUB_FOLDER` | 仓库内子目录 | `articles` |
| `GITHUB_BRANCH` | 目标分支 | `main` |

- **文件路径格式**：`{GITHUB_FOLDER}/{YYYY-MM-DD}-{slug}.md`
  - slug 由标题自动生成（小写、连字符替换空格、去除特殊字符）

- **文件内容格式**（普通文章）：

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

- **文件内容格式**（Twitter/X 推文）：

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
  https://api.github.com/repos/{USERNAME}/{GITHUB_REPO}/contents/{GITHUB_FOLDER}/{filename}

# 创建或更新文件
# 需要先获取文件的 sha（如果存在）
# PUT /repos/{USERNAME}/{GITHUB_REPO}/contents/{GITHUB_FOLDER}/{filename}
```

### 步骤 5：Obsidian 端自动同步

本技能本身**只负责写入 GitHub**。从 GitHub 同步到 Obsidian 由 Obsidian Git 插件按用户在 Obsidian 设置的间隔（如随时 / 1 分钟 / 10 分钟）自动完成，与 Hermes 解耦。

### 步骤 6：回复用户

处理完成后，在 Telegram 回复用户处理结果。**对每条链接独立汇报**，明确标注成功或失败：

- ✅ **成功**：文章标题 / 推文作者、核心摘要（3 句）、标签、类型、GitHub 保存路径
- ❌ **失败**：原始链接 + 失败原因（如 "WAF 拦截"、"内容少于 50 字"、"链接无法访问"）

最后提示：成功的文件会在下一次 Obsidian Git 拉取后出现在知识库中。

---

## 首次使用设置

### A. GitHub Token 配置（必需）

#### 1. 创建 GitHub Personal Access Token

1. 访问 https://github.com/settings/tokens
2. 点击 **Generate new token (classic)**
3. **Note**: 填写 `Hermes Skill`（便于识别）
4. **Expiration**: 选择有效期（建议 90 天或自定义）
5. **Scopes**: 勾选 `repo`（完整仓库读写权限，含私有仓库）
6. 点击 **Generate token**，复制生成的 `ghp_...` 字符串（**只显示一次**）

#### 2. 创建私有仓库

1. 访问 https://github.com/new
2. **Repository name**: 例如 `Hermes-Inbox`（或自定义名称）
3. **Visibility**: 选 **Private**
4. 勾选 **Add a README file**
5. 点击 **Create repository**

#### 3. 写入 Hermes 环境变量

在 Hermes 主机上执行：

```bash
# 创建或编辑 .env 文件
cat >> ~/.hermes/.env << 'EOF'
GITHUB_TOKEN=ghp_你的token粘贴在这里
GITHUB_REPO=Hermes-Inbox
GITHUB_FOLDER=articles
GITHUB_BRANCH=main
EOF

# 验证
source ~/.hermes/.env
echo $GITHUB_TOKEN   # 应输出 ghp_...
echo $GITHUB_REPO    # 应输出 Hermes-Inbox
```

#### 4. 验证 Token 可用

```bash
curl -s -H "Authorization: token $GITHUB_TOKEN" https://api.github.com/user | grep '"login"'
# 应返回类似：  "login": "你的GitHub用户名",
```

---

### B. Obsidian 端配置（必需，用于自动同步）

#### 1. 安装 Obsidian Git 插件

1. 打开 Obsidian → **设置（Settings）**
2. 进入 **第三方插件（Community plugins）** → 关闭 **限制模式（Restricted mode）**
3. 点击 **浏览（Browse）**，搜索 `Obsidian Git`
4. 点击 **安装（Install）** → **启用（Enable）**

#### 2. 把 GitHub 仓库克隆到 Vault 内

打开终端，进入你的 Obsidian Vault 根目录：

```bash
cd ~/Documents/你的Vault名称

# 克隆仓库到 Vault 内子目录（推荐）
git clone https://github.com/你的用户名/Hermes-Inbox.git
```

克隆后 Vault 里会出现一个 `Hermes-Inbox/` 子目录，里面所有 .md 文件都会被 Obsidian 识别为笔记。

#### 3. 配置 Git 用户身份（首次使用 Git 必做）

```bash
git config --global user.name "你的用户名"
git config --global user.email "你的邮箱@example.com"
```

#### 4. 配置 Git 凭据（让插件能拉取私有仓库）

推荐用 GitHub CLI 或 SSH：

```bash
# 方式一：GitHub CLI（最简单）
brew install gh   # 或参考 https://cli.github.com
gh auth login
```

或用 HTTPS + Token：

```bash
git config --global credential.helper store
# 下一次 git pull 时输入用户名和 Token（不是密码）即可
```

#### 5. 在 Obsidian Git 插件设置中配置自动拉取

打开 Obsidian → **设置** → **Obsidian Git**：

| 选项 | 推荐值 | 说明 |
|---|---|---|
| **Vault Backup Interval (minutes)** | `0` | 关闭自动 commit（我们只 pull，不 push）|
| **Auto pull interval (minutes)** | `1` 或 `10` | 自动拉取间隔，可按需调整 |
| **Pull updates on startup** | ✅ 开启 | 打开 Obsidian 时立即拉一次 |

> ⚠️ 注意：如果你克隆的是 Vault 内的子目录，需要在插件的 **Custom base path** 里填写子目录路径（如 `Hermes-Inbox`），否则插件会在 Vault 根目录找 .git 文件夹。

#### 6. 验证

在 Telegram 给 Hermes Bot 发一条测试链接，等待 1-10 分钟（取决于你设置的间隔），笔记应自动出现在 Obsidian 的 `Hermes-Inbox/articles/` 目录下。

---

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
  - **通用 PDF 解析模式**：如果检测到目标链接直接指向一个 PDF 文件，或者页面内包含有效的 PDF 文档直链，优先执行文件下载，并在后台自动调用 pdfminer.six 工具库（通过 pip install pdfminer.six）直接无噪提取 PDF 文本。
  - **路径路由优化**：如果检测到特定多语言区域域名（如带有 en-JP 等后缀）容易触发拦截，优先尝试将其替换并重定向为标准通用英文域名（如包含 /en/ 的通用路径）后再执行抓取。
  - **兜底策略**：如果无头浏览器工具也返回由于高级安全防护拦截导致的空页面或错误，则判定该网址为不可抓取状态，停止无限重试。
- 如果 GitHub API 因未授权失败，回复告知用户 Token 未配置
- 如果链接无法访问，回复用户告知

### 多个链接的失败隔离
- 用户一次发送多个链接时，**逐个独立处理**
- 任何单个链接的失败（抓取失败、写入失败、WAF 拦截等）**不影响其他链接**
- 最终回复中**每条链接独立汇报**成功 / 失败状态，方便用户排查
