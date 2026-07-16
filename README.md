# AI Market & News Intelligence Agent

An autonomous AI agent that researches global markets, sports, and technology news every 12 hours, then delivers a structured intelligence briefing straight to Telegram — fully hands-off, running on GitHub Actions.

![Status](https://img.shields.io/badge/status-active-brightgreen)
![Python](https://img.shields.io/badge/python-3.11-blue)
![Automation](https://img.shields.io/badge/scheduler-GitHub%20Actions-2088FF)

## What it does

Twice a day, the agent:

1. Searches the web for news from the last 24 hours across three domains: global geopolitics/economics, sports, and technology/AI.
2. Reads full articles (not just search snippets) to extract concrete facts — figures, percentages, dates, named events.
3. Cross-checks its own draft against a short list of previously covered stories, so it doesn't repeat itself day to day.
4. Writes a detailed, source-cited report and delivers it to Telegram, automatically splitting long reports across multiple messages if needed.
5. Logs everything — which AI provider handled the run, every search query, every page visited, and the full final report — for later auditing.

No server, no always-on process, no manual triggering required. A GitHub Actions cron job wakes the agent up, it does its job, and shuts back down.

## Why this project

I built this to go deeper than a basic "call an LLM API" script. The interesting engineering problems weren't in the AI call itself — they were everywhere around it:

- **What happens when your only AI provider hits its free-tier quota?** → Built a fallback chain across three providers (Google Gemini, Groq, OpenRouter) with automatic failover.
- **What happens when a provider's client library silently retries for 10+ minutes on a rate limit before failing?** → Disabled internal SDK retries so the fallback logic can react in seconds, not minutes.
- **What happens when the agent's summary is technically correct but empty of real information?** → Rewrote the prompt to explicitly forbid generic filler sentences and require the agent to visit and read source articles before writing each section, with a self-check step before submitting its final answer.
- **What happens when your history file gets corrupted by an incompatible schema from an earlier version?** → Made the history loader validate and skip malformed entries instead of crashing the whole pipeline.
- **What happens when GitHub Actions is a stateless, ephemeral environment?** → Redesigned the script from a `while True` daemon into a single-run script, with state (`history.json`) committed back to the repo after each run.

## Architecture

```
GitHub Actions (cron, every 12h)
        │
        ▼
   app.py (single run)
        │
        ├── FallbackModel ── tries in order: Gemini → Groq → OpenRouter
        │                     (each wraps an OpenAI-compatible client,
        │                      internal retries disabled for fast failover)
        │
        ├── CodeAgent (smolagents) ── the actual research loop
        │     ├── web_search tool  (DuckDuckGo, last-24h filtered)
        │     └── visit_webpage tool  (reads full article content)
        │
        ├── history.json ── last 5 reports, injected into the prompt
        │                     so the agent avoids repeating itself
        │
        ├── run.log ── structured logs of every step, provider, and
        │               the full final report, for debugging after the fact
        │
        └── Telegram delivery ── auto-splits long reports into multiple
                                   messages, falls back to plain text or
                                   a .txt file attachment if formatting fails
```

After each run, `history.json` and `run.log` are committed back to the repository — since GitHub Actions runners are stateless, this is what lets the agent "remember" across runs.

## Tech stack

| Layer | Tool |
|---|---|
| Agent framework | [smolagents](https://github.com/huggingface/smolagents) (Hugging Face) |
| LLM providers | Google Gemini 2.5 Flash, Groq (Llama 3.3 70B), OpenRouter — OpenAI-compatible endpoints |
| Search | DuckDuckGo (via `ddgs`) |
| Delivery | Telegram Bot API |
| Scheduling | GitHub Actions (`schedule` cron trigger) |
| State/logging | JSON file + Python `logging`, committed back to the repo each run |

## Key design decisions

**Multi-provider fallback over a single provider.** Free-tier LLM APIs have daily quotas that are easy to exhaust during active development. Rather than architecting around one provider, `FallbackModel` wraps a list of OpenAI-compatible clients and tries each in order, logging which one ultimately served the request.

**Single-run script, not a daemon.** GitHub Actions runners are destroyed after each job and have a hard execution ceiling. An earlier version of this project used a `while True` loop with `time.sleep()` between runs — which works locally but is the wrong model for ephemeral CI infrastructure. The scheduling responsibility now belongs entirely to the GitHub Actions cron trigger; the script itself just runs once and exits.

**Prompt-level enforcement over hoping for the best.** Early versions produced technically-valid but shallow reports (e.g., restating a headline instead of reporting what was in the article). The fix wasn't a bigger model — it was making the requirements explicit and checkable: the agent must visit at least one source per topic, cite figures per claim, and self-audit its draft against those rules before submitting.

## Setup

```bash
pip install -r requirements.txt
```

Environment variables required (set as GitHub Secrets, never committed):

```
TELEGRAM_TOKEN=
TELEGRAM_CHAT_ID=
GROQ_API_KEY=
GOOGLE_API_KEY=
OPENROUTER_API_KEY=
```

The workflow runs on a cron schedule defined in `.github/workflows/schedule.yml`, and can also be triggered manually via `workflow_dispatch` from the Actions tab.

## Sample output

> **Geopolitics & Global Economy:** Silver spot price dropped over 2% on July 15, 2026, reaching $57.52/oz, driven by escalating US-Iran tensions following a US naval blockade... [full source cited]
>
> **Global Sports:** Argentina defeated England 2-1 after extra time in the FIFA World Cup 2026 semi-final at Mercedes-Benz Stadium, with Lautaro Martínez scoring the decisive goal... [full source cited]
>
> **Technology:** Anthropic researchers identified a "J-space" inside Claude models — an internal representation space containing words that never appear in the model's output but actively influence its problem-solving process... [full source cited]

*(See `/sample-output` for a full example report.)*

## License

MIT — feel free to fork and adapt.

---

*This project evolved through an iterative debugging process — from a local script with a single hardcoded API key, to a fault-tolerant, multi-provider agent running unattended in CI. The commit history reflects that journey.*
