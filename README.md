# Hermes Skill: Web & X Archiver → Obsidian

A natural-language **Hermes Agent skill** that turns any web URL or Twitter/X link you send to your Hermes Bot on Telegram into a clean, summarized, tagged Markdown note — committed to a private GitHub repo and auto-synced into your **Obsidian vault**.

> 📥 Send a link on Telegram → 🧹 Hermes cleans it → ✍️ Summarizes it → 📦 Pushes to GitHub → 📓 Obsidian Git plugin pulls it into your vault.

---

## ✨ Features

- 🌐 **Universal URL support** — Handles any `http(s)://` web page
- 🐦 **Twitter/X anti-scraping bypass** — Auto-rewrites `x.com` / `twitter.com` to `fixupx.com`, with `fxtwitter.com` + headless-browser fallback. No Twitter API key needed.
- 🧠 **Knowledge internalization** — Uses `r.jina.ai` to clean the body, generates a 3-sentence summary and 2–3 semantic tags. Long articles also get per-section takeaways.
- 📦 **User-configurable GitHub delivery** — Write to any private repo you specify via `GITHUB_REPO`
- 📓 **Obsidian-ready** — Pair with the Obsidian Git plugin to auto-pull at your chosen interval (every commit / 1 min / 10 min). Zero manual copy-paste.
- 🛡️ **Robust failure isolation** — Multiple links in one message? Each processed independently, with clear per-link success/failure reporting.
- 🔒 **PDF & WAF fallback strategies** — PDF auto-detection (pdfminer.six), locale-domain rewriting, graceful give-up on hard blocks.

---

## 🏗️ Architecture

```
   You send a URL on Telegram
              │
              ▼
       ┌──────────┐
       │  Hermes  │  ← runs this skill
       └────┬─────┘
            │
            ▼
       ┌──────────┐
       │ r.jina.ai│  ← cleans content
       └────┬─────┘
            │ (fixupx.com proxy for X links,
            │  fxtwitter.com fallback if needed)
            ▼
       ┌──────────┐
       │  GitHub  │  ← your private repo
       └────┬─────┘
            │
            ▼
       ┌──────────┐
       │ Obsidian │  ← Git plugin auto-pulls
       └──────────┘
```

---

## 🚀 Quick Start

### Prerequisites

- A running [Hermes Agent](https://hermes-agent.nousresearch.com) connected to a Telegram bot
- A private GitHub repository (name it whatever you like)
- A GitHub Personal Access Token with `repo` scope
- [Obsidian](https://obsidian.md) + [Obsidian Git plugin](https://github.com/Vinzent03/obsidian-git)

### Install the skill

```bash
cd ~/.hermes/skills/
git clone https://github.com/North7south/Hermes-skill-Web-X-archiver.git
```

Hermes will auto-discover the skill on next start.

### Configure

```bash
cat >> ~/.hermes/.env << 'EOF'
GITHUB_TOKEN=ghp_your_token_here
GITHUB_REPO=Hermes-Inbox
GITHUB_FOLDER=articles
GITHUB_BRANCH=main
EOF
```

Verify:
```bash
source ~/.hermes/.env && echo $GITHUB_TOKEN
```

### Wire Obsidian to GitHub

In Obsidian:
1. Install **Obsidian Git** plugin
2. Clone your repo into your vault: `git clone https://github.com/your-username/Hermes-Inbox.git`
3. In plugin settings: set **Auto pull interval** (e.g. 1 minute), keep **Vault Backup Interval = 0**

Detailed step-by-step setup (Token creation, repo creation, Git credentials, plugin config) is in [SKILL-EN.md](./SKILL-EN.md#first-time-setup).

---

## 📖 Usage

Just send a link to your Hermes Bot on Telegram:

```
https://nousresearch.com/hermes-4/
```

Or a tweet:

```
https://x.com/NousResearch/status/1234567890
```

Hermes will reply with:
- 📌 Title / author
- 🧾 3-sentence summary
- 🏷️ Tags
- 🔗 Saved path

The note arrives in Obsidian after the next Git pull.

Multi-link messages work too — each link is processed independently, and you'll get a per-link success/failure summary.

---

## 📂 Output Format

**Regular articles** → `{GITHUB_FOLDER}/{YYYY-MM-DD}-{slug}.md`

```markdown
---
title: "Article Title"
source: "https://example.com/post"
date: 2026-05-31
tags: [#AI, #Research]
---

## Summary
1. ...
2. ...
3. ...

---

## Content
{cleaned Markdown body}
```

**Tweets** → `{GITHUB_FOLDER}/{YYYY-MM-DD}-tweet-{author}.md`

```markdown
---
title: "Tweet - @username - 2026-05-31"
source: "https://x.com/user/status/123"
date: 2026-05-31
tags: [#Twitter, #AI]
---

## Author
@username (Display Name)

## Key Insights
1. ...
2. ...
3. ...

---

## Original Tweet
{cleaned content}
```

---

## 📑 Skill Documents

- 🇬🇧 [SKILL-EN.md](./SKILL-EN.md) — English skill definition (primary)
- 🇨🇳 [SKILL-CN.md](./SKILL-CN.md) — Chinese skill definition (mirror)

---

## 🤝 Related

- [Hermes Agent](https://hermes-agent.nousresearch.com) — The self-improving AI agent by Nous Research
- [Obsidian](https://obsidian.md) — Local-first knowledge base
- [Obsidian Git plugin](https://github.com/Vinzent03/obsidian-git) — The bridge that closes the loop

---

## 📜 License

MIT © [North7south](https://github.com/North7south)
