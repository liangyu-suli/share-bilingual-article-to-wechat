# share-bilingual-article-to-wechat

A self-contained pipeline that translates English blog posts into Simplified Chinese and pushes a bilingual draft to your WeChat Official Account — verbatim source, six-stage review, no paraphrasing.

Clone the repo, fill in your WeChat credentials, point any AI agent at the slash command, and an article goes from URL to draft in one pass.

## What this is (and isn't)

This is the assembled pipeline, not a portable prompt file. Cloning the repo gives you:

- The translation slash command (`.claude/commands/share-bilingual-article-to-wechat.md`)
- `glossary.json` — approved English↔Chinese term pairs that grow across runs
- An archive of past translations under named slug folders (e.g. `boil-the-ocean/`)
- `.env` scaffolding and the dependency manifest

The slash command depends on this environment — `.env`, the glossary, and the external tools `md2wechat` + `wxmd-cli`. Without it, the pipeline can't run. So install means clone, not curl.

Designed for [Claude Code](https://claude.com/claude-code) and tested there; runnable in any agent that can execute multi-step Markdown instructions from a project's commands directory.

## Pipeline

```
Preflight → Fetch → Pre-process → Translate → Correctness → Fluency → Style → Deliver → Push
```

- Preserves source text verbatim — no paraphrasing, no summarization
- Strips related links, comments, footers, share buttons, and CTAs at the source so they never enter the translation pipeline
- Keeps human names in Latin script — never transliterated
- Honors the original register (formal / conversational / technical)
- **Fluency review uses a Chinese-native LLM critic when `DEEPSEEK_API_KEY` is set** — DeepSeek reads the draft as a native Chinese reader and emits suggested edits; the agent applies them. Falls back to the agent's own native review when the key is absent.
- Halts for human input when the source is genuinely ambiguous
- Grows `glossary.json` with approved term pairs across articles

## Requirements

| Dependency | Why | Install |
|---|---|---|
| AI agent with Markdown command support | Executes the pipeline | Claude Code recommended |
| [md2wechat](https://github.com/geekjourneyx/md2wechat-skill) | Pushes drafts to the WeChat Open API | `go install` or download a release binary |
| [foolgry/editor](https://github.com/foolgry/editor) (wxmd-cli) | Renders WeChat HTML in 20 themes | `git clone --depth 1 https://github.com/foolgry/editor /tmp/foolgry-editor` |
| Python 3 + Pillow | Generates the cover image | `pip install Pillow` |
| Node.js | Runs wxmd-cli | `brew install node` (or your platform's equivalent) |
| `jq` | Builds the DeepSeek request payload (only required when `DEEPSEEK_API_KEY` is set) | `brew install jq` |
| **Optional:** [DeepSeek API key](https://platform.deepseek.com/) | Chinese-native LLM critic for the Stage 4 fluency review | Sign up, create a key, paste into `.env` as `DEEPSEEK_API_KEY` |
| A WeChat Official Account | Receives the draft | Service Account (服务号) or Subscription Account (订阅号) |

## Setup

```bash
git clone https://github.com/liangyu-suli/share-bilingual-article-to-wechat
cd share-bilingual-article-to-wechat

cp .env.example .env
chmod 600 .env
# edit .env:
#   - WECHAT_APPID, WECHAT_SECRET — required, from the WeChat console
#   - DEEPSEEK_API_KEY            — optional, enables the Chinese-LLM
#                                    fluency critic; omit to let the
#                                    agent do the fluency review itself
```

WeChat requires a **static outbound IP**. Add yours to the whitelist:

- WeChat Console → Settings → Development → Basic Configuration → IP Whitelist
- Check your current IP: `curl -s https://api.ipify.org`
- On a rotating home IP, refresh the whitelist when it changes — or run behind a fixed-IP VPN / cloud relay.

The pipeline runs a preflight check on every invocation and halts with a clear message if `.env` is missing, credentials are still placeholders, or a dependency isn't installed — so misconfiguration surfaces before a translation pass.

## Usage

From the project root, in your agent:

```
/share-bilingual-article-to-wechat https://example.com/some-article
/share-bilingual-article-to-wechat path/to/article.md
```

The agent will:

1. Run preflight
2. Fetch the article (verbatim, with related-links / comments / footers stripped at the source)
3. Translate through the six-stage review pipeline
4. Ask which of 20 WeChat HTML themes to use (default: `wechat-elegant`)
5. Save five files under `<slug>/`
6. Push the bilingual HTML as a draft to your WeChat Official Account

## Output

Each article lands as a named folder at the project root:

| File | Contents |
|------|----------|
| `<slug>.md` | Original English source (post-strip) |
| `<slug>.zh.md` | Simplified Chinese translation |
| `<slug>.bilingual.md` | Interleaved EN + ZH (paragraph by paragraph) |
| `<slug>.wechat.html` | WeChat HTML — Chinese only |
| `<slug>.bilingual.wechat.html` | WeChat HTML — bilingual |

Past runs stay in the repo as an archive — see [SOURCES.md](SOURCES.md) for the index.

## Repo layout

```
.
├── .claude/commands/
│   └── share-bilingual-article-to-wechat.md   # the pipeline prompt
├── .env.example                                # template for WeChat creds
├── glossary.json                               # approved term pairs
├── WORKFLOW.md                                 # pipeline spec
├── AGENTS.md                                   # agent role contract
├── SOURCES.md                                  # index of translated articles
└── <slug>/                                     # one folder per article
```

## Credits

- WeChat draft publishing: [md2wechat](https://github.com/geekjourneyx/md2wechat-skill)
- WeChat HTML themes: [foolgry/editor](https://github.com/foolgry/editor)
