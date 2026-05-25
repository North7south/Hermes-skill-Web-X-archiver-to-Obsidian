---
name: article_cleaner_and_saver
description: Automatically fetches, cleans, summarizes web articles or tweets and saves them to the private GitHub repository Hermes-Inbox. Triggered when the user sends a link starting with http/https. Twitter/x.com links are automatically replaced with the fixupx.com anti-scraping proxy.
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

Automatically processes web or tweet links sent by the user — extracts content, generates a summary, and saves it to the private GitHub repository Hermes-Inbox. Supports Twitter/x.com anti-scraping proxy.

## Trigger Condition

Automatically triggered when the user sends or forwards a URL starting with `http://` or `https://`.

## URL Pre-processing (Anti-scraping Bypass)

Before calling the Jina API, inspect the URL domain:

### Twitter/x.com Link Pre-processing

When the URL domain is `twitter.com` or `x.com`:

1. **Detect**: Check whether the URL contains `twitter.com` or `x.com`
2. **Replace**: Substitute the domain — `twitter.com` → `fixupx.com`, `x.com` → `fixupx.com`
3. **Compose**: Final request URL = `https://r.jina.ai/https://fixupx.com/...`

**Examples**:
- Original: `https://x.com/user/status/123`
- Processed: `https://r.jina.ai/https://fixupx.com/user/status/123`

- Original: `https://twitter.com/user/status/456`
- Processed: `https://r.jina.ai/https://fixupx.com/user/status/456`

### Regular Links

For non-Twitter/x.com links, use the original URL directly:
`https://r.jina.ai/{original URL}`

## Workflow

### Step 1: Extract Links

Extract all http/https URLs from the user's message. Multiple links can be processed in a single run.

### Step 2: URL Pre-processing

For each link, check the domain:
- **Twitter/x.com links** → Replace with fixupx.com, then prepend the Jina prefix
- **Regular links** → Prepend the Jina prefix directly

Invoke with:
```bash
curl -s "https://r.jina.ai/{processed URL}"
```

The Jina API returns the cleaned article body in plain Markdown format.

### Step 3A: Regular Article Processing

Read the fetched Markdown body and for each article:

1. **Extract the title** — Identify the article title from the content
2. **Distill a 3-sentence core summary** — Concisely capture the key points in English
3. **Section-level summary (long / structured articles)** — For long articles with multiple sections, add 1–2 sentence takeaways per major section after the core summary, preserving the original structure
4. **Generate 2–3 tags** — In hashtag format, e.g. `#AI #Crypto #Finance`

### Step 3B: Twitter Tweet Processing

When the link is a Twitter/x.com type, extract from the cleaned content:

1. **Extract author** — The tweet publisher's username (e.g. @username) and display name
2. **Distill core insights** — Concisely summarize the tweet content in English, 3-sentence summary
3. **Tags** — Always include `#Twitter`, plus 1–2 additional tags based on content

### Step 4: Save to GitHub Repository

Call the GitHub API to write the content to the `Hermes-Inbox` repository.

#### Repository Info
- **Repo name**: `Hermes-Inbox` (private repository)
- **File path format**: `articles/{YYYY-MM-DD}-{slug}.md`
  - The slug is auto-generated from the title (lowercase, spaces replaced with hyphens, special characters removed)
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

- **File content format** (Twitter tweet):

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
  https://api.github.com/repos/{USERNAME}/Hermes-Inbox/contents/articles/{filename}

# Create or update the file
# Retrieve the file's sha first if it already exists
# PUT /repos/{USERNAME}/Hermes-Inbox/contents/articles/{filename}
```

### Step 5: Reply to the User

After processing, report back to the user with:
- Article title / Tweet author
- Core summary (3 sentences)
- Tags
- Content type (article / tweet)
- Saved path in the GitHub repository

## Notes

### GitHub Authentication
- Prefer environment variables `GITHUB_TOKEN` or `GH_TOKEN`
- The `gh` CLI is also supported: `gh api ...`
- If not configured, inform the user: "GitHub Token is not configured — the skill cannot write to the repository."
- Never ask the user for a token, password, or any credentials.

### Token Permission Requirements
- The token must have `repo` scope (to access private repositories)

### Error Handling
- If the Jina API returns an error, retry once
- If Jina returns a WAF block such as "Max challenge attempts exceeded":
  - **Primary strategy**: Try loading the page with the browser tool (`browser_navigate`) and check for a PDF download link
  - **Generic PDF parsing mode**: If the target URL points directly to a PDF file, or the page contains a valid direct PDF link, download the file and silently invoke `pdfminer.six` (via `pip install pdfminer.six`) to extract the text cleanly
  - **Path routing optimization**: If a locale-specific domain (e.g. one with an `en-JP` suffix) frequently triggers blocks, try redirecting to the standard English domain (e.g. a path containing `/en/`) before retrying
  - **Fallback strategy**: If the headless browser also returns an empty page or error due to advanced security protection, mark the URL as unscrapable and stop retrying
- If the GitHub API fails with an authorization error, inform the user that the token is not configured
- If a link is unreachable, notify the user

### Multiple Links
If the user sends multiple links at once, process them one by one and return a consolidated summary.

## First-time Setup

If the user has not yet configured a GitHub Token, prompt them on first trigger:

Please configure the environment variable `GITHUB_TOKEN` or `GH_TOKEN` in your terminal (requires `repo` scope), or write it to `~/.hermes/.env`. You can verify the setup with `echo $GITHUB_TOKEN`.
