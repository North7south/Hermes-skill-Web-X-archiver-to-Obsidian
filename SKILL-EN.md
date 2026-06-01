---
name: web_x_archiver_to_obsidian
description: A Hermes Agent skill. When a user sends any web page or Twitter/X link to the Hermes Bot on Telegram, Hermes auto-fetches, cleans, summarizes the content, and writes it via the GitHub API to a user-configured private repository. That repo is synced into the user's local Obsidian vault via the Obsidian Git plugin at a user-defined interval. Twitter/x.com links are automatically rewritten to fixupx.com to bypass anti-scraping.
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

# Hermes Skill: Web & X Archiver → Obsidian

A Hermes-native automation.

**Pipeline:**

```
User → Telegram → Hermes Bot → r.jina.ai (clean)
                                    ↓
                          Generate summary + tags
                                    ↓
                            GitHub API (write)
                                    ↓
                  User's custom private repo (e.g. my-inbox)
                                    ↓
                  Obsidian Git plugin auto-pulls on schedule
                                    ↓
                              Obsidian Vault
```

From "send a link" to "knowledge base" — zero manual copy-paste.

## Trigger Condition

Automatically triggered when the user sends or forwards any `http://` / `https://` URL on Telegram (or any other platform connected to Hermes).

## URL Pre-processing (Anti-scraping Bypass)

Before calling the Jina API, inspect the URL domain:

### Twitter/X Link Pre-processing

When the URL domain is `twitter.com` or `x.com`:

1. **Detect**: Check whether the URL contains `twitter.com` or `x.com`
2. **Replace**: Substitute the domain — `twitter.com` → `fixupx.com`, `x.com` → `fixupx.com`
3. **Compose**: Final request URL = `https://r.jina.ai/https://fixupx.com/...`

**Examples**:
- Original: `https://x.com/user/status/123`
- Processed: `https://r.jina.ai/https://fixupx.com/user/status/123`

### Regular Links

For non-Twitter/X links, use the original URL directly:
`https://r.jina.ai/{original URL}`

## Workflow

### Step 1: Receive and Parse the Message

Extract all `http://` / `https://` URLs from the user's Telegram message body.

**Failure isolation principle**: When a single message contains multiple links, **process each link independently**. A failure on one link must NOT block the others. The final reply must clearly mark the success/failure status of each link.

### Step 2: URL Pre-processing

For each link, check the domain:
- **Twitter/X links** → Rewrite to fixupx.com, then prepend the Jina prefix
- **Regular links** → Prepend the Jina prefix directly

Invoke with:
```bash
curl -s "https://r.jina.ai/{processed URL}"
```

The Jina API returns the cleaned article body in plain Markdown.

#### Twitter Fetch Fallback Strategy

`fixupx.com` occasionally fails. **Trigger fallback only when one of the following is true**:
- Returned content is **fewer than 50 characters**, OR
- Returned content contains **only a login prompt** (keywords like "Sign in to X" / "Log in to Twitter")

When triggered, try in order:
1. Retry once with `fxtwitter.com` as the proxy
2. If still failing, load the original page with `browser_navigate`
3. If all fail, mark the link as unscrapable and continue with the remaining links

> ⚠️ Do NOT trigger the fallback when content is longer than 50 chars and is not a login page — this avoids misclassifying short tweets as failures.

### Step 3A: Regular Article Processing

Read the fetched Markdown body. For each article:

1. **Extract the title** — Identify the article title from the content
2. **Core summary (always)** — Distill 3 sentences capturing the key insights
3. **Section-level summary (long / structured articles only)** — Execute ONLY when the article has clear section headers (multiple H2/H3). After the core summary, append 1–2 sentence takeaways per major section, preserving the original structure. Skip for short articles.
4. **Generate 2–3 tags** — In hashtag format, e.g. `#AI #Productivity #Research`

### Step 3B: Twitter/X Tweet Processing

When the link is a Twitter/X type, extract from the cleaned content:

1. **Extract author** — The publisher's username (e.g. @username) and display name
2. **Distill core insights** — Summarize the tweet concisely, 3-sentence summary
3. **Tags** — Always include `#Twitter`, plus 1–2 additional tags based on content

### Step 4: Write to User's Configured GitHub Repository

Call the GitHub API to write the content to the user-configured private repo (default `Hermes-Inbox`, customizable via `GITHUB_REPO` in `~/.hermes/.env`).

#### Repository Config (user-customizable)

| Variable | Description | Default |
|---|---|---|
| `GITHUB_REPO` | Target repo name | `Hermes-Inbox` |
| `GITHUB_FOLDER` | Folder inside repo | `articles` |
| `GITHUB_BRANCH` | Target branch | `main` |

- **File path format**: `{GITHUB_FOLDER}/{YYYY-MM-DD}-{slug}.md`
  - Slug is auto-generated from the title (lowercase, spaces replaced with hyphens, special characters removed)

- **File content format** (regular article):

```markdown
---
title: "{Article Title}"
source: "{original URL}"
date: {YYYY-MM-DD}
tags: [{tag1}, {tag2}, {tag3}]
---

## Summary

1. {First summary sentence}
2. {Second summary sentence}
3. {Third summary sentence}

---

## Content

{Cleaned Markdown body}
```

- **File content format** (Twitter/X tweet):

```markdown
---
title: "Tweet - {Author} - {Date}"
source: "{original URL}"
date: {YYYY-MM-DD}
tags: [#Twitter, {tag2}, {tag3}]
---

## Author

{@username / Display Name}

## Key Insights

1. {First summary sentence}
2. {Second summary sentence}
3. {Third summary sentence}

---

## Original Tweet

{Cleaned Markdown content}
```

#### GitHub API Calls

Use the GitHub REST API to create or update files:

```bash
# Get user info (to determine the GitHub username)
curl -s -H "Authorization: token {GITHUB_TOKEN}" https://api.github.com/user

# Check whether the file already exists
curl -s -H "Authorization: token {GITHUB_TOKEN}" \
  https://api.github.com/repos/{USERNAME}/{GITHUB_REPO}/contents/{GITHUB_FOLDER}/{filename}

# Create or update the file
# Retrieve the file's sha first if it already exists
# PUT /repos/{USERNAME}/{GITHUB_REPO}/contents/{GITHUB_FOLDER}/{filename}
```

### Step 5: Obsidian-side Auto Sync

This skill ONLY writes to GitHub. Syncing from GitHub to Obsidian is handled by the Obsidian Git plugin at a user-configured interval (every commit / 1 min / 10 min), fully decoupled from Hermes.

### Step 6: Reply to the User

After processing, reply on Telegram with **per-link status**:

- ✅ **Success**: Article title / tweet author, 3-sentence summary, tags, content type, GitHub save path
- ❌ **Failure**: Original URL + failure reason (e.g. "WAF blocked", "content < 50 chars", "URL unreachable")

End with a note: successful files will appear in the vault after the next Obsidian Git pull.

---

## First-time Setup

### A. GitHub Token Setup (required)

#### 1. Create a GitHub Personal Access Token

1. Visit https://github.com/settings/tokens
2. Click **Generate new token (classic)**
3. **Note**: `Hermes Skill` (for easy identification)
4. **Expiration**: choose a validity period (90 days or custom recommended)
5. **Scopes**: check `repo` (full repo read/write, including private)
6. Click **Generate token** and copy the `ghp_...` string (**shown only once**)

#### 2. Create a Private Repository

1. Visit https://github.com/new
2. **Repository name**: e.g. `Hermes-Inbox` (or your own name)
3. **Visibility**: select **Private**
4. Check **Add a README file**
5. Click **Create repository**

#### 3. Write the Hermes Environment Variables

On the Hermes host:

```bash
# Create or edit .env
cat >> ~/.hermes/.env << 'EOF'
GITHUB_TOKEN=ghp_paste_your_token_here
GITHUB_REPO=Hermes-Inbox
GITHUB_FOLDER=articles
GITHUB_BRANCH=main
EOF

# Verify
source ~/.hermes/.env
echo $GITHUB_TOKEN   # should print ghp_...
echo $GITHUB_REPO    # should print Hermes-Inbox
```

#### 4. Verify the Token Works

```bash
curl -s -H "Authorization: token $GITHUB_TOKEN" https://api.github.com/user | grep '"login"'
# Should return something like:   "login": "your-github-username",
```

---

### B. Obsidian-side Setup (required for auto-sync)

#### 1. Install the Obsidian Git Plugin

1. Open Obsidian → **Settings**
2. Go to **Community plugins** → turn off **Restricted mode**
3. Click **Browse**, search for `Obsidian Git`
4. Click **Install** → **Enable**

#### 2. Clone the GitHub Repo into Your Vault

In a terminal, go to your Obsidian vault root:

```bash
cd ~/Documents/YourVaultName

# Clone the repo as a sub-folder of the vault (recommended)
git clone https://github.com/your-username/Hermes-Inbox.git
```

Obsidian will pick up the new `Hermes-Inbox/` folder and treat every `.md` file inside as a note.

#### 3. Configure Git Identity (first-time Git users only)

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

#### 4. Configure Git Credentials (so the plugin can pull a private repo)

GitHub CLI or SSH is recommended:

```bash
# Option 1: GitHub CLI (easiest)
brew install gh   # or see https://cli.github.com
gh auth login
```

Or HTTPS + Token:

```bash
git config --global credential.helper store
# Next git pull will ask for username + token (not password)
```

#### 5. Configure Auto-pull in the Obsidian Git Plugin

Open Obsidian → **Settings** → **Obsidian Git**:

| Option | Recommended | Note |
|---|---|---|
| **Vault Backup Interval (minutes)** | `0` | Disable auto-commit — we only pull, never push |
| **Auto pull interval (minutes)** | `1` or `10` | Adjust to taste |
| **Pull updates on startup** | ✅ on | Pull immediately when Obsidian opens |

> ⚠️ If you cloned the repo as a sub-folder of the vault, fill in the plugin's **Custom base path** with the sub-folder name (e.g. `Hermes-Inbox`); otherwise the plugin will look for `.git` at the vault root.

#### 6. Verify

Send Hermes a test link on Telegram. Within 1–10 minutes (depending on your interval), the note should appear under `Hermes-Inbox/articles/` in Obsidian.

---

## Notes

### GitHub Authentication
- Prefer the environment variables `GITHUB_TOKEN` or `GH_TOKEN`
- The `gh` CLI is also supported: `gh api ...`
- If not configured, inform the user: "GitHub Token is not configured — the skill cannot write to the repository."
- Never ask the user for tokens, passwords, or any credentials.

### Token Permission Requirement
- The token needs `repo` scope (private repo read/write)

### Error Handling
- If the Jina API returns an error, retry once
- If Jina returns a WAF block such as "Max challenge attempts exceeded":
  - **Primary strategy**: Try loading the page with the browser tool (`browser_navigate`) and look for a PDF download link
  - **Generic PDF parsing**: If the target URL points directly to a PDF, or the page exposes a direct PDF link, download it and silently invoke `pdfminer.six` (via `pip install pdfminer.six`) for clean text extraction
  - **Path routing optimization**: If a locale-specific domain (e.g. one with an `en-JP` suffix) frequently triggers blocks, try redirecting to the standard English domain (path containing `/en/`) before retrying
  - **Fallback**: If the headless browser also returns an empty page or error due to advanced security protection, mark the URL as unscrapable and stop retrying
- If the GitHub API fails with an authorization error, inform the user that the token is not configured
- If a link is unreachable, notify the user

### Multi-link Failure Isolation
- When the user sends multiple links at once, **process each independently**
- A failure on any single link (fetch failure, write failure, WAF block, etc.) **must NOT affect the others**
- The final reply must report **per-link success/failure status** so the user can diagnose easily
