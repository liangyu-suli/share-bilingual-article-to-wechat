# Translation Workflow Spec

## Overview

A batch translation skill that converts English Markdown content into Simplified Chinese
and (optionally) pushes the result to a WeChat Official Account as a draft.

The pipeline is implemented as a Claude Code slash command — see
`.claude/commands/share-bilingual-article-to-wechat.md` for the executable prompt.
All stages run sequentially in a single agent pass. The agent surfaces clarifying
questions inline when source segments are ambiguous, and halts with a written report
instead of delivering when confidence falls below the gate.

## Pipeline Stages

### Stage 0 — Fetch (URL inputs only)
- Pull raw HTML with `curl` (never `WebFetch`, which paraphrases).
- Strip script/style/nav/footer/header and HTML comments.
- Identify and preserve: title, author, date, every paragraph verbatim, all headings,
  lists, blockquotes.
- Save as `<slug>/<slug>.md`. Continue from that file.

### Stage 1 — Pre-processing
- Parse frontmatter: translate `title` and `description`; pass through `slug`, `date`,
  `tags`, `author` verbatim.
- Segment the document into translatable units (paragraphs, headings, list items).
- Mark non-translatable content: fenced code, inline code, HTML, URLs, image paths,
  **human names**.
- Load glossary entries if available.

### Stage 2 — Translation
- Translate each segment to Simplified Chinese.
- Preserve all Markdown syntax (bold, italic, links, headings) exactly.
- Apply glossary terms where matched.
- Never translate content inside code spans or fenced blocks.

### Stage 3 — Correctness Review
- Compare each translated segment against its source.
- Check for: omissions, additions, mistranslations, glossary violations.
- Produce a corrected draft.
- If a source segment is genuinely ambiguous, surface a clarifying question to the
  user inline before continuing.

### Stage 4 — Fluency Review
- Read the corrected draft without referencing the source.
- Fix unnatural phrasing, grammar, and flow from a native Chinese reader's perspective.
- Do not reintroduce source-language structure.

### Stage 5 — Style Refinement
- Identify the register of the source (formal, conversational, technical).
- Ensure the Chinese output matches that register.
- No external style guide is applied — honor the original voice.

### Stage 6 — Confidence Scoring
Run as an **independent review pass** (ideally via a sub-agent) so the scoring is
not biased by the translator's own reasoning. Score 0.0–1.0 per dimension:

| Dimension | Description |
|---|---|
| `correctness` | Fidelity to source meaning |
| `fluency` | Natural readability in Chinese |
| `style` | Register match with source |

Overall score is the minimum of the three dimensions. If overall < **0.75**, halt
and report the issues to the user instead of delivering.

### Stage 7 — Delivery & Memory Update
- Write the final translated Markdown plus the bilingual / WeChat HTML variants
  (see the slash command for exact file layout).
- Append new approved term pairs to `glossary.json` (see "Glossary" below for the
  inclusion criteria).

## Markdown Handling Rules

| Element | Behavior |
|---|---|
| Frontmatter `title`, `description` | Translate |
| Frontmatter other fields | Pass through verbatim |
| Headings (`#`, `##`, ...) | Translate text, preserve `#` prefix |
| Paragraphs | Translate |
| Bold / italic | Translate text, preserve markers |
| Links | Translate link text, preserve URL |
| Images | Pass through verbatim |
| Inline code | Pass through verbatim |
| Fenced code blocks | Pass through verbatim |
| HTML blocks | Pass through verbatim |
| Human names | Pass through verbatim — never transliterate |

## Content Removal Rules

The "translate verbatim" rule has one explicit exception: when generating the
WeChat HTML variants, the following source sections are dropped entirely
rather than translated:

- Related Links / "Read more" sections
- Comments sections and reader comments
- Footers, navigation, share buttons

What remains and *is* translated verbatim: article title, subtitle, author/date line,
TL;DR / lede, body paragraphs.

## Glossary

Stored in `glossary.json` at the project root. Format:

```json
[
  { "source": "translation memory", "target": "翻译记忆库" },
  { "source": "fluency", "target": "流畅性" }
]
```

A new pair is appended **only if all three hold**:

1. The source term appears 2+ times in the article (or is a recognized term of art).
2. The translation is non-obvious — i.e. not a 1-to-1 dictionary mapping.
3. The same target should hold across future articles in the same domain.

Skip single-use phrases, context-dependent renderings, and proper nouns.
