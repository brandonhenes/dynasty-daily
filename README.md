# Dynasty Daily v3

A production n8n automation system that generates daily dynasty fantasy football intelligence briefings. Aggregates news, market data, prospect scouting, and trade recommendations into a single email delivered every morning.

**Live output:** 7-section email covering NFL news with dynasty impact, prospect rankings, value trends, trade suggestions, buy/sell/hold calls, and waiver wire activity across 30 Sleeper leagues.

---

## Architecture

The system runs as 7 coordinated n8n workflows:

```
┌─────────────────────────────────────────────────┐
│              Main Orchestrator                   │
│  Triggers daily → fans out to pipelines/agents  │
│  Merges all results → builds HTML email → sends │
└──────────┬──────────┬───────────────────────────┘
           │          │
     ┌─────┴──┐  ┌────┴───────┐
     │Sub-RSS │  │Sub-Data    │
     │Pipeline│  │Pipeline    │
     └────────┘  └────────────┘
           │          │
    ┌──────┴──────────┴──────────┐
    │      4 AI Agent Workflows  │
    ├────────────────────────────┤
    │  News Agent                │
    │  Market Agent              │
    │  Prospect Agent            │
    │  Master Agent (synthesis)  │
    └────────────────────────────┘
```

**Sub-RSS Pipeline** — Fetches 18+ RSS feeds (ESPN, CBS Sports, ProFootballTalk, Pro Football Network, Dynasty Nerds, NFL Trade Rumors, FantasyPros), deduplicates, extracts full article text via Jina Reader, filters for dynasty relevance, and caches results.

**Sub-Data Pipeline** — Pulls FantasyCalc dynasty values (448+ players), Sleeper waiver wire activity across 30 leagues, prospect rankings, and signal data. Normalizes everything into Supabase tables. Computes 30-day trends, streaks, and positional rankings.

**News Agent** — Analyzes articles against player values. Identifies dynasty impact (injuries, signings, releases, depth chart changes). Outputs structured actions: BUY, SELL, HOLD, DROP, MONITOR with second-order effects.

**Market Agent** — Processes FantasyCalc value movements. Identifies biggest movers, buy-low/sell-high windows, pick market trends, and young asset opportunities. Adjusts for TEP (tight end premium) scoring.

**Prospect Agent** — Maintains a scouting big board across QB, RB, WR, TE. Integrates PFF grades, combine data, mock draft positions, and NFL player comparisons. Tracks 143+ prospects with draft round projections.

**Master Agent** — Synthesizes outputs from all three agents. Generates the lead story, trade recommendations (safe/speculative/smash tiers), and the consolidated buy/sell/hold list. Applies league-specific context (SF, TEP, 12-team).

## Data Flow

```
RSS Feeds (18 sources)
    ↓ fetch → dedupe → full-text extract → relevance filter
FantasyCalc API
    ↓ pull values → compute deltas → rank by position → cache
Sleeper API (30 leagues)
    ↓ aggregate adds/drops → streak detection → waiver scoring
Supabase (PostgreSQL)
    ↓ store snapshots → track trends → serve to agents
    ↓
4 AI Agents (parallel execution)
    ↓ structured JSON output → validation → merge
    ↓
HTML Email Builder
    ↓ responsive template → section assembly → send via SMTP
```

## Tech Stack

| Layer | Tools |
|-------|-------|
| Orchestration | n8n (self-hosted) |
| Data storage | Supabase (PostgreSQL) |
| APIs consumed | FantasyCalc, Sleeper, Jina Reader, 18 RSS feeds |
| AI/LLM | Claude via n8n AI agents |
| Code nodes | JavaScript (data transformation, scoring, validation) |
| Delivery | SMTP email, responsive HTML |

## What's in the Email

| Section | Content |
|---------|---------|
| **NFL News** | Player transactions with dynasty impact analysis, second-order effects, and action calls |
| **Prospect Big Board** | Top prospects by position with scouting profiles, PFF grades, NFL comps, and draft projections |
| **Dynasty Values** | Pick market trends, biggest movers, young asset targets with 30-day deltas |
| **One Move** | Three trade recommendations at different risk tiers with FC math and TEP adjustments |
| **Buy / Sell / Hold** | Consolidated action list with trend data and reasoning |
| **Waiver Wire** | Top adds/drops across 30 leagues with streak data and pickup rationale |
| **Sources** | Linked articles from all referenced reporting |

## Technical Details

**Workflow complexity:** 70+ nodes across 7 workflows. Fan-out/fan-in pattern for parallel agent execution. Error handling on every external API call. Caching layer prevents redundant fetches. Structured output validation on all AI responses.

**Data pipeline:** Daily snapshots stored in Supabase enable trend calculation. FantasyCalc values tracked with 30-day deltas. Sleeper waiver data aggregated across all 30 leagues with streak detection (consecutive days of adds/drops).

**Debugging:** Built a custom debugging framework to systematically diagnose and resolve 40+ production issues across multiple workflow versions. Common failure modes: RSS feed timeouts, Jina Reader rate limits, AI output format violations, merge node race conditions, Supabase write conflicts.

**JavaScript code nodes handle:**
- RSS deduplication and relevance scoring
- FantasyCalc data normalization and delta computation
- Sleeper API response parsing across 30 league endpoints
- Player name matching across data sources with fuzzy logic
- Prospect data formatting with positional grouping
- HTML email template assembly with responsive layout
- TEP value adjustments for trade math

## Repo Structure

```
dynasty-daily/
├── Dynasty Daily v3 — Main_restructured.json   # Main orchestrator
├── Sub - RSS Pipeline.json                      # RSS feed aggregation
├── Sub - Data Pipeline.json                     # FantasyCalc + Sleeper + Supabase
├── Sub - News Agent.json                        # News analysis agent
├── Sub - Market Agent.json                      # Market/value analysis agent
├── Sub - Prospect Agent.json                    # Prospect scouting agent
├── Sub - Master Agent.json                      # Synthesis + trade recommendations
└── README.md
```

## Setup

1. Import all 7 JSON files into an n8n instance
2. Create a Supabase project with the required tables (fantasycalc_daily, daily_signals, prospect_profiles, fc_daily_diffs, signal_streaks, mention_counts)
3. Configure credentials: Supabase API key, SMTP email, Jina Reader API key
4. Set the Main workflow to trigger on your preferred schedule
5. Update league IDs in the Data Pipeline to point to your Sleeper leagues

## Background

This started as a single workflow pulling one RSS feed. Over 6 months it evolved through 3 major versions, expanding from basic news aggregation to a multi-agent system with its own data layer. The current architecture processes ~450 player values, 143 prospects, 18 news sources, and 30 league endpoints every run.

Built by [Brandon Henes](https://www.linkedin.com/in/brandon-henes-71536898/) as a daily-use tool and a deep dive into workflow orchestration, API integration, data pipeline design, and AI agent architecture.
