# gsc-peec-ai-visibility

A Claude skill that bridges **Google Search Console** with **Peec.ai** to build and maintain an AI-search visibility tracking system.

## What it does

Search queries people type into Google are the same questions they ask AI engines. This skill automates the full workflow:

1. **Discover & create prompts** — pulls high-potential queries from GSC (W-questions and long-tail conversational queries), filters them by impression volume, deduplicates against existing Peec.ai prompts, and creates them in batches.
2. **Build a live dashboard** — creates a persistent Cowork artifact combining GSC impressions/clicks, SISTRIX monthly search volume, and Peec.ai AI-visibility (per model: Gemini, AI Mode, AI Overview) with colour-coded columns and live data refresh.
3. **Low-performer archiving & refresh** — identifies underperforming prompts using dual thresholds (visibility < 20% AND mention count < 110), asks for user confirmation before bulk-deleting, and replaces them with fresh GSC candidates.

---

## Prerequisites

This skill requires **three MCP connectors** to be active in your Claude environment before use.

### 1. Google Search Console MCP

Provides query-level impression and click data for your domain.

Any GSC MCP server works. Popular options:

- **[Google Search Console MCP](https://github.com/adenot/mcp-google-search-console)** (open source, OAuth)
- **Visibly AI MCP** — includes GSC passthrough alongside backlinks and keyword data ([docs](https://www.antonioblago.com/de/entwickler/mcp))

Add to your Claude Code `settings.json`:

```json
{
  "mcpServers": {
    "gsc": {
      "command": "npx",
      "args": ["-y", "mcp-google-search-console"],
      "env": {
        "GOOGLE_CLIENT_ID": "your-client-id",
        "GOOGLE_CLIENT_SECRET": "your-client-secret",
        "GOOGLE_REFRESH_TOKEN": "your-refresh-token"
      }
    }
  }
}
```

### 2. Peec.ai MCP

Provides prompt management, brand visibility reports, and AI-engine data.

Peec AI ships a hosted MCP server at `https://api.peec.ai/mcp`:

```bash
claude mcp add peec-ai --transport streamable-http https://api.peec.ai/mcp
```

Or add to `settings.json`:

```json
{
  "mcpServers": {
    "peec-ai": {
      "type": "http",
      "url": "https://api.peec.ai/mcp"
    }
  }
}
```

A Peec.ai account is required. The first call opens a browser for OAuth authorization; subsequent calls are silent. Full setup docs: [docs.peec.ai/mcp/setup](https://docs.peec.ai/mcp/setup).

### 3. SEO tool with search volume MCP

Provides monthly search volume per keyword. The skill is designed around **SISTRIX** but works with any SEO tool that exposes keyword volume via MCP.

**SISTRIX** (used in this skill's default implementation):

The SISTRIX MCP is available via the official [SISTRIX MCP connector](https://www.sistrix.com). A SISTRIX account with API access is required.

**Alternatives:** Any SEO tool that provides a `keyword_seo_traffic` or equivalent MCP tool can be used. Adapt the tool call name in Phase 2 of the skill accordingly.

---

## Installation

### Via Claude Cowork (recommended)

1. Download `gsc-peec-ai-visibility.skill` from [Releases](../../releases)
2. In Claude Cowork, open **Settings → Plugins → Install from file**
3. Select the downloaded `.skill` file

### Claude Code

```bash
# Clone into your project's skills directory
mkdir -p .claude/skills
git clone https://github.com/alexgum1/gsc2peakai.git .claude/skills/gsc-peec-ai-visibility
```

Claude Code picks up skills from `.claude/skills/` automatically. Restart or reload the session after cloning.

### Global install (all Claude Code projects)

```bash
# macOS
mkdir -p ~/Library/Application\ Support/Claude/skills
git clone https://github.com/alexgum1/gsc2peakai.git \
  ~/Library/Application\ Support/Claude/skills/gsc-peec-ai-visibility
```

### Cursor / VS Code / Windsurf

These clients don't have a unified skills model yet. Copy `SKILL.md` into your project and reference it explicitly in your AI context (e.g. via Cursor's `@Files` or a context pin).

---

## Usage

Once installed and connectors are active, trigger with natural language:

```
"Analysiere meine GSC-Daten und lege die besten W-Fragen als Peec.ai Prompts an."

"Erstelle ein Live-Dashboard für meine KI-Sichtbarkeit mit GSC, SISTRIX und Peec."

"Welche meiner Peec.ai Prompts sind Low-Performer? Archiviere sie und ersetze sie."
```

---

## File structure

```
gsc2peakai/
├── SKILL.md                          # Main skill instructions
├── references/
│   ├── filter-criteria.md            # GSC query filter logic (W-words, thresholds)
│   └── low-performer-criteria.md     # Archiving rules and thresholds
├── evals/
│   └── evals.json                    # Evaluation test cases
├── CONTRIBUTING.md
├── LICENSE
└── README.md
```

---

## Benchmark results

Evaluated across 3 test cases (14 total assertions) comparing with-skill vs. without-skill:

| Configuration | Pass rate | Avg. time |
|---------------|-----------|-----------|
| With skill | **100%** (14/14) | 55s |
| Without skill | 7% (1/14) | 23s |

The skill's most critical improvements are safety-related: user confirmation before bulk deletion, protection of freshly created GSC prompts, and correct dual-threshold low-performer detection — all of which fail without the skill.

---

## Key design decisions

- **Batched writes** — Prompts and deletions are always created/removed in groups of 5 with a 1s delay to avoid Peec.ai connection timeouts.
- **Two-stage GSC loading** — The GSC MCP's query filter is unreliable across versions; the skill uses per-prompt filtered calls with a batch fallback to avoid false zero-impression readings.
- **`unwrap()` helper** — Peec.ai responses can arrive as direct JSON or wrapped in an MCP content envelope; the skill always unwraps before parsing.
- **Visibility ratio scale** — Peec.ai returns `visibility` as a 0–1 ratio; the skill multiplies by 100 for display.
- **SEO tool is swappable** — Phase 2 (live dashboard) uses SISTRIX by default, but any MCP-accessible keyword volume tool can replace it by adapting the tool call name.

---

## Credits

Built by [Alex Gumtz](https://www.linkedin.com/in/alexandergumtz/) for the [Peec.ai MCP Challenge](https://peec.ai) · April 2026.

Not affiliated with Peec AI or SISTRIX. They make the products; this skill is an independent integration.

---

## Contributing

Found a bug, a better filter heuristic, or a new GSC gotcha? See [CONTRIBUTING.md](./CONTRIBUTING.md).

---

## License

MIT — see [LICENSE](./LICENSE).
