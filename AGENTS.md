# AGENTS.md

This file describes the agent roles and conventions for the translation workflow project.

## Project Overview

A multi-stage pipeline that translates English Markdown content into Simplified Chinese
and pushes a bilingual draft to a WeChat Official Account. Packaged as a Claude Code
slash command so any agent can invoke it without knowing the internal stages.

## Skill Interface

Invoked as a Claude Code slash command:

```
/share-bilingual-article-to-wechat <url-or-path-to-markdown>
```

The command runs the full pipeline (fetch → translate → review → deliver → push to
WeChat) in a single agent pass.

- If the source contains genuinely ambiguous segments, the agent surfaces clarifying
  questions to the user inline before continuing.
- If overall confidence falls below 0.75, the agent halts and reports the issues
  instead of delivering.

See `.claude/commands/share-bilingual-article-to-wechat.md` for the executable prompt
and `WORKFLOW.md` for the pipeline spec.

## Agent Roles

These roles are conceptual — the actual implementation runs them as sequential phases
within a single Claude Code pass.

### Orchestrator
- Drives the pipeline: pre-processing → translation → correctness → fluency → style →
  scoring → delivery.
- Accumulates clarifying questions during correctness review; halts and asks the user
  before scoring if any exist.
- Halts and reports issues to the user if any confidence dimension falls below 0.75.

### Preprocessor
- Parses frontmatter: marks `title` and `description` as translatable; all other
  fields pass through verbatim.
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

### Style Refiner
- Identifies the register of the source (formal, conversational, technical).
- Ensures the Chinese output matches that register.
- No external style guide — honors the original voice.

### Confidence Scorer
- Runs as an independent review pass, ideally via a sub-agent so the scoring is not
  biased by the translator's own reasoning.
- Scores correctness, fluency, and style 0.0–1.0.
- Overall score is the minimum of the three dimensions.
- If overall < 0.75, signals the Orchestrator to halt for human review.

### Delivery Agent
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
