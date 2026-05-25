
# Hermes Extension Skill: article_cleaner_and_saver

这是一个基于纯自然语言指令（Prompt-based Skill）训练的 Hermes 自动化网关扩展规则。它通过简单的逻辑声明，使 Bot 具备了通用网页与 Twitter 链路的自动化清洗及归档功能。

## 规则特性
* 多媒介兼容：自动识别 http/https 通用网页，以及 x.com / twitter.com 链接。
* 代理级突破：强制在底层对 Twitter 链接执行 fixupx.com 转义，绕过反爬抓取纯文本。
* 知识内化：调用 r.jina.ai 洗净正文，利用大模型自动内化为 3 句核心摘要与高价值标签。
* 安全交付：自动调用 GitHub API 将结构化 Markdown 写入指定知识库。
