# /share-bilingual-article-to-wechat

Translate an English blog post into Simplified Chinese and push a bilingual draft to WeChat.

## Usage

```
/share-bilingual-article-to-wechat <url>
/share-bilingual-article-to-wechat <path-to-markdown-file>
```

---

## Pipeline

### Preflight — Verify Prerequisites

Run this check **first**, before any fetching or translation. If anything is
missing, halt immediately and tell the user exactly what to fix — do not waste
a translation pass on a run that can't push.

```bash
fail() { echo "PREFLIGHT FAIL: $*" >&2; exit 1; }

# 1. .env exists at project root
[[ -f .env ]] || fail ".env is missing. Copy .env.example to .env and fill in your WeChat credentials."

# 2. WECHAT_APPID and WECHAT_SECRET are set to real (non-placeholder) values
set -a
source .env
set +a

[[ -n "${WECHAT_APPID:-}"  && "$WECHAT_APPID"  != "your_appid_here"      ]] || fail "WECHAT_APPID is not set in .env. Get it from the WeChat console → Settings → Development → Basic Configuration."
[[ -n "${WECHAT_SECRET:-}" && "$WECHAT_SECRET" != "your_appsecret_here"  ]] || fail "WECHAT_SECRET is not set in .env. Generate or reveal it in the WeChat console (same page as the AppID)."

# 3. md2wechat is on PATH (WeChat draft API client)
command -v md2wechat >/dev/null 2>&1 || fail "md2wechat is not installed or not on PATH. See README — install via 'go install' or download the binary from https://github.com/geekjourneyx/md2wechat-skill."

# 4. Python + Pillow available for cover image generation
python3 -c 'import PIL' 2>/dev/null || fail "Python Pillow is not installed. Run: pip install Pillow"

# 5. Node.js is on PATH (wxmd-cli is a Node script invoked in Stage 6)
command -v node >/dev/null 2>&1 || fail "Node.js is not installed or not on PATH. Install it (brew install node, nvm, etc.) before re-running."

# 6. wxmd-cli checkout exists at the expected path
WXMD_CLI=/tmp/foolgry-editor/wxmd-cli/src/index.js
[[ -f "$WXMD_CLI" ]] || fail "wxmd-cli not found at $WXMD_CLI. Clone it with: git clone --depth 1 https://github.com/foolgry/editor /tmp/foolgry-editor  (note: /tmp is wiped on reboot — re-clone if it disappears)."

# 7. DEEPSEEK_API_KEY for the Stage 4 fluency critic — optional.
#    `.env` was already sourced above, so a key defined there takes precedence;
#    if `.env` omits the key, whatever is already in the shell environment
#    flows through. If neither defines it, Stage 4 falls back to this agent's
#    own native review. When the key IS present, `jq` must also be available
#    to build the DeepSeek request payload safely.
if [[ -n "${DEEPSEEK_API_KEY:-}" ]]; then
    command -v jq >/dev/null 2>&1 || fail "jq is not installed (Stage 4 uses it to build the DeepSeek request payload). Install with: brew install jq — or unset DEEPSEEK_API_KEY to fall back to native review."
    echo "DeepSeek fluency critic: enabled (model: ${DEEPSEEK_MODEL:-deepseek-chat})."
else
    echo "DeepSeek fluency critic: disabled (DEEPSEEK_API_KEY not set) — Stage 4 will use the agent's own native review."
fi

echo "Preflight OK."
```

When you (the agent) report a preflight result to the user, **also surface the
DeepSeek line above verbatim** so they know which fluency path Stage 4 will
take on this run. If they expected the critic to be enabled and it isn't,
they'll want to set the key and re-run rather than learn after the fact.

When you (the agent) report a preflight failure to the user, name the specific
missing piece and quote the exact step they need to take. Examples:

- ".env is missing — please run `cp .env.example .env` and fill in your
  `WECHAT_APPID` and `WECHAT_SECRET` from the WeChat console."
- "`WECHAT_SECRET` is still the placeholder value — please paste your real
  AppSecret into `.env`."
- "`md2wechat` isn't installed — please install it before re-running this skill."
- "`node` isn't installed — wxmd-cli needs a Node.js runtime."
- "wxmd-cli isn't at `/tmp/foolgry-editor/wxmd-cli/src/index.js` — please run
  the git clone command from the README. /tmp is wiped on reboot, so this can
  happen even after a successful first install."
- "`DEEPSEEK_API_KEY` is set but `jq` isn't installed — please run
  `brew install jq`, or unset the key to fall back to native fluency review."

After a successful preflight, the WeChat credentials are already exported into
the environment for the rest of the run, so later stages do not need to re-source
`.env`.

### Stage 0 — Fetch (URL only)

If the input is a URL, fetch the raw HTML using `curl` — never use `WebFetch` for
content extraction, as it paraphrases instead of preserving the original text.

Always bind the URL to a shell variable and quote it — never interpolate the user-
supplied string directly into the command line:

```bash
URL='<paste-url-here>'
curl -sL --max-time 30 \
  -A 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36' \
  "$URL" | python3 -c "
import sys, re
html = sys.stdin.read()
html = re.sub(r'<!--.*?-->', '', html, flags=re.DOTALL)
html = re.sub(r'<(script|style|nav|footer|header)\b[^>]*>.*?</\1>', '', html, flags=re.DOTALL | re.IGNORECASE)
text = re.sub(r'<[^>]+>', ' ', html)
text = re.sub(r'[ \t]+', ' ', text)
text = re.sub(r'\n{3,}', '\n\n', text)
print(text.strip())
"
```

`-L` follows redirects, `--max-time 30` caps hangs, and the User-Agent header
keeps Cloudflare/Substack/Medium from returning empty bodies.

From the raw extracted text, identify and preserve:
- Title (exact wording)
- Author (for the WeChat draft `author` field)
- Every paragraph verbatim — do not rephrase, summarize, or rewrite a single word
- All headings, lists, blockquotes belonging to the article body

**Drop entirely** — do not include in the saved Markdown, do not translate, do
not re-attach later. Stripping at the source means the rest of the pipeline
never sees this content:

- **Article dek / subhead / subtitle** — the short descriptive line that often
  sits between the main title and the body. It's marketing copy, not article
  content, and reads as redundant once the body is right below.
- **Publish date** ("Jan 5, 2024", "January 5", "2 days ago", etc.)
- **Read-time** ("5 min read", "·5 minutes", etc.)
- Related Links / "Read more" / "More from this author" sections
- Comments section and every reader comment
- Footers, site navigation, share buttons, subscribe/CTA blocks
- Cookie banners, paywall prompts, newsletter pop-ups

Convert what remains to clean Markdown structure. Derive a slug from the URL
path and save to `<slug>/<slug>.md`. Proceed with that file.

> **Local-Markdown inputs:** if the input is already a Markdown file rather
> than a URL, apply the same drop-list before continuing — open the file, remove
> any related-links / comments / footer sections, save it back, then proceed
> from Stage 1 with the cleaned source.

### Stage 1 — Pre-processing

- Parse frontmatter: translate `title` (this is the *literal* translation —
  Stage 6 generates an appealing rewrite on top). If `description` is present
  (e.g. on local Markdown inputs that include a frontmatter dek), translate
  it; for URL inputs, the dek was already stripped in Stage 0 so this field
  is typically absent. Pass all other frontmatter fields verbatim.
- Mark non-translatable content: fenced code, inline code, image paths, URLs, raw HTML, **human names**.
- Load `glossary.json` if present — apply all entries exactly.

### Stage 2 — Translation

Translate to Simplified Chinese. Preserve all Markdown syntax. Apply glossary matches. Never translate code, URLs, or names.

### Stage 3 — Correctness Review

Compare translation against source paragraph by paragraph. Fix omissions, additions, mistranslations, glossary violations. If a source segment is genuinely ambiguous, surface it as a question before continuing:

```
Before I continue, I need to clarify:

1. [Segment: "…"] Question: … Options: a) … b) …
```

### Stage 4 — Fluency Review

Goal: read the post-Stage-3 draft as a native Chinese reader (no source
reference) and fix translationese, awkward word order, stiff grammar.

Two paths, chosen by whether `DEEPSEEK_API_KEY` is set (the preflight printed
which one is active):

- **DeepSeek critic + agent applies** (key set). A Chinese-trained model reads
  the draft and emits a list of suggested edits. You (the agent) apply them.
  The split keeps a single writer's voice (yours) while bringing in a model
  trained primarily on Chinese to catch issues a primarily-English model
  misses.
- **Native review** (key not set). You do the read yourself, with the same
  goals. Fallback path — same output shape, one fewer pair of eyes.

#### Path A — DeepSeek critic

1. Write the post-Stage-3 Chinese draft to `/tmp/<slug>.zh.draft.md`.

2. Call DeepSeek for a critique:

   ```bash
   curl -sS https://api.deepseek.com/v1/chat/completions \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $DEEPSEEK_API_KEY" \
     -d "$(jq -n \
       --arg model "${DEEPSEEK_MODEL:-deepseek-chat}" \
       --rawfile draft /tmp/<slug>.zh.draft.md \
       '{
         model: $model,
         temperature: 0.3,
         messages: [
           {role:"system", content:"You are a native Simplified-Chinese editor. Read the Chinese text below WITHOUT any English source. Flag spots that read as translationese — awkward word order, stiff syntax, redundant connectives, calques. For each issue output one line in the exact format:  <original phrase>  →  <suggested replacement>  ——  <one-line reason>. Do not rewrite the whole text. If it already reads naturally, output the single line: NO_EDITS."},
           {role:"user",   content:$draft}
         ]
       }')" \
     | jq -r '.choices[0].message.content'
   ```

   `DEEPSEEK_MODEL` defaults to `deepseek-chat`. Override in `.env` to use a
   different DeepSeek model (e.g. a newer Flash release).

3. Apply each suggested edit when it genuinely improves fluency; skip any that
   change meaning or break Markdown structure. If the critique is `NO_EDITS`,
   keep the draft as-is.

4. Proceed to Stage 5 with the post-edit draft. `/tmp/<slug>.zh.draft.md` is
   wiped on reboot — no manual cleanup needed.

5. If the curl call fails (network, auth, rate limit), don't halt — log a one-
   line notice to the user ("DeepSeek critic failed: <reason>; falling back to
   native review") and continue with Path B for this run.

#### Path B — Native review (fallback)

Read the draft yourself with a Chinese-reader hat on and apply the same kind
of edits — translationese, awkward order, stiff phrasing. Output is the same
shape; only the second pair of eyes is missing.

### Stage 5 — Style Refinement

Match the register of the source (formal / conversational / technical). No external style guide — honor the original voice.

### Stage 6 — Deliver

#### Step 1 — Generate an appealing ZH title

Before writing any output file, produce an appealing Simplified-Chinese
headline for the article. The literal translation from Stage 2 is the
starting point; what we want here is a punchy, idiomatic ZH headline suited
for a WeChat Official Account.

If `DEEPSEEK_API_KEY` is set, ask DeepSeek (it sees both the EN original and
the literal ZH so it can balance fidelity and appeal):

```bash
APPEALING_ZH_TITLE=$(curl -sS https://api.deepseek.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $DEEPSEEK_API_KEY" \
  -d "$(jq -n \
    --arg model "${DEEPSEEK_MODEL:-deepseek-chat}" \
    --arg en "<EN title>" \
    --arg zh "<literal ZH title from Stage 2>" \
    '{
      model: $model,
      temperature: 0.7,
      messages: [
        {role:"system", content:"You write Simplified-Chinese headlines for a WeChat Official Account. Given an English article title and its literal Chinese translation, produce ONE appealing, punchy, idiomatic headline. Keep it close in length to the literal version. Preserve the core meaning. Use natural Chinese phrasing that catches attention without becoming clickbait. Output ONLY the headline — no quotes, no explanation, no surrounding whitespace."},
        {role:"user",   content:("English: " + $en + "\nLiteral Chinese: " + $zh)}
      ]
    }')" \
  | jq -r '.choices[0].message.content')
```

If `DEEPSEEK_API_KEY` is unset, generate the appealing title yourself — same
constraints (punchy, idiomatic, faithful to meaning, no clickbait), no API
call. If the DeepSeek request fails, do the same — log a one-line notice and
fall back.

The appealing ZH title replaces the literal one in every output:

- `<slug>.zh.md` frontmatter `title`
- The H1 (or top heading) of `<slug>.bilingual.md`, paired with the EN title
- The cover image in Stage 7 (the `<zh_title>` placeholder in the PIL snippet)
- The WeChat draft `title` field — format stays `<EN title> / <appealing ZH
  title>`. The EN half is the "subtitle" side of the bilingual title concat
  and stays as-is.

#### Step 2 — Write the output files

Write four files (using `<slug>` as the folder and base name). The original
`<slug>.md` from Stage 0 stays in place, so the final folder holds five files.

**`<slug>.zh.md`** — Chinese only. Frontmatter with translated title/description + refined body.

**`<slug>.bilingual.md`** — Interleaved. English is copied verbatim — never
modified.

- **Paragraphs** are paired into a single Markdown paragraph: write the English
  line ending with a backslash (CommonMark hard line break), then the Chinese
  on the next line, then a blank line to close the paragraph. This renders as
  one `<p>EN<br>ZH</p>`, so the bottom margin sits between pairs instead of
  between the two languages — the EN and ZH read as a visual pair.

  ```markdown
  The cat sat on the mat.\
  猫坐在垫子上。

  The dog barked at the moon.\
  狗对着月亮叫。
  ```

- **Headings, list items, and blockquotes** stay as separate blocks — English
  unit followed immediately by its Chinese translation, blank line between
  pairs. (Hard breaks don't work inside headings, and list items already group
  visually.)

**Source-link footer (URL inputs only).** Append a separator and the source
URL at the end of the body of both `<slug>.zh.md` and `<slug>.bilingual.md`:

```markdown
---

原文链接：<original url>
```

The Stage 7 draft push also sets `content_source_url`, which WeChat renders as
the native **阅读原文** button at the bottom of the article — leave that as
is. The inline footer is additive (visible in the article body itself, and in
the `.md` files when read directly); the button is the official channel.

For **local Markdown inputs** (no URL): skip this footer — there's no
canonical source to link to.

**Style selection — ask the user before generating HTML.**

List the available themes and ask which to use:

```
Available styles:
  wechat-default      默认公众号风格
  latepost-depth      晚点风格
  wechat-ft           金融时报
  wechat-anthropic    Claude
  wechat-claude-song  Claude Song
  wechat-tech         技术风格
  wechat-elegant      优雅简约
  wechat-deepread     深度阅读
  wechat-nyt          纽约时报
  wechat-jonyive      Jony Ive
  wechat-medium       Medium 长文
  wechat-apple        Apple 极简
  kenya-emptiness     原研哉·空
  hische-editorial    Hische·编辑部
  ando-concrete       安藤·清水
  gaudi-organic       高迪·有机
  kami                Kami
  guardian            Guardian 卫报
  nikkei              Nikkei 日経
  lemonde             Le Monde 世界报

Which style would you like? (default: wechat-elegant)
```

Use the chosen style (or `wechat-elegant` if the user skips) for both HTML files below.

**`<slug>.wechat.html`** — Chinese-only WeChat HTML:

```bash
cd /tmp/foolgry-editor/wxmd-cli
node src/index.js typeset --input <slug>/<slug>.zh.md --style <chosen-style> --output html > <slug>/<slug>.wechat.html
```

**`<slug>.bilingual.wechat.html`** — Bilingual WeChat HTML:

```bash
cd /tmp/foolgry-editor/wxmd-cli
node src/index.js typeset --input <slug>/<slug>.bilingual.md --style <chosen-style> --output html > <slug>/<slug>.bilingual.wechat.html
```

No post-HTML cleanup is needed — related links, comments, and footers were
already dropped from the Markdown in Stage 0, so the generated HTML contains
only the article body.

Tell the user the paths to open both `.wechat.html` files in their browser for local preview.

### Stage 7 — Push to WeChat Draft

Push the **bilingual** WeChat HTML as a draft. Credentials were already loaded
into the environment by preflight (`set -a; source .env; set +a`), so run
`md2wechat` directly — **never** inline `WECHAT_APPID=… WECHAT_SECRET=…` per
command, as that writes the secrets to shell history.

If for any reason the environment has been cleared (new shell, agent restart),
re-source `.env` before continuing:

```bash
set -a
source .env
set +a
```

**Article title:** `<EN title> / <appealing ZH title>` — the EN half is the
"subtitle" side of the bilingual concat and stays verbatim; the ZH half is
the appealing headline produced in Stage 6 Step 1.
**Author:** original article author (from the source metadata)

Steps:

1. Generate a cover image — solid `#07c160` background, white ZH title centred,
   using a CJK-capable font (the PIL default bitmap font cannot render Chinese):

   ```python
   from PIL import Image, ImageDraw, ImageFont

   img = Image.new('RGB', (900, 500), color=(7, 193, 96))
   draw = ImageDraw.Draw(img)

   font = None
   for path in (
       '/System/Library/Fonts/PingFang.ttc',
       '/System/Library/Fonts/STHeiti Medium.ttc',
       '/System/Library/Fonts/Hiragino Sans GB.ttc',
   ):
       try:
           font = ImageFont.truetype(path, 72)
           break
       except OSError:
           continue
   if font is None:
       font = ImageFont.load_default()

   draw.text((450, 250), '<zh_title>', fill='white', anchor='mm', font=font)
   img.save('/tmp/<slug>-cover.jpg', quality=92)
   ```

2. Upload the cover and capture the `media_id` from the JSON response into a
   shell variable so the next step can reference it:

   ```bash
   MEDIA_ID=$(md2wechat upload_image /tmp/<slug>-cover.jpg \
     | python3 -c 'import sys, json; print(json.load(sys.stdin)["media_id"])')
   echo "Cover media_id: $MEDIA_ID"
   ```

   If `md2wechat`'s output wraps the WeChat API response under a different key
   (e.g. `.data.media_id` or `.thumb_media_id`), adjust the JSON path in the
   `python3 -c` snippet — the rest of this stage uses `$MEDIA_ID` as the
   captured value.

3. Build `/tmp/<slug>-draft.json` using Python `json.dump` (never shell-escape
   HTML manually):

   ```json
   {"articles":[{
     "title": "<EN title> / <ZH title>",
     "author": "<original author>",
     "content": "<bilingual wechat html content>",
     "content_source_url": "<original url or empty string>",
     "thumb_media_id": "<value of $MEDIA_ID from step 2>",
     "need_open_comment": 0,
     "only_fans_can_comment": 0
   }]}
   ```

   `need_open_comment` is `0` by default. Only set it to `1` if the target account
   is a Service Account (服务号) that has been granted the comment permission —
   Subscription Accounts and accounts without the permission will reject the
   draft with an API error.

4. Push: `md2wechat create_draft /tmp/<slug>-draft.json`

5. Report the returned `media_id` and confirm the draft is ready in WeChat.

**If the push fails with IP whitelist error:** tell the user to add their
current outbound IP to the WeChat console IP whitelist (Settings → Development
→ Basic Configuration → IP Whitelist). WeChat requires a static outbound IP —
home networks with rotating IPs will need to refresh the whitelist on every
new address.

Check current outbound IP with: `curl -s https://api.ipify.org`

Finally, update `glossary.json`. Append a new entry **only if all three hold**:

1. The source term appears 2+ times in the article (or is a recognized term of art).
2. The translation is non-obvious — not a 1-to-1 dictionary mapping.
3. The same target should hold across future articles in the same domain.

Skip single-use phrases, context-dependent renderings, and proper nouns.

---

## Rules

| Element | Behavior |
|---|---|
| Frontmatter `title`, `description` | Translate |
| All other frontmatter fields | Verbatim |
| Headings, paragraphs, bold, italic, link text | Translate |
| Human names | Verbatim — never transliterate |
| URLs, image paths | Verbatim |
| Inline code, fenced code, HTML | Verbatim |
| Related links, comments, footers, share buttons, CTAs | Drop at source (Stage 0) |
