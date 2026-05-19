# Translation Workflow Spec

## Overview

A batch translation skill that converts English Markdown content into Simplified Chinese
and (optionally) pushes the result to a WeChat Official Account as a draft.

The pipeline is implemented as a Markdown slash command — see
`.claude/commands/share-bilingual-article-to-wechat.md` for the executable prompt.
Designed for Claude Code; runnable in any AI agent that can execute multi-step
Markdown instructions from a project's commands directory. All stages run
sequentially in a single agent pass. The agent surfaces clarifying questions
inline when source segments are ambiguous.

Stage 4 (Fluency Review) optionally delegates to a Chinese-native LLM
(DeepSeek) as a critic when `DEEPSEEK_API_KEY` is set; the agent then applies
the suggested edits. If the key is absent, Stage 4 falls back to the agent's
own native review.

## Pipeline Stages

### Stage 0 — Fetch (URL inputs only)
- Pull raw HTML with `curl` (never `WebFetch`, which paraphrases). Bind the
  user-supplied URL to a shell variable and quote it — never interpolate
  unquoted into the command line.
- Strip `<script>`, `<style>`, `<nav>`, `<footer>`, `<header>` blocks and HTML
  comments from the raw response.
- Identify and preserve from the article body: title, author, every paragraph
  verbatim, all headings, lists, and blockquotes.
- **Drop entirely at the source** — do not include in the saved Markdown, do not
  translate, do not re-attach later:
  - Article dek / subhead / subtitle (the descriptive line between title and body)
  - Publish date and read-time ("Jan 5, 2024", "5 min read", etc.)
  - Related Links / "Read more" / "More from this author" sections
  - Comments section and every reader comment
  - Footers, site navigation, share buttons, subscribe / CTA blocks
  - Cookie banners, paywall prompts, newsletter pop-ups
- Save what remains as `<slug>/<slug>.md`. Continue from that file.
- **Local Markdown inputs** apply the same drop-list before Stage 1 — open the
  file, remove any related-links / comments / footer sections, save back, then
  proceed.

### Stage 1 — Pre-processing
- Parse frontmatter: translate `title` (literal — Stage 6 produces the
  appealing rewrite on top). Translate `description` if present; for URL
  inputs the dek was already stripped in Stage 0, so the field is typically
  absent. Pass through `slug`, `date`, `tags`, `author` verbatim.
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
Read the corrected draft without referencing the source. Fix unnatural
phrasing, grammar, and flow from a native Chinese reader's perspective. Do not
reintroduce source-language structure.

Two paths, chosen by whether `DEEPSEEK_API_KEY` is set:

- **DeepSeek critic + agent applies** (key set). The agent writes the post-
  Stage-3 draft to a temp file, calls DeepSeek with a system prompt that asks
  for a numbered list of `<original> → <suggestion> —— <reason>` edits, then
  applies the edits that genuinely improve fluency. The split keeps a single
  writer's voice (the agent's) while bringing in a Chinese-trained model as a
  reader. Override the model via `DEEPSEEK_MODEL` (defaults to
  `deepseek-chat`). If the API call fails, log a one-line notice and fall
  back to native review for this run.
- **Native review** (key absent). The agent performs the fluency read itself
  with the same goals. Same output shape — only the second pair of eyes is
  missing.

The exact request payload lives in the slash command; this spec only fixes
the contract.

### Stage 5 — Style Refinement
- Identify the register of the source (formal, conversational, technical).
- Ensure the Chinese output matches that register.
- No external style guide is applied — honor the original voice.

### Stage 6 — Delivery & Memory Update
- **Generate an appealing ZH title.** Take the EN title plus the literal ZH
  title (from Stage 2) and produce a punchy, idiomatic Simplified-Chinese
  headline suited for WeChat — same meaning, more appeal, similar length, no
  clickbait. Delegated to DeepSeek when `DEEPSEEK_API_KEY` is set; otherwise
  the agent writes it. This appealing title replaces the literal one in every
  output (zh.md frontmatter, bilingual.md heading, cover image, WeChat draft
  title). The bilingual WeChat title stays `<EN title> / <appealing ZH title>`
  — the EN half is the "subtitle" line and is preserved as-is.
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

The "translate verbatim" rule has one explicit exception: the sections listed
in Stage 0's drop-list are removed from the source Markdown **before**
translation, not after. By the time the translator sees the document, only
the article body remains.

Stripping at the source means:

- The translation pipeline never spends a pass on content that's going to be dropped.
- The bilingual `<slug>.bilingual.md` and the generated WeChat HTML are clean
  by construction — no post-HTML DOM surgery needed.
- The original `<slug>.md` saved in Stage 0 is itself the cleaned source —
  there is no separate "full source" snapshot.

What is translated verbatim: article title, subtitle, author / date line, TL;DR
or lede, body paragraphs, headings, lists, and blockquotes belonging to the
article body.

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
