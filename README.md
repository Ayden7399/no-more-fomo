# ai-digest

Daily AI intelligence briefing for Claude Code. Aggregates Twitter KOLs, AI lab blogs, tech podcasts, and HackerNews into a single digest.

## Install

Add to your Claude Code skills directory:

```bash
# Clone into your skills folder
git clone https://github.com/freemty/ai-digest.git ~/.claude/skills/ai-digest
```

Or add it as a plugin dependency in your `settings.json`.

## Usage

```
/ai-digest                    # Full daily digest
/ai-digest --full             # Include Tier 2 company accounts
/ai-digest --transcripts      # Add podcast transcript summaries
/ai-digest --podcasts-only    # Only check podcast feeds
/ai-digest --hn-only          # Only HN stories
/ai-digest @someone           # Add extra Twitter handle
```

## Sources

### Twitter (16 default + 6 on-demand)

**Tier 1 (always):** @_akhaliq, @karpathy, @dotey, @bcherny, @oran_ge, @trq212, @swyx, @emollick, @drjimfan, @simonw, @hardmaru, @ylecun, @cursor_ai, @AnthropicAI, @OpenAI, @GoogleDeepMind

**Tier 2 (`--full`):** @xai, @WindsurfAI, @cognition, @replit, @huggingface, @llama_index

### AI Lab Blogs
- DeepMind (RSS)
- Anthropic (web scrape)
- OpenAI (RSS)

### Podcasts (with transcript support)
- No Priors (YouTube subs)
- Latent Space (Substack transcript)
- Dwarkesh Podcast (Substack transcript)
- Training Data / Sequoia (YouTube subs)

### HackerNews
- AI agent stories (Algolia API, 24h window)
- Broader AI coverage (LLM/GPT/Claude/Gemini)

## Prerequisites

- [xreach](https://github.com/nicepkg/xreach) (`npm i -g xreach-cli`) for Twitter
- `curl` for RSS/HN (standard)
- [yt-dlp](https://github.com/yt-dlp/yt-dlp) for podcast transcripts (optional)
- Jina Reader (free, no auth)

## Output

Saves daily digest to `~/ai-digest/YYYY-MM-DD.md` with sections:
- Top Highlights (top 3)
- Papers
- Models & Releases
- Tools & Demos
- AI Agents
- Lab Updates
- Podcasts (with transcript summaries when `--transcripts`)
- HN Threads
- Industry

## Scheduling

```bash
claude -p "run /ai-digest --transcripts and save the report" \
  --allowedTools "Bash,Read,Write,Glob" \
  --output-format stream-json
```

Recommended: 9:00 AM local time.

## License

MIT
