# Translation Workflow

A Claude Code skill that translates English blog posts into Simplified Chinese and pushes bilingual drafts directly to WeChat Official Account.

## Prerequisites

- [Claude Code](https://claude.ai/code) CLI installed
- [md2wechat](https://github.com/geekjourneyx/md2wechat-skill) installed (`go install` or download binary)
- Python 3 with `Pillow` (`pip install Pillow`)
- A WeChat Official Account (服务号 or 订阅号)

## Setup

1. Clone this repo and open it in Claude Code
2. Copy `.env.example` to `.env` and fill in your credentials:
   ```
   WECHAT_APPID=your_appid_here
   WECHAT_SECRET=your_appsecret_here
   ```
3. Add your machine's outbound IP to the WeChat IP whitelist:
   - WeChat Console → Settings → Development → Basic Configuration → IP Whitelist
   - Check your current IP: `curl -s https://api.ipify.org`
   - WeChat requires a **static outbound IP**. On home networks with rotating
     IPs you'll need to refresh the whitelist whenever the address changes —
     or run the skill behind a fixed-IP VPN / cloud relay.

The skill runs a preflight check on every invocation and halts with a clear
message if `.env` is missing, the credentials are still placeholders, or
`md2wechat` / Pillow isn't installed — so a misconfigured run won't waste a
translation pass.

## Usage

In Claude Code, run the skill with a URL or a local Markdown file:

```
/share-bilingual-article-to-wechat https://example.com/some-article
/share-bilingual-article-to-wechat path/to/article.md
```

## What it does

1. Fetches the full article content verbatim (no paraphrasing)
2. Translates to Simplified Chinese through a 6-stage review pipeline
3. Saves the source plus four derived files under `<slug>/`:

| File | Description |
|------|-------------|
| `<slug>.md` | Original English source |
| `<slug>.zh.md` | Simplified Chinese translation |
| `<slug>.bilingual.md` | Interleaved EN + ZH (paragraph by paragraph) |
| `<slug>.wechat.html` | WeChat-ready HTML (Chinese only) |
| `<slug>.bilingual.wechat.html` | WeChat-ready HTML (bilingual) |

4. Pushes the bilingual article as a draft to your WeChat Official Account

## Translation pipeline

Pre-processing → Translation → Correctness Review → Fluency Review → Style Refinement → Confidence Scoring → Delivery → WeChat Push

- Preserves original text verbatim — no paraphrasing
- Human names are never transliterated
- Honors the original voice and register
- Halts for human input when confidence < 0.75 or decisions need to be made
- Maintains `glossary.json` of approved term pairs across runs

## Credits

WeChat HTML conversion and draft publishing powered by [md2wechat](https://github.com/geekjourneyx/md2wechat-skill).

## Translated articles

See [SOURCES.md](SOURCES.md) for the index of translated articles and their output files.
