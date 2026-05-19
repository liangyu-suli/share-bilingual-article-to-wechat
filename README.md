# /share-bilingual-article-to-wechat

A Claude Code skill that translates English blog posts into Simplified Chinese and pushes bilingual drafts directly to WeChat Official Account.

## Install

Add this skill to any Claude Code project:

```bash
# Clone into your project's skills directory
git clone https://github.com/liangyu-suli/share-bilingual-article-to-wechat .claude/commands/share-bilingual-article-to-wechat

# Or copy just the skill file
curl -o .claude/commands/share-bilingual-article-to-wechat.md \
  https://raw.githubusercontent.com/liangyu-suli/share-bilingual-article-to-wechat/main/.claude/commands/share-bilingual-article-to-wechat.md
```

**For AI agents** — add to your agent's system prompt or tool list:

```
Skill: /share-bilingual-article-to-wechat
Source: https://raw.githubusercontent.com/liangyu-suli/share-bilingual-article-to-wechat/main/.claude/commands/share-bilingual-article-to-wechat.md
Invoke: /share-bilingual-article-to-wechat <url-or-markdown-path>
```

## Prerequisites

- [Claude Code](https://claude.ai/code) CLI
- [md2wechat](https://github.com/geekjourneyx/md2wechat-skill) — WeChat draft publisher
- [@foolgry/wxmd-cli](https://github.com/foolgry/editor) — HTML theme renderer (`npm install -g @foolgry/wxmd-cli`, then clone the repo to `/tmp/foolgry-editor`)
- Python 3 with Pillow (`pip install Pillow`)
- A WeChat Official Account (服务号 or 订阅号)

## Setup

1. Copy `.env.example` to `.env` and fill in your credentials:
   ```
   WECHAT_APPID=your_appid_here
   WECHAT_SECRET=your_appsecret_here
   ```
2. Add your machine's outbound IP to the WeChat IP whitelist:
   - WeChat Console → Settings → Development → Basic Configuration → IP Whitelist
   - Check your current IP: `curl -s https://api.ipify.org`
   - WeChat requires a **static outbound IP**. On home networks with rotating IPs you'll need to refresh the whitelist whenever the address changes — or run the skill behind a fixed-IP VPN / cloud relay.

The skill runs a preflight check on every invocation and halts with a clear message if `.env` is missing, credentials are placeholders, or a dependency isn't installed.

## Usage

```
/share-bilingual-article-to-wechat https://example.com/some-article
/share-bilingual-article-to-wechat path/to/article.md
```

The skill will ask you to pick an HTML theme from 20 options before generating output.

## What it does

1. Fetches the full article verbatim (no paraphrasing) — strips comments, links, footers at the source
2. Translates to Simplified Chinese through a 6-stage review pipeline
3. Asks you to choose a WeChat HTML theme (20 styles available)
4. Saves five files under `<slug>/`:

| File | Description |
|------|-------------|
| `<slug>.md` | Original English source |
| `<slug>.zh.md` | Simplified Chinese translation |
| `<slug>.bilingual.md` | Interleaved EN + ZH (paragraph by paragraph) |
| `<slug>.wechat.html` | WeChat HTML — Chinese only |
| `<slug>.bilingual.wechat.html` | WeChat HTML — bilingual |

5. Pushes the bilingual article as a draft to your WeChat Official Account

## Translation pipeline

Preflight → Fetch → Pre-processing → Translation → Correctness Review → Fluency Review → Style Refinement → Confidence Scoring → Deliver → WeChat Push

- Preserves original text verbatim — no paraphrasing
- Human names are never transliterated
- Honors the original voice and register
- Halts for human input when confidence < 0.75 or decisions need to be made
- Maintains `glossary.json` of approved term pairs across runs

## Credits

- WeChat draft publishing: [md2wechat](https://github.com/geekjourneyx/md2wechat-skill)
- WeChat HTML themes: [@foolgry/wxmd-cli](https://github.com/foolgry/editor)

## Translated articles

See [SOURCES.md](SOURCES.md) for the index of translated articles and output files.
