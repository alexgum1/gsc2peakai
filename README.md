# gsc-peec-ai-visibility

A Claude skill that bridges **Google Search Console** with **Peec.ai** to build and maintain an AI-search visibility tracking system.

## What it does

Search queries people type into Google are the same questions they ask AI engines. This skill automates the full workflow:

1. **Discover & create prompts** — pulls high-potential queries from GSC (W-questions and long-tail conversational queries), filters them by impression volume, deduplicates against existing Peec.ai prompts, and creates them in batches.
2. **Build a live dashboard** — creates a persistent Cowork artifact combining GSC impressions/clicks, SISTRIX monthly search volume, and Peec.ai AI-visibility (per model: Gemini, AI Mode, AI Overview) with colour-coded columns and live data refresh.
3. **Low-performer archiving & refresh** — identifies underperforming prompts using dual thresholds (visibility < 20% AND mention count < 110), asks for user confirmation before bulk-deleting, and replaces them with fresh GSC candidates.

## Required connectors

| Connector | Purpose |
|-----------|---------|
| [Peec.ai MCP](https://api.peec.ai/mcp) | Prompt management, brand reports, visibility data |
| Google Search Console MCP | Query data (impressions, clicks) |
| SISTRIX MCP | Monthly search volume per keyword |

All three must be connected in your Claude environment before using this skill.

## Installation

### Via Claude Cowork (recommended)

1. Download `gsc-peec-ai-visibility.skill` from [Releases](../../releases)
2. In Claude Cowork, open **Settings → Plugins → Install from file**
3. Select the downloaded `.skill` file

### Manual (Claude Code / CLI)

```bash
# Clone this repo
git clone https://github.com/agumtz/gsc2peakai.git

# The skill folder is gsc2peakai/ — point your Claude config at it
```

## Usage

Once installed, trigger the skill with natural language:

```
"Analysiere meine GSC-Daten und lege die besten W-Fragen als Peec.ai Prompts an."

"Erstelle ein Live-Dashboard für meine KI-Sichtbarkeit mit GSC, SISTRIX und Peec."

"Welche meiner Peec.ai Prompts sind Low-Performer? Archiviere sie und ersetze sie."
```

## File structure

```
gsc2peakai/
├── SKILL.md                          # Main skill instructions
├── references/
│   ├── filter-criteria.md            # GSC query filter logic (W-words, thresholds)
│   └── low-performer-criteria.md     # Archiving rules and thresholds
├── evals/
│   └── evals.json                    # Evaluation test cases
└── README.md
```

## Benchmark results

Evaluated across 3 test cases (14 total assertions) comparing with-skill vs. without-skill:

| Configuration | Pass rate | Avg. time |
|---------------|-----------|-----------|
| With skill | **100%** (14/14) | 55s |
| Without skill | 7% (1/14) | 23s |

The skill's most critical improvements are safety-related: user confirmation before bulk deletion, protection of freshly created GSC prompts, and correct dual-threshold low-performer detection — all of which fail without the skill.

## Key design decisions

- **Batched writes** — Prompts and deletions are always created/removed in groups of 5 with a 1s delay to avoid Peec.ai connection timeouts.
- **Two-stage GSC loading** — The GSC MCP's query filter is unreliable across versions; the skill uses per-prompt filtered calls with a batch fallback to avoid false zero-impression readings.
- **`unwrap()` helper** — Peec.ai responses can arrive as direct JSON or wrapped in an MCP content envelope; the skill always unwraps before parsing.
- **Visibility ratio scale** — Peec.ai returns `visibility` as a 0–1 ratio; the skill multiplies by 100 for display.

## License

MIT
