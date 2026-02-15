# OpenClaw Memory — Five-Layer Memory Protection

> Your AI has the same attention problem you do — and we fixed it the same way your brain does.

A drop-in memory system for [OpenClaw](https://openclaw.ai) agents that eliminates post-compaction amnesia through continuous observation, reflection, and reactive triggers.

## The Problem

OpenClaw's context compaction is lossy by design. It takes 200k tokens of conversation and crushes it to ~10-20k tokens of summary. Specifics get lost: exact numbers, subtle decisions, the reasoning behind choices.

## The Solution

Five layers of protection that work **with** (not against) native compaction:

1. ⏰ **Observer Cron** (every 15 min) — Continuously extracts durable facts from session transcripts
2. 🎯 **Reactive Watcher** (inotify/fswatch) — Fires within seconds during heavy conversations
3. 🛡️ **Pre-Compaction Hook** (memoryFlush) — Emergency capture right before compaction fires
4. 📝 **Session Startup** — Loads all saved memory at the start of every new session
5. 🔄 **Session Recovery** (startup check) — Catches any observations missed by the watcher during manual resets

**Cost:** ~$0.10-0.20/month using Gemini 2.5 Flash via OpenRouter.
**Context overhead:** ~4.5% of window (saves 20-30% that would be wasted re-explaining).

## Quick Start

### Option A: Let your agent install it (recommended) 🤖

Just tell your OpenClaw agent:

> Read https://raw.githubusercontent.com/ooeintellisuite/openclaw-memory/main/INSTALL-AGENT.md and follow the instructions to install OpenClaw Memory.

Your agent will read the guide, install the scripts, configure the cron jobs, and set up the pre-compaction hooks — all by itself. You just approve the steps.

### Option B: Install manually

```bash
# 1. Clone this repo
git clone https://github.com/ooeintellisuite/openclaw-memory.git
cd openclaw-memory

# 2. Run the installer
./install.sh

# 3. Done — memory system is active
```

## Architecture

```
Session JSONL files (raw transcripts)
    ↓ (every 15 min + reactive triggers + pre-compaction + session recovery)
Observer Agent (compress via LLM → observations.md)
    ↓ (when observations > 8000 words)
Reflector Agent (consolidate → compressed observations.md)
    ↓ (session startup)
Agent loads: observations.md + favorites.md + daily memory
```

See [docs/architecture.md](docs/architecture.md) for the full breakdown.

## Requirements

- OpenClaw (any recent version)
- bash, jq, curl
- An LLM API key (OpenRouter recommended — Gemini 2.5 Flash is ~$0.001/run)
- `inotify-tools` (Linux) or `fswatch` (macOS) for the reactive watcher

## Environment config (current approach)

This fork currently uses workspace `.env` for runtime configuration (`OPENROUTER_API_KEY`, paths, model).

- Example template: [`.env.example`](.env.example)
- Installer updates this block idempotently and tightens permissions to `600`.

## Security/Hardening Defaults (fork)

- Installer does **not** auto-run package manager installs with sudo.
- API key prompt is hidden during interactive install.
- `.env` file permissions are tightened (`chmod 600`) after setup.
- Observer masks common secret/token patterns before sending transcript snippets to the LLM.
- Optional dry-run mode for observer testing without external API calls:

```bash
OBSERVER_DRY_RUN=true bash ~/clawd/scripts/observer.sh
```

## Best Practice: Real-Time Task Logging (Layer 6)

The five automated layers capture **durable facts** — names, preferences, decisions. But they can miss **active task state**: what you're currently doing, what's been approved, what step is next.

For maximum resilience, teach your agent to write task state to disk **as it happens**, not just when the automated layers fire:

```markdown
# Add this to your AGENTS.md or equivalent system prompt:

When the user approves an action, when you complete a significant step, 
or when you receive important details mid-task, immediately append a brief 
note to today's memory file (memory/YYYY-MM-DD.md). Don't wait for the 
observer or the flush.
```

**Why:** If compaction fires mid-task, the automated summary keeps broad strokes but loses specifics. With task state already on disk, the post-compaction session reads the daily file and picks up exactly where things stand.

**Cost:** ~50-100 tokens per write. Negligible compared to the tokens lost re-explaining context after compaction.

Think of it as: the five layers are your safety net. Real-time logging is the discipline that means you rarely need the net.

## Inspired By

- [Mastra Observational Memory](https://mastra.ai) (94.87% on LongMemEval benchmark)
- Stanford Generative Agents (Park et al., 2023)
- MemGPT/Letta tiered memory architecture
- Cognitive science: hippocampal replay during sleep transitions

## License

MIT

## Session Configuration Tip

With the memory system running, we recommend extending your idle timeouts (the observer preserves context across resets):

```json
"session": {
  "reset": { "mode": "daily", "atHour": 5, "idleMinutes": 360 },
  "resetByType": { "group": { "mode": "daily", "atHour": 5, "idleMinutes": 180 } }
}
```

See [docs/architecture.md](docs/architecture.md) for the full rationale.
