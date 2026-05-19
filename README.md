# /share-bilingual-article-to-wechat

An AI agent skill that translates English blog posts into Simplified Chinese and pushes bilingual drafts directly to WeChat Official Account.

## Install

Feed the skill definition to your agent:

```bash
# Download the skill file
curl -o share-bilingual-article-to-wechat.md \
  https://raw.githubusercontent.com/liangyu-suli/share-bilingual-article-to-wechat/main/.claude/commands/share-bilingual-article-to-wechat.md
```

Then load it into your agent however it accepts skill/instruction files — as a system prompt addition, a tool definition, or a commands directory entry.

**Invoke:**
```
/share-bilingual-article-to-wechat <url>
/share-bilingual-article-to-wechat <path-to-markdown-file>
```

## Prerequisites

- [md2wechat](https://github.com/geekjourneyx/md2wechat-skill) — WeChat draft publisher
- [@foolgry/wxmd-cli](https://github.com/foolgry/editor) — HTML theme renderer
  ```bash
  npm install -g @foolgry/wxmd-cli
  git clone --depth 1 https://github.com/foolgry/editor /tmp/foolgry-editor
  ```
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
   - WeChat requires a **static outbound IP**. On home networks with rotating IPs you'll need to refresh the whitelist whenever the address changes — or run behind a fixed-IP VPN / cloud relay.

The skill runs a preflight check on every invocation and halts with a clear message if `.env` is missing, credentials are placeholders, or a dependency isn't installed.

## What it does

1. Fetches the full article verbatim (no paraphrasing) — strips comments, links, and footers at the source
2. Translates to Simplified Chinese through a 6-stage review pipeline
3. Asks you to pick a WeChat HTML theme (20 styles available)
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
