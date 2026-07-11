# daily-papers

**English** · [简体中文](README.zh-CN.md)

A [Claude Code](https://claude.com/claude-code) slash command that builds you a
personalized morning research briefing — every day, from the source, in your
field. Point it at your interests and it will:

1. **Pull** yesterday's preprints from [arXiv](https://arxiv.org) and yesterday's
   peer-reviewed papers from high-impact journals via [PubMed](https://pubmed.ncbi.nlm.nih.gov),
   plus the day's top AI-industry news.
2. **Score and rank** every candidate for relevance to *your* research profile,
   with explicit bonuses for new computational methods and top-tier venues.
3. **De-duplicate** against everything it has shown you before, so the same
   headline never resurfaces twice.
4. **Write** a curated Markdown note (Obsidian-ready) — the top 5 papers explained
   in plain language, each with a "why it matters to you" and an honest caveat.
5. **Narrate** it as a spoken-audio briefing (text-to-speech, optional background
   music), render a cover image, and mux both into a short MP4.
6. Optionally produce a **second language** edition (note + audio) in one pass.

The result is a note, an audio file, and a video sitting in your vault every
morning — the papers that actually matter to you, pre-read and de-hyped.

> The command ships configured for a spatial-omics / computational-pathology
> reader, but every field is a variable. Retarget it to NLP, robotics, systems,
> climate science, or anything with an arXiv/PubMed footprint by editing one
> block — see [Customizing](#customizing).

---

## How it works

`daily-papers` is a **prompt, not a program**. It is a single Markdown file that
instructs Claude Code to orchestrate the pipeline using its built-in tools
(`WebFetch` for the APIs, `Bash` for `edge-tts`/`ffmpeg`/`python`, and file
writes for the note). There is no server and no scraper to maintain — the model
does the fetching, filtering, judgement calls, and prose. That is what makes the
"score this for *my* interests" and "explain it to a sharp labmate" steps
possible; they are editorial judgements, not regex.

```
 arXiv API ─┐
 PubMed API ─┼─▶  fetch ─▶ score & rank ─▶ de-dup ─▶ top 5
 HN / RSS ──┘                                          │
                                                       ▼
                          Markdown note ◀── write ──┐  │
                          Audio (edge-tts + ffmpeg) ◀──┤
                          Cover image (Pillow) ───────┤
                          MP4 (ffmpeg) ───────────────┘
```

### Why these sources

- **arXiv** for same-day preprints — the leading edge, before peer review.
- **PubMed with an EDAT (electronic-deposit) window**, not a print-date filter.
  Nature-family journals index online weeks before their print date; a naive
  single-day `pdat` query misses almost all of them. The command queries a
  4-day EDAT window *and* a print-date window in parallel and pools the results.
- **Hacker News + The Verge + MIT Tech Review** for the industry-news lead, with
  cross-note de-duplication so a week-long story is reported once.

### Ranking, briefly

Each paper gets a 1–10 relevance score against your `RESEARCH_INTERESTS`, then:
`+1` if it's from a top-tier journal on your list, and `+2` if its primary
contribution is a **new method/tool/model** rather than an application of
existing methods. Ties break toward methods papers, then toward journals over
preprints. The bias is deliberate: it's built for a methods developer who wants
things they can *build on*.

---

## Requirements

- **[Claude Code](https://docs.claude.com/en/docs/claude-code)** — this command runs inside it.
- **Python 3** with [Pillow](https://pillow.readthedocs.io) for the cover image:
  `pip install pillow`
- **[edge-tts](https://github.com/rany2/edge-tts)** for text-to-speech (free, no API key):
  `pip install edge-tts`
- **[ffmpeg](https://ffmpeg.org)** for audio mixing and video muxing:
  `brew install ffmpeg` (macOS) · `sudo apt install ffmpeg` (Debian/Ubuntu) · [Windows builds](https://ffmpeg.org/download.html)
- A **CJK-capable font** only if you generate the Chinese edition. macOS and
  Windows ship one; on Linux install Noto CJK
  (`sudo apt install fonts-noto-cjk`). The cover script auto-detects a font and
  falls back gracefully.

No API keys are needed — arXiv, PubMed, Hacker News, and edge-tts are all free
and unauthenticated.

---

## Install

Claude Code loads slash commands from `~/.claude/commands` (user-global) or
`.claude/commands` (per-project). Drop the command file in either:

```bash
git clone https://github.com/liranmao/daily-papers.git
cd daily-papers

# user-global (available in every project)
mkdir -p ~/.claude/commands
cp commands/daily-papers.md ~/.claude/commands/

# …or per-project (only in the current repo)
mkdir -p .claude/commands
cp commands/daily-papers.md .claude/commands/
```

Then, **before first run**, open the copied `daily-papers.md` and edit the
**Configuration** block at the top — at minimum set `OUTPUT_DIR` to a real path.

---

## Usage

In Claude Code, run:

```
/daily-papers
```

The command walks through fetch → score → de-dup → write note → generate audio →
cover → video, printing a digest table at the end with the paths it created:

```
📄 Note:  Notes/2026-07-11 — <Headline>.md
🎧 Audio: Notes/audio/2026-07-11 — Daily Papers.mp3
🎬 Video: Notes/video/2026-07-11 — Daily Papers.mp4
```

### Run it automatically each morning

Pair it with Claude Code's scheduling so the briefing is waiting when you wake up:

```
/schedule create "0 7 * * *" /daily-papers
```

(Or wire `claude -p "/daily-papers"` into `cron` / `launchd` / Task Scheduler.)

---

## Customizing

Everything reader-specific lives in the **Configuration** block at the top of
`commands/daily-papers.md`. The most impactful fields:

- **`RESEARCH_INTERESTS`** — the topics papers are scored against. Tailor this first.
- **`ARXIV_CATEGORIES`** — which [arXiv categories](https://arxiv.org/category_taxonomy) to pull.
- **`JOURNALS`** + **`PUBMED_KEYWORDS`** — the peer-reviewed side of the funnel.
- **`OUTPUT_DIR`** / **`BGM_PATH`** — where output lands and whether to mix music.
- **`MAKE_CHINESE`** — set `no` for an English-only pipeline.

A full field-by-field reference, plus a worked example of repurposing the
command for a different field, is in [`config.example.md`](config.example.md).

---

## Output layout

```
OUTPUT_DIR/
└── NOTES_SUBDIR/
    ├── 2026-07-11 — <Headline>.md      # the daily note (papers + AI news + radio script)
    ├── audio/
    │   ├── 2026-07-11 — Daily Papers.mp3
    │   └── 2026-07-11 — Daily Papers CN.mp3   # if MAKE_CHINESE = yes
    ├── video/
    │   └── 2026-07-11 — Daily Papers.mp4
    └── CN/
        └── 2026-07-11 — <中文标题>.md          # if MAKE_CHINESE = yes
```

Notes are plain Markdown with YAML frontmatter (`date`, `journals`, `topics`,
`top_paper`), so they drop straight into an [Obsidian](https://obsidian.md)
vault and are queryable by Dataview, but they're just as readable in any text
editor.

---

## Contributing

Issues and PRs welcome — especially additional source integrations (bioRxiv,
Semantic Scholar, OpenReview), more voice/language presets, and font-detection
fixes for less-common Linux distros. Keep the command a single self-contained
Markdown file so it stays trivial to install.

## License

[MIT](LICENSE).
