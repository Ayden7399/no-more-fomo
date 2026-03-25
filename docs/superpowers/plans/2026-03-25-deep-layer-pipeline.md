# Deep Layer Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Upgrade SKILL.md from a single-phase digest to a two-phase pipeline, fully utilizing youtube-transcript, xreach search, and s.jina.ai.

**Architecture:** This is a prompt-only skill (no application code). All changes go into SKILL.md which is the executable specification that Claude follows. README.md and docs/zh.md get minor documentation updates.

**Tech Stack:** Markdown (SKILL.md prompt engineering), YAML (config schema)

**Spec:** `docs/superpowers/specs/2026-03-25-deep-layer-pipeline-design.md`

**Important — locating edits:** Since each task modifies SKILL.md and shifts line numbers, **do NOT use line numbers to locate edit points.** Instead, search for the anchor text specified in each step (shown in backticks). Use Edit tool's `old_string` matching.

---

### Task 1: Update Language Default and Prerequisites

**Files:**
- Modify: `SKILL.md` (top section: frontmatter through Prerequisites)

- [ ] **Step 1: Update language rule**

Find the line starting with `**IMPORTANT: All digest output MUST be in English.**` and replace:
```
**IMPORTANT: All digest output MUST be in English.** Summaries, descriptions, and commentary — everything in the saved markdown file and the summary shown to the user must be English. Even if source content is in Chinese or other languages, translate to English for the digest.
```
With:
```
**IMPORTANT: Digest output language follows the `language` config setting (default: `zh`).** When `zh`: summaries, section titles, and commentary in Chinese; speaker names and technical terms (model names, method names) stay in original form. When `en`: all output in English. Check `~/.no-more-fomo/config.yaml` for the `language` field; if absent, default to `zh`.
```

- [ ] **Step 2: Add new prerequisites**

After the line `- **Jina Reader** — free, no auth needed`, add:
```
- **baoyu-youtube-transcript** (optional) — podcast transcript download with chapters and speaker detection. Path: `~/.claude/plugins/ljg-skills/.agents/skills/baoyu-youtube-transcript`. If not installed, falls back to yt-dlp.
- **bun** (optional) — runtime for youtube-transcript scripts. Required only if youtube-transcript is installed.
- **yt-dlp** — podcast YouTube subtitles (fallback if youtube-transcript unavailable)
```

- [ ] **Step 3: Verify the changes read correctly**

Read SKILL.md lines 1-30 to confirm both edits are in place and consistent.

- [ ] **Step 4: Commit**

```bash
git add SKILL.md
git commit -m "feat: update language default to zh, add youtube-transcript prereqs"
```

---

### Task 2: Update Podcast Source Section — Transcript Retrieval

**Files:**
- Modify: `SKILL.md` (Tech Podcasts section)

**Note:** This describes tool usage and fallback chain as reference documentation. The actual trigger timing is controlled by Phase 2 (Task 3).

- [ ] **Step 1: Replace transcript retrieval instructions**

Find `**Transcript retrieval (with \`--transcripts\`):**` and replace everything from that line through the `yt-dlp --flat-playlist "ytsearch1:PODCAST_NAME EPISODE_TITLE" --print url` code block (inclusive) with:

```markdown
**Transcript retrieval (Phase 2 — automatic, no flag needed):**

Phase 2 automatically processes new podcast episodes. Primary method uses youtube-transcript:
```bash
# Step 1: Find YouTube URL for the episode
# Preferred: extract youtube.com URL from RSS <enclosure> or <link>
# Fallback: search YouTube
yt-dlp --flat-playlist "ytsearch1:PODCAST_NAME EPISODE_TITLE" --print url
```

```bash
# Step 2: Download transcript with chapters and speaker detection
bun ~/.claude/plugins/ljg-skills/.agents/skills/baoyu-youtube-transcript/scripts/main.ts VIDEO_URL \
  --chapters --speakers \
  --languages en,zh \
  --output-dir ~/no-more-fomo/.cache/pods
```

**Fallback chain** (if youtube-transcript fails or is not installed):
1. `yt-dlp --write-auto-sub --sub-lang en --skip-download` — auto-generated subtitles
2. `curl -s "https://r.jina.ai/POST_URL"` — Substack transcript (Latent Space, Dwarkesh)
3. Keep basic episode entry (title + description only)

Substack-based podcasts (Latent Space, Dwarkesh) embed full transcripts in posts and can be fetched directly via Jina Reader as a fallback.
```

- [ ] **Step 2: Verify changes**

Read SKILL.md lines 72-110 to confirm the section is coherent.

- [ ] **Step 3: Commit**

```bash
git add SKILL.md
git commit -m "feat: replace yt-dlp transcript with youtube-transcript + fallback chain"
```

---

### Task 3: Add Phase 2 Process Section

**Files:**
- Modify: `SKILL.md` — update Enrich section + insert Phase 2 after Process step 8

- [ ] **Step 1: Update Process step 4 — remove old podcast transcript handling**

Find the following block in the Enrich section (anchor: `**Podcast transcripts** — always fetch`):
```
**Podcast transcripts** — always fetch (not just with --transcripts). For each new episode:
- Substack (Latent Space, Dwarkesh): `curl -s "https://r.jina.ai/POST_URL"` — extract key points from transcript
- YouTube (No Priors, Training Data): `yt-dlp --write-auto-sub --sub-lang en --skip-download`
```

Replace with:
```
**Podcast transcripts** — basic metadata extracted in Phase 1 (title, date, description, link). Full transcript processing (TLDR, chapters, speaker quotes) happens in Phase 2 below. Phase 1 writes a placeholder: `> ⏳ 深度摘要生成中...`
```

- [ ] **Step 2: Add Phase 1 completion step**

Find `Save to \`~/no-more-fomo/YYYY-MM-DD.md\` (create directory if needed).` and insert **immediately after** that line, before `### 7. Relevance Check`:

```markdown
### 6.5. Save Phase 1 Digest & Extract Hot Topics

Save the digest file immediately: `~/no-more-fomo/YYYY-MM-DD.md`

Before proceeding to Phase 2, scan all digest entries and extract high-frequency entities:
- Paper names mentioned by 2+ different sources
- Model names mentioned by 2+ KOLs
- Tool/repo names discussed in both Twitter and HN
- Collect 3-5 hot topics (pass to Phase 2 Topic Search)

Podcast entries at this point have the placeholder `⏳ 深度摘要生成中...` — these will be filled in Phase 2.
```

- [ ] **Step 3: Add Phase 2 section**

Find `### 8. Summary to User` section. After its last line (the `- Any \`[RELEVANT]\` items called out` bullet), insert the complete Phase 2 section:

```markdown
## Phase 2: Deep Layer (same session, after Phase 1)

Phase 2 runs in the same Claude Code session immediately after Phase 1 saves the digest. Skip Phase 2 entirely if `--quick` flag is set.

### Phase 2 Step A: Parallel Network Requests

Launch ALL of the following as parallel Bash tool calls:

**Podcast transcripts** (if `podcasts.depth` is `full`, which is the default):
For each new episode found in Phase 1 (up to `max_episodes` per podcast, default 3):
```bash
# Check cache first
ls ~/no-more-fomo/.cache/pods/**/summary.md 2>/dev/null | grep "EPISODE_SLUG"
# If no cached summary, download transcript:
bun ~/.claude/plugins/ljg-skills/.agents/skills/baoyu-youtube-transcript/scripts/main.ts VIDEO_URL \
  --chapters --speakers --languages en,zh \
  --output-dir ~/no-more-fomo/.cache/pods
```

**Topic Search** (if `topic_search.enabled`, default true):
For each hot topic extracted in step 6.5 (max 5):
```bash
xreach search "TOPIC_NAME" --type top -n 15 --json
```

**Discovery** (if `discovery.enabled`, default true):
For each topic in `papers.topics` config (max 3):
```bash
curl -s "https://s.jina.ai/latest%20TOPIC%20research%202026"
```

### Phase 2 Step B: AI Processing (Serial)

**Podcast structured summaries** — for each episode with downloaded transcript:

1. Read the generated `.md` file from `~/no-more-fomo/.cache/pods/{channel}/{title}/transcript.md`
2. If `--speakers` was used, reference the speaker identification prompt at:
   `~/.claude/plugins/ljg-skills/.agents/skills/baoyu-youtube-transcript/prompts/speaker-transcript.md`
3. Identify speakers (host vs guest) from video metadata (title, channel, description)
4. Generate structured summary in the configured `language` (default `zh`):

```markdown
**TLDR:** [3 sentences: core thesis, most surprising insight, practical takeaway]

**章节:**
- *[Chapter Title]* — [1-2 sentence summary of this chapter]
- *[Chapter Title]* — [1-2 sentence summary]

**关键引用:**
> **[Speaker Name]:** "[Translated quote]" [HH:MM:SS]
> **[Speaker Name]:** "[Translated quote]" [HH:MM:SS]
```

5. Cache the summary to `~/no-more-fomo/.cache/pods/{channel}/{title}/summary.md`

Speaker names stay in original form (English). Technical terms stay in original form.
Select 2-3 quotes that are most insightful or surprising.

**Topic Search analysis** — for each xreach search result set:
- Filter: `likeCount > 200`, exclude tweets from KOLs already in the digest (deduplicate by handle)
- If >80% overlap with existing KOL tweets, skip this topic
- Extract 2-3 external perspectives (disagreements, supplementary info, user feedback)
- Format as a `> 社区热议:` blockquote appended to the matching digest entry

**Discovery filtering** — for each s.jina.ai result set:
- Deduplicate: remove any URL already in the digest
- Filter: only keep technical content (papers, blog posts, conference pages — not news aggregators or SEO)
- Keep max 3 per topic, format as:
```markdown
- **[类型]** [Title] — [1-2 sentence summary in configured language] | [link](URL)
```
Where 类型 is one of: 博客, 会议, 报告, 教程

### Phase 2 Step C: Update Digest File

Read `~/no-more-fomo/YYYY-MM-DD.md` and apply updates:

1. **Podcasts:** Find each `⏳ 深度摘要生成中...` placeholder → replace with the structured summary (TLDR + chapters + quotes). If transcript failed, replace placeholder with basic description from RSS.

2. **Topic Search:** Find matching entries by title → append `> 社区热议:` blockquote after the entry.

3. **Discovery:** Find the `---` line immediately above the `Sources:` line → insert before it:
```markdown
## 发现 (Beyond arxiv/HN)
来自全网搜索的技术内容，未被 KOL 推文或 HN 覆盖。

[discovery entries here]
```
Only add this section if there are discovery results. If empty, skip.

4. **Update Sources line** to include Phase 2 counts:
```
Sources: Tier1-KOLs(N) [Tier2-Companies(N)] Labs(N) Podcasts(N/深度N) HN(N) HF-Trending(N) arxiv(N) 社区补充(N) 发现(N)
```
`社区补充` and `发现` only appear if Phase 2 produced results for them.
```

- [ ] **Step 4: Verify the new section reads correctly**

Read the full Phase 2 section to verify it's coherent and all placeholders are filled.

- [ ] **Step 5: Commit**

```bash
git add SKILL.md
git commit -m "feat: add Phase 2 deep layer — podcast summaries, topic search, discovery"
```

---

### Task 4: Update Arguments Table and Config Schema

**Files:**
- Modify: `SKILL.md` (Arguments section + Config format section)

- [ ] **Step 1: Update Arguments table**

Find `## Arguments` section. Replace the entire table (from `| Argument | Effect |` through the last row `| \`--query "term"\` |`) with:

```markdown
## Arguments

| Argument | Effect |
|----------|--------|
| (none) | Phase 1 (all default sources) + Phase 2 (deep processing) |
| `--full` | Also fetch Tier 2 company/product accounts |
| `--quick` | Phase 1 only — skip Phase 2 deep processing |
| `--transcripts` | *(deprecated)* Phase 2 does this by default now |
| `@handle` | Add extra Twitter accounts (repeatable) |
| `--twitter-only` | Skip blogs, podcasts, and HN. Topic Search + Discovery still run. |
| `--hn-only` | Skip Twitter, blogs, and podcasts. Topic Search + Discovery still run. |
| `--podcasts-only` | Only podcast feeds + Phase 2 podcast deep processing |
| `--no-save` | Print results, don't save to file |
| `--query "term"` | Add custom HN search query |

**Flag combinations:**

| Combo | Behavior |
|-------|----------|
| `--quick` | Phase 1 only |
| `--quick --full` | Phase 1 + Tier 2, no Phase 2 |
| `--podcasts-only` | RSS + Phase 2 deep summaries |
| `--podcasts-only --quick` | RSS only, no deep summaries |
| `--twitter-only --quick` | Phase 1 Twitter only |
```

- [ ] **Step 2: Update config schema — add new fields**

Find the config YAML block in the `## User Config` section. Locate the `hn:` block ending with `- "computer vision"`. After it, before `language: en`, add:

```yaml
podcasts:
  # ... existing add/remove fields stay unchanged ...
  depth: full                   # full | none (default: full)
                                #   full = TLDR + chapters + speaker quotes
                                #   none = title + description only (skip Phase 2 podcasts)
  max_episodes: 3               # Max episodes per podcast to deep-process (default: 3)
  cache_dir: ~/no-more-fomo/.cache/pods  # Transcript cache directory

discovery:
  enabled: true                 # Enable s.jina.ai discovery layer (default: true)
  max_per_topic: 3              # Max discoveries per topic (default: 3)

topic_search:
  enabled: true                 # Enable xreach search supplementation (default: true)
  min_mentions: 2               # Entity must be mentioned N+ times to trigger search (default: 2)
  max_topics: 5                 # Max topics to search (default: 5)
```

Also change `language: en` to `language: zh` with updated comment:
```yaml
language: zh                    # zh | en — output language (default: zh)
```

- [ ] **Step 3: Verify both sections**

Read the Arguments table and Config sections to confirm they're complete and consistent.

- [ ] **Step 4: Commit**

```bash
git add SKILL.md
git commit -m "feat: add --quick flag, deprecate --transcripts, extend config schema"
```

---

### Task 5: Update Output Template, Categorize Table, and Section Titles

**Files:**
- Modify: `SKILL.md` (Categorize table + Format & Output template)

- [ ] **Step 1: Add language-conditional note and update Categorize table**

Find `### 5. Categorize`. After the heading, before the table, add:

```markdown
**Section titles follow `language` config.** The table below shows both versions:
```

Then update the Categorize table to include both zh/en titles:

```markdown
| Section (zh) | Section (en) | Content |
|--------------|-------------|---------|
| 模型与发布 | Models & Releases | New models, checkpoints, fine-tunes, API launches |
| 工具与演示 | Tools & Demos | Libraries, frameworks, demos, open-source tools |
| AI Agents | AI Agents | Agent frameworks, benchmarks, real-world agent stories |
| 实验室动态 | Lab Updates | DeepMind / Anthropic / OpenAI blog highlights |
| 播客 | Podcasts | New episodes from tracked shows (last 7 days) |
| HN 讨论 | HN Threads | Top HN discussions on AI/agents (with comment count) |
| 行业动态 | Industry | Company announcements, funding, policy, safety |
| HF 热门论文 | HF Trending Papers | HuggingFace community-upvoted papers (bottom, Part 1) |
| arxiv: [主题] | arxiv: [Topic] | Per-topic arxiv search results (bottom, Part 2) |
| 发现 | Discovery | Phase 2: s.jina.ai web search results (bottom, Part 3) |
```

- [ ] **Step 2: Update the Format & Output markdown template**

Find `### 6. Format & Output`. Before the template code block, add:

```markdown
**The template below uses `zh` section titles (default).** When `language: en`, use the English equivalents from the Categorize table above.
```

Then within the template, make these replacements (use `replace_all: false` for each, they're unique):

| Find | Replace with |
|------|-------------|
| `## Top Highlights` | `## 今日要点` |
| `## Models & Releases` | `## 模型与发布` |
| `## Tools & Demos` | `## 工具与演示` |
| `## AI Agents` | `## AI Agents` (keep as-is) |
| `## Lab Updates` | `## 实验室动态` |
| `## Podcasts (Last 7 Days)` | `## 播客 (Last 7 Days)` |
| `## HN Threads` | `## HN 讨论` |
| `## Industry` | `## 行业动态` |
| `## HF Trending Papers` | `## HF 热门论文` |

Also replace the Podcasts entry format:
```markdown
- **[Show Name]** [Episode Title] — [guest name & role] | [link](URL)
  > [3-5 sentence summary: key thesis, most surprising insight, practical takeaway]
```
With:
```markdown
- **[Show Name]** [Episode Title] — [guest name & role] | [link](URL)
  > ⏳ 深度摘要生成中...
```

- [ ] **Step 3: Add Discovery section and update Sources line in template**

Find the `---` line immediately before `Sources:` in the template. Insert before it:

```markdown
## 发现 (Beyond arxiv/HN)
来自全网搜索的技术内容，未被 KOL 推文或 HN 覆盖。

- **[类型]** [Title] — [summary in configured language] | [link](URL)

---
```

Then update the Sources line to:
```markdown
Sources: Tier1-KOLs(N) [Tier2-Companies(N)] Labs(N) Podcasts(N/深度N) HN(N) HF-Trending(N) arxiv(N) 社区补充(N) 发现(N)
Total: N items
```

- [ ] **Step 4: Verify template is complete**

Read the full Categorize + Format & Output sections to confirm all section titles are consistent.

- [ ] **Step 5: Commit**

```bash
git add SKILL.md
git commit -m "feat: update output template — zh titles, categorize table, podcast placeholder, discovery"
```

---

### Task 6: Update README.md and docs/zh.md

**Files:**
- Modify: `README.md`
- Modify: `docs/zh.md`

- [ ] **Step 1: Update README.md**

1. In the Usage section, add `--quick` flag:
```bash
/no-more-fomo --quick           # Quick digest — skip deep processing
```

2. In the "What makes this different" section, enhance the podcast bullet:
```
- Podcasts come with **structured summaries: TLDR, chapter breakdown, and speaker-attributed quotes** extracted from transcripts
```

3. In Prerequisites, add youtube-transcript:
```
- [baoyu-youtube-transcript](https://github.com/nicepkg/xreach) (optional) — enhanced podcast transcript processing
- [bun](https://bun.sh) — runtime for youtube-transcript (optional)
```

4. In the config example, add the new fields after the `hn:` block:
```yaml
discovery:
  enabled: true
  max_per_topic: 3

topic_search:
  enabled: true
```

5. Change `language: en` to `language: zh` in the config example.

- [ ] **Step 2: Update docs/zh.md**

1. Add `--quick` to usage section:
```bash
/no-more-fomo --quick           # 快速简报 — 跳过深度处理
```

2. Update the podcast bullet point:
```
- 播客附带 **结构化摘要：TLDR + 章节要点 + 说话者标注的关键引用**
```

3. Add youtube-transcript to prerequisites:
```
- [baoyu-youtube-transcript](optional) — 增强播客 transcript 处理
- [bun](https://bun.sh) — youtube-transcript 运行时（可选）
```

- [ ] **Step 3: Verify both files**

Read README.md and docs/zh.md to confirm consistency with SKILL.md.

- [ ] **Step 4: Commit**

```bash
git add README.md docs/zh.md
git commit -m "docs: update README and zh.md for Phase 2 pipeline"
```

---

### Task 7: Final Verification

**Files:**
- Read: `SKILL.md` (full file)

- [ ] **Step 1: Full read-through of SKILL.md**

Read the entire SKILL.md from top to bottom. Check for:
- No contradictions between Phase 1 and Phase 2 instructions
- Config field names consistent everywhere (e.g., `podcasts.depth` not `podcast.depth`)
- All fallback chains are complete
- Arguments table matches the flag combination table
- Language rule is consistent (default zh, configurable)
- youtube-transcript path is consistent throughout

- [ ] **Step 2: Cross-reference with spec**

Compare SKILL.md against spec (`docs/superpowers/specs/2026-03-25-deep-layer-pipeline-design.md`) to ensure nothing was missed:
- [ ] Language breaking change applied
- [ ] Prerequisites updated
- [ ] Phase 2 podcast flow with youtube-transcript
- [ ] Phase 2 topic search with xreach search
- [ ] Phase 2 discovery with s.jina.ai
- [ ] File update mechanism (placeholder → replace)
- [ ] Config schema complete
- [ ] Flag combinations documented
- [ ] Fallback chain complete
- [ ] Sources line format updated

- [ ] **Step 3: Fix any issues found**

If any inconsistencies or gaps found, fix them.

- [ ] **Step 4: Final commit (if fixes needed)**

```bash
git add SKILL.md README.md docs/zh.md
git commit -m "fix: address consistency issues from final review"
```
