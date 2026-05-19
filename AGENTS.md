# AGENTS.md

This file describes the agent roles and conventions for the translation workflow project.

## Project Overview

A multi-stage pipeline that translates English Markdown content into Simplified Chinese
and pushes a bilingual draft to a WeChat Official Account. Packaged as a Markdown
slash command — designed for Claude Code, runnable in any AI agent that loads
project command files.

## Skill Interface

Invoked as a Markdown slash command from the project root:

```
/share-bilingual-article-to-wechat <url-or-path-to-markdown>
```

The command runs the full pipeline (fetch → translate → review → deliver → push to
WeChat) in a single agent pass.

- If the source contains genuinely ambiguous segments, the agent surfaces clarifying
  questions to the user inline before continuing.
- Stage 4 (Fluency Review) optionally delegates to a Chinese-native LLM
  (DeepSeek) as a critic when `DEEPSEEK_API_KEY` is set; the agent applies the
  suggested edits. Without the key it does the fluency read itself.
- Stage 6 generates an appealing ZH title (DeepSeek when the key is set,
  otherwise the agent itself) and uses it in every output. The bilingual
  WeChat title stays `<EN title> / <appealing ZH title>` — the EN half is the
  preserved "subtitle" line.

See `.claude/commands/share-bilingual-article-to-wechat.md` for the executable prompt
and `WORKFLOW.md` for the pipeline spec.

## Agent Roles

These roles are conceptual — the actual implementation runs them as sequential phases
within a single agent pass.

### Orchestrator
- Drives the pipeline: pre-processing → translation → correctness → fluency →
  style → delivery → push.
- Accumulates clarifying questions during correctness review; halts and asks the user
  before fluency review if any exist.
- Surfaces the preflight's "DeepSeek fluency critic: enabled/disabled" line to the
  user so they know which Stage 4 path the run is taking.

### Preprocessor
- Parses frontmatter: marks `title` as translatable (literal — the appealing
  rewrite happens in Stage 6); translates `description` when present (URL
  inputs usually won't have one, since the dek is stripped in Stage 0). All
  other fields pass through verbatim.
- Segments the document into translatable units.
- Marks non-translatable blocks: fenced code, inline code, HTML, URLs, image paths,
  human names.
- Loads `glossary.json` if present.

### Translator
- Translates each segment into Simplified Chinese.
- Preserves all Markdown syntax exactly.
- Applies glossary matches.
- Never translates content inside code spans or fenced blocks.

### Correctness Reviewer
- Compares each translated segment against its source sentence-by-sentence.
- Checks for omissions, additions, mistranslations, glossary violations.
- Produces a corrected draft.
- Adds ambiguous source segments to the clarifying-questions list.

### Fluency Reviewer
- Reads the corrected draft without referencing the source.
- Fixes unnatural phrasing, grammar, and flow as a native Chinese reader.
- Does not reintroduce source-language structure.
- **When `DEEPSEEK_API_KEY` is set**, delegates the read to DeepSeek as a
  critic (system prompt asks for a list of `<original> → <suggestion> ——
  <reason>` edits) and applies the suggestions. Override the model via
  `DEEPSEEK_MODEL` (default: `deepseek-chat`). If the API call fails, logs a
  one-line notice and falls back to native review for this run.
- **Without the key**, performs the fluency read itself — same goals, same
  output shape, one fewer pair of eyes.

### Style Refiner
- Identifies the register of the source (formal, conversational, technical).
- Ensures the Chinese output matches that register.
- No external style guide — honors the original voice.

### Delivery Agent
- Generates the appealing ZH title (DeepSeek when keyed, otherwise itself)
  and uses it in every output.
- Writes the final translated Markdown and the bilingual / WeChat HTML variants.
- Appends approved segment pairs to `glossary.json` (see WORKFLOW.md for the
  inclusion criteria).
- Pushes the bilingual WeChat HTML as a draft via `md2wechat`.

## Conventions

- Source and target: English → Simplified Chinese (`zh-Hans`).
- Reference documents are organized into named folders at the project root
  (e.g., `boil-the-ocean/`).
- WeChat credentials live in `.env` (gitignored, mode 0600). Source them once per
  shell session rather than inlining them per command.
- See `WORKFLOW.md` for the full pipeline spec and Markdown handling rules.
- See `.claude/commands/share-bilingual-article-to-wechat.md` for the executable
  pipeline.
