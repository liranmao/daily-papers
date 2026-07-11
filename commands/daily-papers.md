Fetch yesterday's top research papers and AI news for the reader profile defined in the Configuration block below, then produce a curated Obsidian note, a spoken-audio briefing, a cover image, and a short video. Follow these steps exactly.

---

## Configuration

Everything reader-specific lives here. Edit these values to make the command yours; the rest of the file reads from them. (See `config.example.md` in the repo for a fuller explanation of each field.)

- **OUTPUT_DIR** — absolute path to the folder where notes/audio/video are written.
  `~/Documents/DailyPapers`
- **NOTES_SUBDIR** — subfolder under OUTPUT_DIR for the markdown notes.
  `Notes`
- **BGM_PATH** — optional background-music track to mix under the audio; leave blank to skip.
  ``
- **READER_PROFILE** — one sentence describing who this is for and what they care about.
  `A PhD researcher in spatial omics, histology, and foundation models for biology.`
- **RESEARCH_INTERESTS** — the topics used to score paper relevance (one per line).
  - Spatial transcriptomics / spatial omics (Visium, MERFISH, Slide-seq, Xenium, Stereo-seq)
  - Computational histology / digital pathology (H&E, WSI, tissue segmentation)
  - Foundation models for biology / pathology (ViTs, large-scale pretraining)
  - Single-cell genomics methods with a spatial component
  - Multimodal omics integration
- **ARXIV_CATEGORIES** — arXiv categories to pull, as two comma-groups (bio, then CS/ML).
  Group 1: `q-bio.GN,q-bio.QM,q-bio.CB`
  Group 2: `cs.CV,cs.LG,eess.IV`
- **JOURNALS** — the high-impact journals to search on PubMed and to keep after filtering.
  Nature, Science, Nature Methods, Nature Biotechnology, Nature Medicine, Nature Biomedical Engineering, Nature Computational Science, Nature Communications, Cell
- **PUBMED_KEYWORDS** — Title/Abstract terms OR'd together in the PubMed query.
  spatial transcriptomics, spatial omics, histology, whole slide, foundation model, single cell, deep learning, digital pathology, multiomics, sequencing
- **TOPIC_TAGS** — controlled vocabulary for the note's `topics` field and cover image.
  空间转录组, 单细胞, 数字病理, 基础模型, 多模态, 轨迹推断, WSI, 多组学, 空间组学
- **EN_VOICE** / **CN_VOICE** — edge-tts voices for the English and Chinese briefings.
  `en-US-AriaNeural` / `zh-CN-XiaoxiaoNeural`
- **MAKE_CHINESE** — set to `yes` to also produce the Chinese note + audio (Step 9), `no` to skip.
  `yes`

Throughout the steps below, wherever a path or list is needed, substitute the value from this block. `<TODAYS-DATE>` means today's date in `YYYY-MM-DD`.

---

## Step 1 — Fetch from arXiv (preprints)

Use WebFetch to call the arXiv API. Use yesterday's date in `YYYYMMDD` format for the date range.

Fetch URL 1 (from ARXIV_CATEGORIES group 1):
`https://export.arxiv.org/api/query?search_query=cat:<G1_A>+OR+cat:<G1_B>+OR+cat:<G1_C>&sortBy=submittedDate&sortOrder=descending&max_results=40`

Fetch URL 2 (from ARXIV_CATEGORIES group 2):
`https://export.arxiv.org/api/query?search_query=cat:<G2_A>+OR+cat:<G2_B>+OR+cat:<G2_C>&sortBy=submittedDate&sortOrder=descending&max_results=40`

Substitute the categories from ARXIV_CATEGORIES. Keep only papers whose `<published>` date is yesterday. Discard anything older.

---

## Step 1b — Fetch yesterday's top AI news

Fetch from these three sources in parallel:

**Source 1 — Hacker News:**
`https://hacker-news.firebaseio.com/v0/topstories.json`

This returns a list of story IDs. Fetch the top 30 item details one by one:
`https://hacker-news.firebaseio.com/v0/item/<id>.json`

Keep only stories whose title mentions AI, LLM, machine learning, GPT, Claude, Gemini, OpenAI, Anthropic, model, or neural. Filter to stories posted within the last 48 hours (use the `time` Unix timestamp field).

**Source 2 — The Verge AI RSS:**
`https://www.theverge.com/rss/ai-artificial-intelligence/index.xml`

**Source 3 — MIT Technology Review RSS:**
`https://www.technologyreview.com/feed/`

For RSS feeds, parse `<item>` entries and keep only those published yesterday.

**De-duplicate AI news against previous daily notes (required — do this before selecting).** RSS feeds and Hacker News keep the same big story on the front page for several days, so the same item will resurface day after day. Never re-run a news item that already appeared in a prior note. Check the `## AI News` section of recent notes and drop any candidate whose story is already covered:

```bash
cd "OUTPUT_DIR/NOTES_SUBDIR/"
# Pull the AI News sections from the last ~5 notes to see what was already run
for f in $(ls -t *.md | grep -v "<TODAYS-DATE>" | head -5); do
  echo "=== $f ==="
  awk '/## AI News/{flag=1} /^## Paper 1/{flag=0} flag' "$f"
done
```

Read those sections and exclude any candidate that reports the **same underlying event** as something already covered — match on the story, not the exact wording. A genuinely new development in an ongoing saga is allowed only if it adds a concrete new fact (a ruling, a new number, a reversal) — otherwise drop it.

**Select top 3 AI news items** across all sources — prioritise significance (major model releases, policy, breakthroughs) over minor updates. If de-duplication leaves fewer than 3 fresh significant items, widen the net with Hacker News search (`https://hn.algolia.com/api/v1/search?tags=front_page&hitsPerPage=40` and `https://hn.algolia.com/api/v1/search_by_date?query=AI&tags=story&numericFilters=points%3E80&hitsPerPage=30`), keeping only items from the last ~48 hours and not already covered, rather than re-running a stale headline. For each item keep four things:
- **headline** — plain and specific, name the actor/product
- **what happened** — one sentence, concrete facts (who, what, number if there is one)
- **why it matters** — one short sentence on the "so what": who is affected or what changes
- **source name**

Write these like you're telling a smart colleague over coffee, not reading a press release. No breathless adjectives; let the fact carry the weight.

---

## Step 2 — Fetch from PubMed (high-impact journals)

Use WebFetch to search PubMed for papers from the JOURNALS list. These are the highest priority — always prefer them over arXiv preprints.

**Why a wider window:** Nature-family journals publish online (epub) days or weeks before the print date. PubMed indexes them at epub time, so a strict single-day `pdat` filter misses most of them. Use `EDAT` (electronic deposit date) with a 4-day window to reliably catch recent publications.

Compute the dates:
- FOUR_DAYS_AGO = yesterday minus 3 days, as `YYYY/MM/DD`
- YESTERDAY = yesterday's date, as `YYYY/MM/DD`

Run **two parallel PubMed searches** and pool the results. Build the `("Journal A"[Journal]+OR+...)` clause from JOURNALS and the `(term[Title/Abstract]+OR+...)` clause from PUBMED_KEYWORDS.

**Query A — EDAT window (catches epub-ahead-of-print):**
`https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=pubmed&term=(<JOURNALS_CLAUSE>)+AND+(<KEYWORDS_CLAUSE>)&datetype=edat&mindate=FOUR_DAYS_AGO&maxdate=YESTERDAY&retmax=20&retmode=json`

**Query B — pdat window (catches papers with yesterday's print date):**
`https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=pubmed&term=(<JOURNALS_CLAUSE>)+AND+(<KEYWORDS_CLAUSE>)&datetype=pdat&mindate=FOUR_DAYS_AGO&maxdate=YESTERDAY&retmax=20&retmode=json`

Replace the placeholders and dates. Deduplicate the combined ID list before fetching abstracts.

If the ID list is non-empty, fetch abstracts with:
`https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi?db=pubmed&id=<comma-separated-ids>&rettype=abstract&retmode=xml`

Parse each paper's title, authors, journal, abstract, and PMID from the XML.

**Strict journal filter:** After parsing, discard any paper whose journal name does not exactly match one on the JOURNALS list (case-insensitive). Do not keep near-misses (e.g. "Communications Biology", "Scientific Reports", "PNAS") unless they are on your list.

---

## Step 3 — Score relevance (1–10) across all sources

Pool all papers from Steps 1 and 2. Score each for relevance to RESEARCH_INTERESTS.

Score 8–10: directly addresses one of the interests
Score 5–7: adjacent (general methods in the field, relevant ML methods)
Score 1–4: unrelated — discard

**Journal bonus:** Add +1 to any paper from a top-tier journal on JOURNALS (e.g. Nature Methods, Nature Biotechnology, Nature, Cell). A journal paper scoring 8 beats an arXiv preprint scoring 9 when selecting the top 5.

**Method bonus (highest priority):** Add +2 to any paper whose primary contribution is a **new computational method, algorithm, model, or software tool** — a paper that *builds* something, as opposed to one that *applies* existing methods to a domain question. When two papers score equally, always rank the method paper above the application paper. The reader is a methods developer — new methods they could adopt or build on are the top priority.

---

## Step 3b — De-duplicate against previously covered papers

Before selecting the top 5, exclude any paper that already appeared in a prior note (PubMed's EDAT window overlaps and journals stay indexed for a while, so this check is required).

Build a stable identifier for each candidate: its **DOI** for PubMed papers, or its **arXiv id** for preprints. Then grep every existing note for those identifiers:

```bash
cd "OUTPUT_DIR/NOTES_SUBDIR/"
for id in <doi-or-arxiv-id-1> <doi-or-arxiv-id-2> ... ; do
  echo "=== $id ==="
  grep -rl "$id" --include='*.md' . | grep -v "<TODAYS-DATE>" || echo "FRESH"
done
```

Any candidate whose identifier is found in a note other than today's is a duplicate — drop it before ranking. Only papers marked `FRESH` are eligible. If dropping duplicates leaves fewer than 5 eligible papers scoring ≥ 5, pull in the next-best fresh candidates (noting lower relevance) rather than re-including a duplicate.

---

## Step 4 — Pick top 5

Select the 5 highest-scoring papers from the de-duplicated pool across both arXiv and PubMed (after the journal and method bonuses). When scores are equal, rank in this order: (1) method papers over application papers, (2) journal papers over preprints. Fill the list with method/tool papers whenever qualifying ones exist; fall back to application papers only if there aren't enough methods papers scoring ≥ 5. If fewer than 5 scored ≥ 5, include the next best and note lower relevance.

---

## Step 5 — Write the Obsidian markdown note

Create ONE file at:
`OUTPUT_DIR/NOTES_SUBDIR/YYYY-MM-DD — <Headline>.md`

Where `<Headline>` is a combined, eye-catching headline that weaves together the biggest AI news item AND the top paper finding — like a newspaper front page covering both. It should feel like breaking news. Use "&" or "—" to join the two threads. Keep it under 80 characters, no jargon, no quotes.

Examples of good combined headlines:
- "Karpathy Joins Anthropic & AI Reads Tumors at Single-Cell Scale"
- "GPT-5 Drops as Spatial Atlas Maps the Human Brain in 3D"

Also write a Chinese version of the headline — vivid, punchy, same energy. This goes inside the note only, not in the filename.

For `journals`: list the unique journal names of the top 5 papers (deduplicated).
For `topics`: list the unique tags covered across all 5 papers, drawn from TOPIC_TAGS.
For `top_paper`: one-line description of Paper 1 (the highest-scoring paper).

Use this exact format:

```
---
date: YYYY-MM-DD
category: DailyPapers
top_paper: "<one-line description of top paper>"
journals: [<journal1>, <journal2>, ...]
topics: [<topic1>, <topic2>, ...]
---

# <English combined headline>
## <Chinese combined headline>

> YYYY-MM-DD

---

## AI News

**<headline>** — <what happened, one sentence>. **Why it matters:** <one sentence>. *(Source)*
**<headline>** — <what happened, one sentence>. **Why it matters:** <one sentence>. *(Source)*
**<headline>** — <what happened, one sentence>. **Why it matters:** <one sentence>. *(Source)*

---

## Paper 1 — <news-style headline for this paper>

**Authors:** <first author> et al. | **Journal:** <journal name> | **Year:** YYYY
**URL:** <DOI link or arXiv URL>
**Relevance:** <high | medium | low> — <half-line reason it made today's cut>

**What's new:** <one plain-language line on the core contribution — spell out any acronym the first time it appears>

### The gist
> <2–3 conversational sentences. Explain it the way you'd explain it to a sharp labmate who doesn't work in this exact subfield: what problem were they stuck on, what did they actually do, and what came out. Avoid undefined jargon. No "leverages", "utilizes", "novel framework" filler.>

**Why it matters to you:** <1–2 sentences connecting to the reader's own work (see READER_PROFILE). Be concrete: could they adopt the method, borrow an idea, use the dataset/benchmark, or does it challenge an assumption? If the link is thin, say honestly why it's still worth a glance rather than forcing a connection.>

**Worth noting:** <one sentence — an honest caveat, limitation, or open question inferable from the abstract (small cohort, one tissue type, no code, claim not yet validated). Omit only if the abstract genuinely offers nothing.>

---

## Paper 2 — <news-style headline>

[same structure as Paper 1]

---

## Paper 3 — <news-style headline>

[same structure as Paper 1]

---

## Paper 4 — <news-style headline>

[same structure as Paper 1]

---

## Paper 5 — <news-style headline>

[same structure as Paper 1]
```

Fill in every field you can from the abstract. Leave fields as `-` if not determinable.

**Tone rule for the whole note:** write like a knowledgeable friend who reads these papers so the reader doesn't have to — warm, direct, plain-spoken. Respect their intelligence but don't hide behind jargon. Every paper should leave them knowing (1) what it is in one breath, (2) whether it touches their work, and (3) what to be skeptical of.

At the bottom of the file, after Paper 5, append this section — leave it empty for now:

```
---

## Radio Script

```

---

## Step 6 — Write the radio script and generate audio

### 6a — Write the radio script

First, run this command to get today's exact date for the script:
```bash
date "+%A, %B %-d"
```
Use the output verbatim in the greeting — do not compute the weekday yourself.

Write the script in natural spoken English — no markdown, no symbols, no URLs, no asterisks. Write it as a calm, smart science radio host opening their morning show — someone who genuinely finds this interesting and wants the listener to get it, not just hear it. When a paper uses a technical term, unfold it in a few spoken words the first time. For each paper, land the "so what" out loud. Vary sentence length so it breathes. Use this structure:

```
Good morning. It's [output of date command above]. Here's your Daily Papers briefing.

Before the papers, a quick look at yesterday's AI news. [News item 1 — 1–2 natural sentences: what happened, then why it matters, name the source]. [News item 2 — same]. [News item 3 — same].

Now for yesterday's papers.

[Paper 1 — transition: "First up,"]
[3–4 sentences: what they did, key result, why it matters. Name the journal naturally, e.g. "published yesterday in Nature Methods".]

[Paper 2 — transition: "Next," / "Our second paper,"]
[3–4 sentences.]

[Paper 3 — transition: "Third,"]
[3–4 sentences.]

[Paper 4 — transition: "Fourth,"]
[3–4 sentences.]

[Paper 5 — transition: "And finally,"]
[3–4 sentences.]

That's your Daily Papers briefing for [Today's Month Day]. Have a productive day.
```

Do two things with this script:
1. **Save to `/tmp/daily_papers_script.txt`** (for audio generation)
2. **Append into the markdown note** under the `## Radio Script` section — paste the full plain text inside a blockquote (`> `) so it renders cleanly in Obsidian.

### 6b — Generate speech and mix with BGM

**Step 1 — Generate speech** (uses EN_VOICE):
```bash
edge-tts --voice EN_VOICE --rate=-3% -f /tmp/daily_papers_script.txt --write-media /tmp/daily_papers_speech.mp3
```

**Step 2 — Check if BGM exists** (skip mixing if BGM_PATH is blank):
```bash
ls "BGM_PATH"
```

**Step 3a — If BGM exists, mix with ffmpeg:**
```bash
ffmpeg -y -i /tmp/daily_papers_speech.mp3 \
  -stream_loop -1 -i "BGM_PATH" \
  -filter_complex "[1:a]volume=0.08,afade=in:d=3,afade=out:st=9999:d=4[bgm];[0:a][bgm]amix=inputs=2:duration=first[out]" \
  -map "[out]" -codec:a libmp3lame -q:a 3 \
  "OUTPUT_DIR/NOTES_SUBDIR/audio/YYYY-MM-DD — Daily Papers.mp3"
```

`-stream_loop -1` loops the BGM to cover the full speech length. `volume=0.08` keeps it subtle at 8%.

**Step 3b — If BGM does not exist or BGM_PATH is blank, use speech only:**
```bash
mkdir -p "OUTPUT_DIR/NOTES_SUBDIR/audio"
cp /tmp/daily_papers_speech.mp3 "OUTPUT_DIR/NOTES_SUBDIR/audio/YYYY-MM-DD — Daily Papers.mp3"
```

Do not fall back to another voice — if edge-tts fails, report the error. After the command runs, confirm the audio file was created.

---

## Step 7 — Print digest summary

Print a clean summary:

```
## Daily Papers — YYYY-MM-DD

### AI News
1. <headline> (Source)
2. <headline> (Source)
3. <headline> (Source)

### Papers
| # | Headline | Journal | Relevance |
|---|----------|---------|-----------|
| 1 | ...      | Nature Methods | high |
| 2 | ...      | arXiv   | medium |
...

📄 Note:  NOTES_SUBDIR/YYYY-MM-DD — <Headline>.md
🎧 Audio: NOTES_SUBDIR/audio/YYYY-MM-DD — Daily Papers.mp3
🎬 Video: NOTES_SUBDIR/video/YYYY-MM-DD — Daily Papers.mp4
```

---

## Step 8 — Generate cover image and video

### 8a — Prepare content variables

From the digest, derive the following. Do NOT copy or truncate existing titles — **rewrite each item** to be punchy and accurate.

**EP_DATE**: yesterday's date as `MM.DD`

**TITLE_ZH**: the Chinese combined headline from the note (the `## <Chinese headline>` line, without `##`).

**NEWS** — 3 items, each short. Distil to the sharpest factual claim; name the key actor or product.

**PAPERS** — 5 items, each short. Rules:
1. Always include the single most important tag from TOPIC_TAGS, matched to what the paper actually delivers.
2. Include a result or method hook — not just the topic.
3. Avoid generic words ("study", "method", "framework") standing alone.
4. **No special symbols** — no ×, →, ▸, ·, ÷ or arrow/math/bullet characters. A colon after a method name is fine.

### 8b — Write and run the cover image script

Write the following Python script to `/tmp/gen_cover_today.py`, filling in the variables from 8a. It uses cross-platform font fallbacks so it works on macOS, Linux, and Windows.

```python
EP_DATE  = "<MM.DD>"
TITLE_ZH = "<Chinese headline>"
NEWS    = ["<news 1>", "<news 2>", "<news 3>"]
PAPERS  = ["<p1>", "<p2>", "<p3>", "<p4>", "<p5>"]
OUTPUT  = "/tmp/cover_today.png"

from PIL import Image, ImageDraw, ImageFont
import random, glob, os

W, H = 1080, 1920
img = Image.new("RGB", (W, H))
draw = ImageDraw.Draw(img)
for y in range(H):
    t = y / H
    draw.line([(0, y), (W, y)], fill=(int(8+t*12), int(12+t*8), int(30+t*40)))

CYAN, WHITE, GREY, DIM = (0,210,220), (255,255,255), (140,155,175), (60,75,100)

# Cross-platform font resolution: try a CJK-capable font, then any TTF, then default.
CJK_CANDIDATES = [
    "/System/Library/Fonts/STHeiti Medium.ttc",            # macOS
    "/System/Library/Fonts/PingFang.ttc",                  # macOS
    "/usr/share/fonts/opentype/noto/NotoSansCJK-Medium.ttc",   # Linux (Noto CJK)
    "/usr/share/fonts/truetype/noto/NotoSansCJK-Regular.ttc",
    "C:/Windows/Fonts/msyh.ttc",                           # Windows (Microsoft YaHei)
]
LATIN_CANDIDATES = [
    "/System/Library/Fonts/HelveticaNeue.ttc",
    "/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf",
    "C:/Windows/Fonts/arial.ttf",
]
def first_existing(paths):
    for p in paths:
        if os.path.exists(p):
            return p
    return None
CJK  = first_existing(CJK_CANDIDATES) or first_existing(glob.glob("/usr/share/fonts/**/*.ttf", recursive=True))
LAT  = first_existing(LATIN_CANDIDATES) or CJK
def font(path, size):
    try:
        return ImageFont.truetype(path, size)
    except Exception:
        return ImageFont.load_default()

f_brand = font(LAT, 68)
f_title = font(CJK, 72)

draw.rectangle([80, 148, 400, 151], fill=CYAN)
draw.text((80, 168), "DailyPapers", font=f_brand, fill=WHITE)
bx, by = 460, 176
draw.rounded_rectangle([bx, by, bx+180, by+50], radius=25, fill=CYAN)
draw.text((bx+22, by+9), f"EP · {EP_DATE}", font=font(LAT, 28), fill=(5,15,35))
draw.rectangle([80, 280, W-80, 282], fill=DIM)

lines = [TITLE_ZH[i:i+13] for i in range(0, len(TITLE_ZH), 13)]
ty = 340
for line in lines:
    draw.text((80, ty), line, font=f_title, fill=WHITE); ty += 100
draw.rectangle([80, ty+10, 480, ty+14], fill=CYAN); ty += 80

draw.text((80, ty), "今日 AI 要闻", font=font(CJK, 40), fill=CYAN); ty += 60
for n in NEWS:
    draw.text((80, ty), f"· {n}", font=font(CJK, 38), fill=GREY); ty += 58
ty += 30; draw.rectangle([80, ty, W-80, ty+2], fill=DIM); ty += 30
draw.text((80, ty), "今日精选论文 · 5 篇", font=font(CJK, 40), fill=CYAN); ty += 60
for p in PAPERS:
    draw.text((80, ty), f"· {p}", font=font(CJK, 34), fill=(200,210,225)); ty += 52

ty += 40; bars = 38; bw = (W-160)//bars; random.seed(42)
for i in range(bars):
    bh = random.randint(12, 80); cx = 80 + i*bw + bw//2
    a = 0.35 + 0.65*(bh/80); c = tuple(int(v*a) for v in CYAN)
    draw.rounded_rectangle([cx-bw//2+2, ty, cx+bw//2-2, ty+bh], radius=4, fill=c)

img.save(OUTPUT, quality=95)
print(f"Saved: {OUTPUT}")
```

Then run:
```bash
python3 /tmp/gen_cover_today.py
```

### 8c — Combine cover image and audio into MP4

```bash
mkdir -p "OUTPUT_DIR/NOTES_SUBDIR/video"
ffmpeg -y \
  -loop 1 -i /tmp/cover_today.png \
  -i "OUTPUT_DIR/NOTES_SUBDIR/audio/YYYY-MM-DD — Daily Papers.mp3" \
  -c:v libx264 -tune stillimage -pix_fmt yuv420p \
  -c:a aac -b:a 192k -shortest \
  "OUTPUT_DIR/NOTES_SUBDIR/video/YYYY-MM-DD — Daily Papers.mp4"
```

After ffmpeg completes, confirm the file exists and print its size. The temp files are intermediate and need not be kept.

---

## Step 9 — Write Chinese translation note (only if MAKE_CHINESE = yes)

If MAKE_CHINESE is `no`, skip this step entirely.

Create a Chinese-language version of the note at:
`OUTPUT_DIR/NOTES_SUBDIR/CN/YYYY-MM-DD — <Chinese Headline>.md`

Where `<Chinese Headline>` is the Chinese combined headline (the `## <Chinese headline>` line, without `##`). Use the same date as the English note.

```bash
mkdir -p "OUTPUT_DIR/NOTES_SUBDIR/CN"
```

Use this format — translate all English prose, keep URLs, DOIs, author names, and journal names unchanged:

```
---
date: YYYY-MM-DD
category: DailyPapers
top_paper: "<中文一句话描述第一篇论文>"
journals: [<journal1>, <journal2>, ...]
topics: [<topic1>, <topic2>, ...]
lang: zh
---

# <中文标题>

> YYYY-MM-DD

---

## AI 要闻

**<标题>** — <发生了什么，一句话>。**为何重要：** <一句话>。*（来源）*
**<标题>** — <发生了什么，一句话>。**为何重要：** <一句话>。*（来源）*
**<标题>** — <发生了什么，一句话>。**为何重要：** <一句话>。*（来源）*

---

## 论文一 — <中文新闻式标题>

**作者：** <第一作者> 等 | **期刊：** <期刊名> | **年份：** YYYY
**链接：** <DOI 或 arXiv URL>
**相关性：** <高 | 中 | 低> — <半句话说明今天为何入选>

**新意何在：** <一句话通俗描述核心贡献，术语首次出现时用大白话解释>

### 速览
> <2–3 句对话式通俗总结：他们卡在什么问题上、具体做了什么、得到了什么结果。避免未经解释的术语，不要"利用""基于……的新颖框架"之类的空话。>

**与你的研究有何关系：** <1–2 句，联系读者自己的方向（见 READER_PROFILE）。要具体：能否直接用这个方法、借鉴某个思路、用它的数据集／基准，或它是否挑战了你的某个既有假设？若关联不强，就诚实说明为何仍值得一看。>

**值得留意：** <一句话，从摘要中能推断出的诚实局限或存疑之处。摘要确实无可说时才省略此行。>

---

## 论文二 — <中文新闻式标题>

[同论文一结构]

---

## 论文三 · 论文四 · 论文五

[同论文一结构，各自成节]
```

Translation guidelines:
- Translate all section headers, body text, and summaries into fluent, natural Chinese.
- Keep author names, journal names, DOIs, and arXiv URLs exactly as-is.
- For paper headlines, reuse the punchy Chinese rewrites from Step 8a (PAPERS), expanded to headline length.
- The `top_paper` frontmatter field should be in Chinese.
- Do NOT include the Radio Script section in the Chinese note.
- After saving, confirm the file path and size.

### 9b — Write Chinese radio script and generate audio

Write a Chinese-language radio script narrating the same content, as a calm, knowledgeable Chinese science podcast host. Unpack a technical term in a few spoken words the first time it appears. For every paper, say the "所以呢" out loud. 句子长短交错，别念成清单。Use this structure:

```
早上好。今天是[YYYY年M月D日，星期X]。欢迎收听 Daily Papers 早报。

先来看看昨日 AI 圈的动态。[新闻一 — 1–2句：先说发生了什么，再说为何重要，点明来源]。[新闻二 — 同上]。[新闻三 — 同上]。

下面进入今天的论文精选。

[论文一 — "首先，"] [3–4句：做了什么、关键结果、为何重要。自然提及期刊。]
[论文二 — "接下来，"] [3–4句。]
[论文三 — "第三篇，"] [3–4句。]
[论文四 — "第四篇，"] [3–4句。]
[论文五 — "最后，"] [3–4句。]

以上就是 Daily Papers [M月D日] 的早报内容。祝你今天科研顺利。
```

Do two things with this script:
1. **Save to `/tmp/daily_papers_script_cn.txt`**
2. **Append into the CN markdown note** under a new `## 播客稿` section, inside a blockquote (`> `).

Then generate speech and mix with BGM (uses CN_VOICE):

```bash
edge-tts --voice CN_VOICE --rate=-3% -f /tmp/daily_papers_script_cn.txt --write-media /tmp/daily_papers_speech_cn.mp3
```

If BGM_PATH is set and exists, mix as in Step 6b; otherwise copy speech only:
```bash
cp /tmp/daily_papers_speech_cn.mp3 "OUTPUT_DIR/NOTES_SUBDIR/audio/YYYY-MM-DD — Daily Papers CN.mp3"
```

After the command runs, confirm the file was created and add its path to the digest:
`🎧 CN Audio: NOTES_SUBDIR/audio/YYYY-MM-DD — Daily Papers CN.mp3`
