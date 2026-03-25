# Deep Layer Pipeline — 信息获取工具充分利用

**Date:** 2026-03-25
**Status:** Draft
**Scope:** SKILL.md 流程升级 + config 扩展

**Breaking change:** 此 spec 将 SKILL.md 中的 `language` 默认值从 `en` 改为 `zh`。SKILL.md 第 12 行的 "All digest output MUST be in English" 规则将被更新为"默认中文，可通过 `language: en` 配置切回英文"。Section 标题统一用中文（如 `## 播客` 替代 `## Podcasts`）。

## Problem

no-more-fomo 的 digest 流程未充分利用已安装的信息获取工具：

| 工具 | 已用能力 | 未用能力 |
|------|---------|---------|
| **youtube-transcript** | 完全未用 | 章节分割、说话者识别、缓存、批量下载、多语言 |
| **xreach** | `tweets` | `search`（全 Twitter 搜索）、`thread`（对话链） |
| **Jina Reader** | `r.jina.ai`（阅读） | `s.jina.ai`（搜索发现） |
| **yt-dlp** | 自动字幕下载 | 被 youtube-transcript 替代 |

播客摘要质量尤其不足——部分集只有标题，缺乏基于 transcript 的结构化总结。

## Solution: Two-Phase Pipeline

将 digest 生成拆为两个阶段。Phase 1 保持现有速度，Phase 2 在同一会话中继续执行深度处理。

### 执行模型

两个 Phase 在同一个 Claude Code 会话中**顺序执行**（不是后台进程）：
1. Phase 1 完成 → 立即保存 digest 文件 → 用户已可阅读
2. Claude 继续在同一会话中执行 Phase 2
3. Phase 2 内部，**Bash 层面**的网络请求可并行（多个 Bash tool call），但 AI 摘要生成是串行的

### Architecture

```
Phase 1: Quick Layer (现有流程，微调)
  并行 fetch (多个 Bash tool call):
    Twitter KOLs + Blogs + RSS + arxiv + HN + HuggingFace
  → Parse → Filter → Enrich → Categorize
  → 输出基础 digest (播客带占位符)
  → 提取当天高频话题 (传给 Phase 2)
  → 保存文件

Phase 2: Deep Layer (同一会话，继续执行)
  Step A — 并行网络请求 (多个 Bash tool call):
    ├── youtube-transcript 下载所有新 episode transcript
    ├── xreach search 搜索高频话题
    └── s.jina.ai 搜索 topics
  Step B — 串行 AI 处理:
    ├── 逐集生成播客结构化摘要 (说话者识别 + TLDR + 章节)
    ├── 分析 xreach search 结果，提取社区视角
    └── 过滤 s.jina.ai 结果，生成发现条目
  Step C — 回填更新 digest 文件
```

Phase 1 完成后立即保存文件，用户不用等 Phase 2。Phase 2 任何子任务失败不影响基础 digest。

## Phase 2 Detail: Podcast Deep

### 流程

```
Phase 1 (Quick):
  RSS fetch → 提取新 episode 列表 (title, date, link)
  → 对每集: 找到 YouTube URL
    优先: RSS <enclosure> 或 <link> 中的 youtube.com URL
    备选: yt-dlp --flat-playlist "ytsearch1:PODCAST EPISODE_TITLE" --print url
    失败: 标记该集为 "YouTube URL 未找到"，进入 fallback 链
  → digest 中写入: 标题 + 日期 + 链接 + "⏳ 深度摘要生成中..."

Phase 2 (Deep):
  Step 1: 并行下载所有 episode transcript (多个 Bash tool call)
    youtube-transcript 工具路径:
      ~/.claude/plugins/ljg-skills/.agents/skills/baoyu-youtube-transcript
    命令:
      bun ~/.claude/plugins/ljg-skills/.agents/skills/baoyu-youtube-transcript/scripts/main.ts VIDEO_URL
        --chapters --speakers
        --languages en,zh
        --output-dir ~/no-more-fomo/.cache/pods

  Step 2: 串行 AI 处理 (逐集)
    读取 youtube-transcript 生成的 .md 文件 (含 SRT 格式字幕 + 章节 + 元数据)
    → 参考工具内置的说话者识别提示模板:
      ~/.claude/plugins/ljg-skills/.agents/skills/baoyu-youtube-transcript/prompts/speaker-transcript.md
    → 识别 host vs guest + 标注说话者
    → 生成结构化摘要: TLDR + 章节要点 + 关键引用
    → 将摘要缓存到 ~/no-more-fomo/.cache/pods/{channel}/{title}/summary.md
```

### 输出格式

所有播客摘要默认中文输出。说话者名字保留英文原名，技术术语保留原文。

```markdown
## 播客 (Last 7 Days)

- **[Dwarkesh]** Terence Tao — Kepler, Newton, and Mathematical Discovery (Mar 20) | [link](URL)

  **TLDR:** Tao 认为 AI 已将假设生成成本降至接近零，验证成为新瓶颈。
  同行评审正被 AI 生成的投稿淹没。人机混合在数学领域的主导地位将
  比预期持续更久。

  **章节:**
  - *开普勒——高温采样的 LLM* — 二十年随机尝试各种轨道形状，
    最终靠第谷的精确数据破解
  - *验证是新瓶颈* — 生成想法已经很廉价，判断哪些想法重要
    需要数十年的学科文化积累
  - *数学发现的未来* — AI 不会取代数学家，但会重塑"做数学"的含义

  **关键引用:**
  > **Terence Tao:** "开普勒本质上在运行一个高温采样过程——
  > 用二十年时间尝试每一种可能的轨道形状。" [00:15:32]
  >
  > **Dwarkesh Patel:** "所以你是说，犯错的成本在正确的成本之前
  > 就先降为零了？" [00:22:10]
```

### 缓存策略

- youtube-transcript 自动生成缓存目录：`~/no-more-fomo/.cache/pods/{channel-slug}/{title-slug}/`
  - `{channel-slug}` 和 `{title-slug}` 由工具自动 kebab-case 化（空格→连字符，特殊字符去除）
  - 工具同时维护 `.index.json` 映射 videoId → 目录路径
- 同一集二次运行直接读缓存，跳过下载
- AI 摘要结果也缓存到同目录：`summary.md`
- Phase 2 先检查 `summary.md` 存在 → 直接读取，不重新生成

### Fallback 链

```
YouTube URL 找到:
  youtube-transcript → 完整结构化摘要 (TLDR + 章节 + 引用)
    ↓ 失败 (字幕不可用/IP blocked)
  yt-dlp --write-auto-sub → 纯文本 transcript → 简化摘要 (无章节/说话者)
    ↓ 失败

YouTube URL 未找到 或 上述均失败:
  Jina Reader (Substack post) → 从文章提取要点 (Latent Space, Dwarkesh)
    ↓ 失败
  保留 Phase 1 的基础条目 (标题+描述)，移除占位符
```

## Phase 2 Detail: Topic Search

### 定位

保守使用。不作为独立信源，只为已有条目补充社区讨论广度信号。

### 流程

```
Phase 1 结束时:
  扫描 digest 所有条目 → 提取高频实体:
  - 论文名 (出现 2+ 次的 arxiv paper)
  - 模型名 (被多人提及的 model release)
  - 工具名 (被多人讨论的 repo/product)
  → 得到 3-5 个高频话题

Phase 2 - Topic Search:
  对每个高频话题:
    xreach search "TOPIC_NAME" --type top -n 15 --json
    → 过滤: likeCount > 200, 排除已有 KOL 的推文 (去重)
    → 提取: 有价值的外部视角 (不同意见、补充信息、使用反馈)
    → 追加到对应条目
```

### 输出格式

```markdown
- **OpenCode** — TypeScript 开源 AI coding agent... | [github](...) | HN 673 pts
  > 社区热议: 多名开发者反馈在大型 monorepo 上表现优于 Cursor，
  > 但插件生态尚不成熟 (来自 5 条高互动推文)
```

### 限制

- 最多搜索 5 个话题
- 每个话题最多补充 2-3 条外部视角
- 搜索结果和 KOL 推文高度重复 (>80%) → 跳过该话题
- xreach search 失败 → 静默跳过

## Phase 2 Detail: Discovery Layer

### 定位

用 Jina 搜索模式发现 arxiv API + HN 之外的内容。

### 流程

```
读取 config.yaml 中的 papers.topics (默认: "AI agent", "LLM")

对每个 topic:
  curl -s "https://s.jina.ai/latest TOPIC research 2026"
  # s.jina.ai 使用 URL path 作为搜索 query，空格用 URL 编码 (%20)
  # 等效: curl -s "https://s.jina.ai/latest%20TOPIC%20research%202026"
  → 返回 5-10 条搜索结果 (title + URL + snippet)
  → 去重: 排除已在 digest 中出现的 URL
  → 过滤: 只保留技术内容
  → 每个 topic 保留 2-3 条新发现
```

### 输出格式

```markdown
## 发现 (Beyond arxiv/HN)
来自全网搜索的技术内容，未被 KOL 推文或 HN 覆盖。

- **[博客]** Scaling Laws for Agent Tool Use — Deeplearning.ai 深度分析
  agent 工具调用的 scaling behavior | [link](...)
- **[会议]** ICLR 2026 Outstanding Paper: ReasonFlux — 基于 flow matching
  的推理加速框架 | [link](...)
```

### 限制

- 最多搜索 3 个 topic
- 每个 topic 最多补充 3 条
- 去重是硬性的：和 digest 已有 URL 任何匹配都跳过
- s.jina.ai 失败 → 静默跳过，不新增 section

## File Update Mechanism

```
Phase 1 → 写入 ~/no-more-fomo/YYYY-MM-DD.md (完整基础 digest)
Phase 2 → 读取同一文件，定向更新:
  1. 播客: 找到 "⏳ 深度摘要生成中..." → 替换为结构化摘要
  2. Topic Search: 找到对应条目 → 追加社区热议备注
  3. Discovery: 在 Sources 行上方的 "---" 分隔线之前插入 "## 发现" section
     (锚点: 找到 "Sources:" 开头的行，在其上方的 "---" 前插入)
```

逐个子任务完成后立即写入文件。

Phase 2 全部完成后更新 Sources 行：

```markdown
---
Sources: Tier1-KOLs(N) [Tier2-Companies(N)] Labs(N) Podcasts(N/深度N) HN(N) HF-Trending(N) arxiv(N) 社区补充(N) 发现(N)
Total: N items
```

其中 `Podcasts(7/深度4)` 表示 7 集播客、4 集有深度摘要。`社区补充` 和 `发现` 仅在 Phase 2 产出时出现。

## Config Extension

```yaml
# ~/.no-more-fomo/config.yaml (新增部分)

podcasts:
  depth: full          # full | none
                       #   full = TLDR + 章节 + 引用 (默认)
                       #   none = 只有标题+描述 (跳过 Phase 2 播客处理)
  max_episodes: 3      # 每个播客最多处理几集 (默认 3)
  cache_dir: ~/no-more-fomo/.cache/pods

discovery:
  enabled: true        # 是否启用 s.jina.ai 发现层
  max_per_topic: 3     # 每个 topic 最多补充几条

topic_search:
  enabled: true        # 是否启用 xreach search 补充
  min_mentions: 2      # 实体被提及几次才触发搜索
  max_topics: 5        # 最多搜索几个话题

language: zh           # 默认改为 zh
```

### 默认行为（无 config 时）

| 设置 | 默认值 |
|------|--------|
| `podcasts.depth` | `full` |
| `podcasts.max_episodes` | `3` |
| `discovery.enabled` | `true` |
| `topic_search.enabled` | `true` |
| `language` | `zh` |

### 新增 CLI Flag

- `--quick`：跳过 Phase 2，只出基础 digest。可与其他 flag 组合使用（`--quick --full` = 基础 digest + Tier 2 账号，无深度处理）
- `--transcripts`：**废弃**。Phase 2 默认就做深度摘要，此 flag 变为 no-op。SKILL.md Arguments 表中保留但标注 "(deprecated, Phase 2 默认启用)"

### Flag 组合行为

| 组合 | 行为 |
|------|------|
| `--quick` | Phase 1 only |
| `--quick --full` | Phase 1 + Tier 2 accounts, no Phase 2 |
| `--podcasts-only` | Phase 1 只抓播客 RSS + Phase 2 深度摘要 |
| `--podcasts-only --quick` | Phase 1 只抓播客 RSS, 无深度摘要 |
| `--twitter-only` / `--hn-only` | 不触发 Phase 2 播客处理 (无播客数据)，但 Topic Search 和 Discovery 仍执行 |
| `--twitter-only --quick` | Phase 1 only, 只 Twitter |

### 向后兼容

所有新字段都有默认值。现有用户的 config.yaml 不需要改动。

## Prerequisites Update

SKILL.md Prerequisites 需新增：
- **baoyu-youtube-transcript** — 播客 transcript 下载（通过 `npx skills add` 或 ljg-skills plugin 安装）
- **bun** — youtube-transcript 脚本运行时

如果 youtube-transcript 未安装，Phase 2 播客处理自动降级到 yt-dlp fallback。

## SKILL.md Changes Summary

需要更新 SKILL.md 的以下部分：
1. 第 12 行：English-only 规则改为 `language` 配置控制，默认 `zh`
2. Prerequisites：新增 youtube-transcript + bun
3. Process Section 4 (Enrich)：播客部分改为 Phase 2 流程
4. Arguments 表：新增 `--quick`，`--transcripts` 标注 deprecated
5. Config format：新增 `podcasts.depth`、`discovery`、`topic_search` 字段
6. Section 标题：根据 `language` 配置决定中/英文

## Non-Goals

- Playwright/browse 集成：当前场景不需要 JS 渲染，curl + Jina 足够
- WebFetch 替代 Jina：token 成本过高，不适合批量
- xreach search 作为独立信源：噪音太大，只做补充
- 播客预处理 cron：增加架构复杂度和分发门槛
