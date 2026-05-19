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

# 3. md2wechat is on PATH
command -v md2wechat >/dev/null 2>&1 || fail "md2wechat is not installed or not on PATH. See README — install via 'go install' or download the binary from https://github.com/geekjourneyx/md2wechat-skill."

# 4. Python + Pillow available for cover image generation
python3 -c 'import PIL' 2>/dev/null || fail "Python Pillow is not installed. Run: pip install Pillow"

echo "Preflight OK."
```

When you (the agent) report a preflight failure to the user, name the specific
missing piece and quote the exact step they need to take. Examples:

- ".env is missing — please run `cp .env.example .env` and fill in your
  `WECHAT_APPID` and `WECHAT_SECRET` from the WeChat console."
- "`WECHAT_SECRET` is still the placeholder value — please paste your real
  AppSecret into `.env`."
- "`md2wechat` isn't installed — please install it before re-running this skill."

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
- Author, date, read time if present
- Every paragraph verbatim — do not rephrase, summarize, or rewrite a single word
- All headings, lists, blockquotes

Convert to clean Markdown structure. Derive a slug from the URL path and save to `<slug>/<slug>.md`. Proceed with that file.

### Stage 1 — Pre-processing

- Parse frontmatter: translate `title` and `description`; pass all other fields verbatim.
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

Read the translation as a native Chinese reader — no source reference. Fix unnatural phrasing, grammar, stiff constructions.

### Stage 5 — Style Refinement

Match the register of the source (formal / conversational / technical). No external style guide — honor the original voice.

### Stage 6 — Confidence Scoring

Run this stage as an **independent review pass** — delegate it to a sub-agent
(via the Agent tool) so the scorer does not see the translator's reasoning. The
sub-agent receives the source and the final draft and returns a score 0.0–1.0
on each of:

- `correctness` — fidelity to source meaning
- `fluency` — natural readability in Chinese
- `style` — register match with source

Overall = minimum of the three. If overall < **0.75**, halt and report the
flagged segments instead of delivering.

### Stage 7 — Deliver

Write four files (using `<slug>` as the folder and base name). The original
`<slug>.md` from Stage 0 stays in place, so the final folder holds five files.

**`<slug>.zh.md`** — Chinese only. Frontmatter with translated title/description + refined body.

**`<slug>.bilingual.md`** — Interleaved. Each English unit (heading, paragraph, list block) followed immediately by its Chinese translation. English is copied verbatim — never modified.

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

**Content removal.** After generating either HTML, strip the following sections
by removing the corresponding DOM nodes (use Python + html.parser if needed):
- Related Links / "Read more" sections
- Comments section and any reader comments
- Footers, navigation, share buttons

Keep: article title, subtitle, author/date line, TL;DR / lede, body paragraphs.

Tell the user the paths to open both `.wechat.html` files in their browser for local preview.

### Stage 8 — Push to WeChat Draft

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

**Article title:** `<EN title> / <ZH title>` (e.g. `Boil the Ocean / 煮沸海洋`)
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

2. Upload cover: `md2wechat upload_image /tmp/<slug>-cover.jpg` — capture
   `media_id` from the JSON response.

3. Build `/tmp/<slug>-draft.json` using Python `json.dump` (never shell-escape
   HTML manually):

   ```json
   {"articles":[{
     "title": "<EN title> / <ZH title>",
     "author": "<original author>",
     "content": "<bilingual wechat html content>",
     "content_source_url": "<original url or empty string>",
     "thumb_media_id": "<media_id from step 2>",
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
| Related links, comments, footers (WeChat HTML only) | Drop |
