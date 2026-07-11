# daily-papers

[English](README.md) · **简体中文**

一个 [Claude Code](https://claude.com/claude-code) 斜杠命令，为你每天生成一份
**个性化的晨间科研简报**——每天、从源头、聚焦你所在的领域。设定好你的兴趣，它会：

1. **抓取** 昨日 [arXiv](https://arxiv.org) 预印本、通过 [PubMed](https://pubmed.ncbi.nlm.nih.gov)
   索引的高影响力期刊论文，以及当天的 AI 行业要闻。
2. **打分排序**——按你的研究画像给每篇候选论文评估相关性，并对**新计算方法**和
   **顶级期刊**给予明确加分。
3. **去重**——与此前推送过的所有内容比对，同一条头条绝不重复出现。
4. **撰写** 一份精选 Markdown 笔记（适配 Obsidian）——用大白话讲清 Top 5 论文，
   每篇都附上"对你有何意义"和一句诚实的局限提醒。
5. **配音**——生成语音简报（文本转语音，可选背景音乐）、渲染封面图，并把两者
   合成一段短视频 MP4。
6. 可选：一次生成**第二语言版本**（笔记 + 音频）。

最终，每天早上你的知识库里都会躺着一份笔记、一个音频和一段视频——真正与你相关、
已替你读过、且去除了浮夸的论文。

> 该命令默认配置面向"空间组学／计算病理"读者，但每一项都是变量。只需编辑一个配置块，
> 即可将它改造为面向 NLP、机器人、系统、气候科学，或任何在 arXiv/PubMed 上有踪迹的
> 领域——参见 [自定义](#自定义)。

---

## 工作原理

`daily-papers` 是一段**提示词，而非程序**。它是单个 Markdown 文件，指挥 Claude Code
用其内置工具编排整条流水线（`WebFetch` 调 API，`Bash` 跑 `edge-tts`/`ffmpeg`/`python`，
文件写入生成笔记）。没有服务器，也没有需要维护的爬虫——抓取、筛选、判断和文字都由模型
完成。正因如此，"按**我的**兴趣打分""讲给聪明的同门听"这类步骤才成为可能——它们是
编辑判断，不是正则表达式。

```
 arXiv API ─┐
 PubMed API ─┼─▶  抓取 ─▶ 打分排序 ─▶ 去重 ─▶ Top 5
 HN / RSS ──┘                                    │
                                                 ▼
                          Markdown 笔记 ◀── 撰写 ──┐  │
                          音频 (edge-tts + ffmpeg) ◀──┤
                          封面图 (Pillow) ───────────┤
                          MP4 (ffmpeg) ─────────────┘
```

### 为什么选这些数据源

- **arXiv**：抓取当日预印本——同行评审之前的最前沿。
- **PubMed 采用 EDAT（电子存档日期）窗口**，而非出版日过滤。Nature 系期刊常在正式
  印刷日之前数周就已在线索引；若用简单的单日 `pdat` 查询，几乎会全部错过。本命令
  并行查询 4 天 EDAT 窗口**与**出版日窗口，再合并结果。
- **Hacker News + The Verge + MIT Tech Review**：作为行业要闻的头条来源，并做跨笔记
  去重，让持续一周的大新闻只报道一次。

### 排序逻辑（简述）

每篇论文先按你的 `RESEARCH_INTERESTS` 打 1–10 的相关性分，然后：属于清单上顶级期刊
`+1`，主要贡献是**新方法／工具／模型**（而非把现有方法套用到某个问题上）`+2`。
同分时优先方法类论文，其次期刊优于预印本。这种偏好是刻意的：它服务于希望获得
**可复用、可延展成果**的方法开发者。

---

## 环境要求

- **[Claude Code](https://docs.claude.com/en/docs/claude-code)**——命令在其中运行。
- **Python 3** 及 [Pillow](https://pillow.readthedocs.io)（用于封面图）：`pip install pillow`
- **[edge-tts](https://github.com/rany2/edge-tts)**（文本转语音，免费、无需密钥）：`pip install edge-tts`
- **[ffmpeg](https://ffmpeg.org)**（音频混音与视频合成）：
  `brew install ffmpeg`（macOS）· `sudo apt install ffmpeg`（Debian/Ubuntu）· [Windows 版本](https://ffmpeg.org/download.html)
- 仅当生成中文版时才需要**支持中日韩的字体**。macOS 与 Windows 自带；Linux 上安装
  Noto CJK（`sudo apt install fonts-noto-cjk`）。封面脚本会自动探测字体并优雅回退。

无需任何 API 密钥——arXiv、PubMed、Hacker News 和 edge-tts 全部免费且无需鉴权。

---

## 安装

Claude Code 从 `~/.claude/commands`（全局）或 `.claude/commands`（项目级）加载斜杠命令。
把命令文件放到任一位置：

```bash
git clone https://github.com/liranmao/daily-papers.git
cd daily-papers

# 全局（在所有项目中可用）
mkdir -p ~/.claude/commands
cp commands/daily-papers.md ~/.claude/commands/

# ……或项目级（仅当前仓库可用）
mkdir -p .claude/commands
cp commands/daily-papers.md .claude/commands/
```

然后在**首次运行前**，打开复制后的 `daily-papers.md`，编辑顶部的**Configuration** 配置块——
至少要把 `OUTPUT_DIR` 设为一个真实路径。

---

## 使用

在 Claude Code 中运行：

```
/daily-papers
```

命令会依次完成 抓取 → 打分 → 去重 → 写笔记 → 生成音频 → 封面 → 视频，最后打印一张
摘要表，列出它生成的文件路径：

```
📄 笔记：Notes/2026-07-11 — <标题>.md
🎧 音频：Notes/audio/2026-07-11 — Daily Papers.mp3
🎬 视频：Notes/video/2026-07-11 — Daily Papers.mp4
```

### 每天早晨自动运行

配合 Claude Code 的定时功能，让简报在你醒来前就准备好：

```
/schedule create "0 7 * * *" /daily-papers
```

（或把 `claude -p "/daily-papers"` 接入 `cron` / `launchd` / 任务计划程序。）

---

## 自定义

所有与读者相关的设置都在 `commands/daily-papers.md` 顶部的 **Configuration** 配置块里。
最关键的几项：

- **`RESEARCH_INTERESTS`**——论文打分所依据的主题，优先调这一项。
- **`ARXIV_CATEGORIES`**——抓取哪些 [arXiv 分类](https://arxiv.org/category_taxonomy)。
- **`JOURNALS`** + **`PUBMED_KEYWORDS`**——漏斗中的同行评审侧。
- **`OUTPUT_DIR`** / **`BGM_PATH`**——输出位置，以及是否混入背景音乐。
- **`MAKE_CHINESE`**——设为 `no` 即为纯英文流水线。

逐项字段说明，以及把命令改造到其他领域的完整示例，见
[`config.example.md`](config.example.md)。

---

## 输出结构

```
OUTPUT_DIR/
└── NOTES_SUBDIR/
    ├── 2026-07-11 — <标题>.md            # 当日笔记（论文 + AI 要闻 + 播报稿）
    ├── audio/
    │   ├── 2026-07-11 — Daily Papers.mp3
    │   └── 2026-07-11 — Daily Papers CN.mp3   # 当 MAKE_CHINESE = yes
    ├── video/
    │   └── 2026-07-11 — Daily Papers.mp4
    └── CN/
        └── 2026-07-11 — <中文标题>.md          # 当 MAKE_CHINESE = yes
```

笔记是带 YAML frontmatter（`date`、`journals`、`topics`、`top_paper`）的纯 Markdown，
可直接放入 [Obsidian](https://obsidian.md) 知识库并被 Dataview 检索，同时在任意文本
编辑器中也一样可读。

---

## 设计说明与局限

- **它是编辑，不是索引。** 价值在于筛选和大白话改写，而非面面俱到的覆盖。它只呈现
  最相关的约 5 篇，而非全部。
- **大模型的判断会出错。** 相关性分数和一句话摘要是模型对摘要的解读。笔记刻意为每篇
  论文保留"值得留意"一栏，让它成为筛选器而非吹捧机——但引用前请自行核实。
- **日期窗口是启发式的。** 4 天 EDAT 窗口以少量重复（由去重步骤处理）换取不漏掉
  提前在线发表的论文。若你关注的期刊行为不同，可在 Step 2 调整。
- **API 有速率限制且无需鉴权。** 请友好使用；默认值（`max_results=40`、HN 前 30 条）
  都在限额之内。

---

## 参与贡献

欢迎提 Issue 和 PR——尤其是更多数据源接入（bioRxiv、Semantic Scholar、OpenReview）、
更多语音／语言预设，以及针对小众 Linux 发行版的字体探测修复。请保持命令为单一、
自包含的 Markdown 文件，以便安装始终简单。

## 许可证

[MIT](LICENSE)。
