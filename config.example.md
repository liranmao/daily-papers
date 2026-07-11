# Configuration reference

`daily-papers` is a single self-contained Claude Code slash command. All the
reader-specific settings live in the **Configuration** block at the top of
[`commands/daily-papers.md`](commands/daily-papers.md). To make the command
yours, open that file and edit those values in place — there is no separate
config file to load. This page documents each field.

| Field | What it does |
|---|---|
| `OUTPUT_DIR` | Absolute path to the folder where notes, audio, and video are written. Point it at your Obsidian vault, a plain folder, anywhere. |
| `NOTES_SUBDIR` | Subfolder under `OUTPUT_DIR` for the markdown notes (audio/ and video/ are nested inside it). |
| `BGM_PATH` | Optional background-music MP3 mixed under the narration at 8% volume. Leave blank to use narration only. |
| `READER_PROFILE` | One sentence describing who the digest is for. Drives tone and the "why it matters to you" lines. |
| `RESEARCH_INTERESTS` | The topics used to score each paper's relevance (1–10). This is the single most important field to tailor. |
| `ARXIV_CATEGORIES` | Two comma-groups of [arXiv categories](https://arxiv.org/category_taxonomy) to pull preprints from. |
| `JOURNALS` | High-impact journals searched on PubMed and kept after the strict journal filter. |
| `PUBMED_KEYWORDS` | Title/Abstract terms OR'd together in the PubMed query. |
| `TOPIC_TAGS` | Controlled vocabulary for the note's `topics` frontmatter and the cover image. |
| `EN_VOICE` / `CN_VOICE` | [edge-tts](https://github.com/rany2/edge-tts) voice names for the English and Chinese briefings. Run `edge-tts --list-voices` to see options. |
| `MAKE_CHINESE` | `yes` to also produce the Chinese note + audio (Step 9); `no` to skip. |

## Example: repurposing for a different field

Say you're an NLP researcher, not a biologist. You'd change roughly:

```
READER_PROFILE:    An NLP PhD student focused on retrieval, agents, and evaluation.
RESEARCH_INTERESTS:
  - Retrieval-augmented generation and long-context methods
  - LLM agents, tool use, and planning
  - Evaluation, benchmarks, and interpretability
ARXIV_CATEGORIES:  Group 1: cs.CL,cs.LG,cs.AI    Group 2: cs.IR,cs.MA,stat.ML
JOURNALS:          (leave the PubMed step lighter — most work is on arXiv)
TOPIC_TAGS:        RAG, agents, evaluation, long-context, interpretability, RLHF
MAKE_CHINESE:      no
```

Everything downstream — scoring, the note format, the audio, the cover, the
video — adapts automatically because it reads from these values.
