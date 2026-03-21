---
name: ai-digest
description: "Daily AI intelligence briefing from Twitter KOLs, AI lab blogs, tech podcasts, and HackerNews. Use when user says 'digest', 'daily', 'AI news', 'today's papers', 'what's new in AI', or on scheduled cron."
disable-model-invocation: false
---

# AI Digest

Multi-source daily AI intelligence: Twitter KOLs + AI lab blogs + tech podcasts + HackerNews.

## When to Use

- User asks about today's AI papers, news, or trending research
- Morning routine check-in on new releases
- Scheduled daily cron trigger

**When NOT to use:** Searching for a specific paper (use WebFetch or arxiv directly).

## Prerequisites

- **xreach** (`npm i -g xreach-cli`) — Twitter/X data. Requires auth: `xreach auth`
- **curl** — RSS feeds and HN API (standard on all systems)
- **Jina Reader** — free, no auth needed (`https://r.jina.ai/URL`)

## Sources

### Twitter — Tier 1: KOLs (Default, always fetched)

| Handle | Who | Focus | Count |
|--------|-----|-------|-------|
| @_akhaliq | AK | Papers, models, tools — highest signal/volume | `-n 50` |
| @karpathy | Andrej Karpathy | Deep insights, tutorials, AI commentary | `-n 20` |
| @dotey | Baoyu | Chinese AI community, translations, commentary | `-n 30` |
| @bcherny | Boris Cherny | Claude Code core dev, coding agents | `-n 20` |
| @oran_ge | OrangeAI | AI research, tools, commentary | `-n 20` |
| @trq212 | Thariq | Claude Code core dev | `-n 20` |
| @swyx | swyx | Latent Space host, AI engineering ecosystem | `-n 20` |
| @emollick | Ethan Mollick | Wharton professor, AI adoption & impact | `-n 20` |
| @drjimfan | Jim Fan | NVIDIA robotics/embodied AI research | `-n 20` |
| @simonw | Simon Willison | LLM tooling, Datasette creator, pragmatic builder | `-n 20` |
| @hardmaru | David Ha | Sakana AI CEO, creative AI research | `-n 20` |
| @ylecun | Yann LeCun | Meta/NYU, AI theory debates, high signal | `-n 20` |
| @cursor_ai | Cursor | AI-native code editor | `-n 15` |
| @AnthropicAI | Anthropic | Claude, safety research | `-n 15` |
| @OpenAI | OpenAI | GPT, API, product launches | `-n 15` |
| @GoogleDeepMind | Google DeepMind | Models, research papers | `-n 15` |

### Twitter — Tier 2: More Companies & Tools (on-demand via `--full`)

| Handle | Who | Focus |
|--------|-----|-------|
| @xai | xAI | Grok, compute infrastructure |
| @WindsurfAI | Windsurf | AI coding (Codeium) |
| @cognition | Cognition | Devin, autonomous coding agent |
| @replit | Replit | AI-native IDE & deployment |
| @huggingface | Hugging Face | Open-source models & datasets |
| @llama_index | LlamaIndex | RAG, AI agents for documents |

Users can also add any handle via arguments: `/ai-digest @someone`

### AI Lab Blogs

| Source | URL | Method |
|--------|-----|--------|
| DeepMind | `https://deepmind.google/blog/rss.xml` | Jina Reader |
| Anthropic | `https://www.anthropic.com/research` + `https://www.anthropic.com/news` | Jina Reader (no RSS) |
| OpenAI | `https://openai.com/blog/rss.xml` | Jina Reader |

### Tech Podcasts (RSS)

| Podcast | RSS Feed | Transcript Source | Focus |
|---------|----------|-------------------|-------|
| No Priors | `https://rss.art19.com/no-priors-ai` | YouTube subs | AI/ML/startups — top researcher interviews |
| Latent Space | `https://api.substack.com/feed/podcast/1084089.rss` | Substack post (embedded) | AI engineering deep dives |
| Dwarkesh Podcast | `https://apple.dwarkesh-podcast.workers.dev/feed.rss` | Substack post (embedded) | Long-form interviews with AI leaders |
| Training Data (Sequoia) | `https://feeds.megaphone.fm/trainingdata` | YouTube subs | AI/tech from Sequoia Capital |

**Transcript retrieval (with `--transcripts`):**

Substack-based podcasts (Latent Space, Dwarkesh) embed full transcripts in posts:
```bash
curl -s "https://r.jina.ai/POST_URL"  # transcript is inline with timestamps
```

YouTube-based podcasts (No Priors, Training Data) — use yt-dlp:
```bash
yt-dlp --write-auto-sub --sub-lang en --skip-download -o "%(title)s" "VIDEO_URL"
```

To find the YouTube video URL for a podcast episode, search YouTube:
```bash
yt-dlp --flat-playlist "ytsearch1:PODCAST_NAME EPISODE_TITLE" --print url
```

### HackerNews

Two parallel searches via HN Algolia API, filtered to last 24h:
1. `ai agent` — agent-specific stories
2. `LLM OR GPT OR Claude OR Gemini` — broader AI coverage

## Process

### 1. Fetch All Sources (PARALLEL)

Launch ALL fetches in parallel. Use separate Bash tool calls.

**Twitter — Tier 1 (always, one call per account):**
```bash
xreach tweets @_akhaliq --json -n 50
xreach tweets @karpathy --json -n 20
xreach tweets @dotey --json -n 30
xreach tweets @bcherny --json -n 20
xreach tweets @oran_ge --json -n 20
xreach tweets @trq212 --json -n 20
xreach tweets @swyx --json -n 20
xreach tweets @emollick --json -n 20
xreach tweets @drjimfan --json -n 20
xreach tweets @simonw --json -n 20
xreach tweets @hardmaru --json -n 20
xreach tweets @ylecun --json -n 20
xreach tweets @cursor_ai --json -n 15
xreach tweets @AnthropicAI --json -n 15
xreach tweets @OpenAI --json -n 15
xreach tweets @GoogleDeepMind --json -n 15
```

**Twitter — Tier 2 (only with `--full` flag):**
```bash
xreach tweets @xai --json -n 15
xreach tweets @WindsurfAI --json -n 15
xreach tweets @cognition --json -n 15
xreach tweets @replit --json -n 15
xreach tweets @huggingface --json -n 15
xreach tweets @llama_index --json -n 15
# + any extra @handles from arguments
```

**AI Lab Blogs (one call per source):**
```bash
curl -s "https://r.jina.ai/https://deepmind.google/blog/rss.xml"
curl -s "https://r.jina.ai/https://www.anthropic.com/news"
curl -s "https://r.jina.ai/https://openai.com/blog/rss.xml"
```

**Podcasts (one call per feed):**
```bash
curl -s "https://r.jina.ai/https://rss.art19.com/no-priors-ai"
curl -s "https://r.jina.ai/https://api.substack.com/feed/podcast/1084089.rss"
curl -s "https://r.jina.ai/https://apple.dwarkesh-podcast.workers.dev/feed.rss"
curl -s "https://r.jina.ai/https://feeds.megaphone.fm/trainingdata"
```

**HackerNews:**
```bash
YESTERDAY=$(python3 -c "import time; print(int(time.time()) - 86400)")
curl -s "https://hn.algolia.com/api/v1/search?query=ai+agent&tags=story&numericFilters=created_at_i%3E$YESTERDAY&hitsPerPage=20"
```
```bash
YESTERDAY=$(python3 -c "import time; print(int(time.time()) - 86400)")
curl -s "https://hn.algolia.com/api/v1/search?query=LLM+OR+GPT+OR+Claude+OR+Gemini&tags=story&numericFilters=created_at_i%3E$YESTERDAY&hitsPerPage=15"
```

### 2. Filter

**Twitter:**
- Filter to last 24h
- Include original tweets and quote tweets with commentary
- Exclude pure retweets (`isRetweet: true` without `isQuote: true`) and reply threads
- Quality: `likeCount > 100` for @_akhaliq, `> 50` for others (lower on weekends)

**Blogs:** Only posts from last 7 days. Skip non-technical posts (hiring, events).

**Podcasts:** Only episodes from last 7 days. Extract: title, date, description (200 chars), link.

**HN:** `points > 20`. Deduplicate against Twitter (same URL = merge, note both sources).

### 3. Categorize

| Section | Content |
|---------|---------|
| Papers | arxiv, OpenReview, research blog posts |
| Models & Releases | New models, checkpoints, fine-tunes, API launches |
| Tools & Demos | Libraries, frameworks, demos, open-source tools |
| AI Agents | Agent frameworks, benchmarks, real-world agent stories |
| Lab Updates | DeepMind / Anthropic / OpenAI blog highlights |
| Podcasts | New episodes from tracked shows (last 7 days) |
| HN Threads | Top HN discussions on AI/agents (with comment count) |
| Industry | Company announcements, funding, policy, safety |

### 4. Output

```markdown
# AI Digest — YYYY-MM-DD

## Top Highlights
1. [Most important — 1 sentence]
2. [Second — 1 sentence]
3. [Third — 1 sentence]

## Papers
- **[Title]** — [1-line summary] | [link] | @source | Likes: N

## Models & Releases
- **[Name]** — [what it does] | [link] | @source | Likes: N

## Tools & Demos
- **[Name]** — [what it does] | [link] | @source | Likes: N

## AI Agents
- **[Title]** — [1-line summary] | [link] | @source | Likes: N

## Lab Updates
- **[DeepMind]** [Title] — [summary] | [link]
- **[Anthropic]** [Title] — [summary] | [link]
- **[OpenAI]** [Title] — [summary] | [link]

## Podcasts (Last 7 Days)
- **[Show Name]** [Episode Title] — [guest/topic] | [link]
  > [3-sentence summary from transcript, if --transcripts]

## HN Threads
- **[Title]** — [points] pts, [comments] comments | [url] | [hn_link]

## Industry
- ...

---
Sources: Tier1-KOLs(N) [Tier2-Companies(N)] Labs(N) Podcasts(N) HN(N)
Total: N items
```

Save to `~/ai-digest/YYYY-MM-DD.md` (create directory if needed).

### 5. Relevance Check

If user has CLAUDE.md or memory files with project keywords, tag matching items with `[RELEVANT]`.

### 6. Summary to User

Print concise summary:
- Top 3 highlights with links
- Count per section
- New podcast episodes (always mention)
- Any `[RELEVANT]` items called out

## Arguments

| Argument | Effect |
|----------|--------|
| (none) | Tier 1 KOLs + blogs + podcasts + HN |
| `--full` | Also fetch Tier 2 company/product accounts |
| `--transcripts` | Fetch transcripts for new podcast episodes and append summaries |
| `@handle` | Add extra Twitter accounts (repeatable) |
| `--twitter-only` | Skip blogs, podcasts, and HN |
| `--hn-only` | Skip Twitter, blogs, and podcasts |
| `--podcasts-only` | Only check podcast feeds |
| `--no-save` | Print results, don't save to file |
| `--query "term"` | Add custom HN search query |

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Including all retweets | Only quote tweets with commentary |
| Sequential fetching | ALL fetches must be parallel |
| Missing URLs | Always extract arxiv/GitHub/HF links |
| Ignoring dedup | Same URL from Twitter + HN = one entry |
| Stale blog/podcast posts | Blogs: 7 days, Podcasts: 7 days |
| Empty section hidden | Show "No new episodes this week" |

## Scheduling

```bash
claude -p "run /ai-digest and save the report" \
  --allowedTools "Bash,Read,Write,Glob" \
  --output-format stream-json
```

Recommended: 9:00 AM local time.

## Fallback

- xreach fails: `curl -s "https://r.jina.ai/https://twitter.com/HANDLE"`
- HN API fails: `curl -s "https://r.jina.ai/https://news.ycombinator.com"`
- Podcast RSS fails: `curl -s "https://r.jina.ai/PODCAST_WEBSITE"`
