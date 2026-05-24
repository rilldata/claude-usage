# Claude Code Usage Dashboard

How much have you actually spent on Claude Code? Which projects burn the most tokens? Which tools is Claude calling for you most often? Which single prompt cost you $1.50?

This is a small [Rill](https://docs.rilldata.com) project that turns the local Claude Code session logs on your laptop into interactive dashboards. No data leaves your machine — Rill reads the JSONL transcripts directly out of `~/.claude/projects/`.

## What you'll see

Two dashboards on the same data:

**Claude Code Usage Overview** (canvas) — an at-a-glance report
- KPI cards: total cost, total messages, tokens (output / cache read), unique sessions, unique projects, avg cost per session
- Tokens and messages over time, broken down by model
- Top projects, top sessions, and tools ranked by cost
- Donut of messages by role, stacked bars of messages by model
- A per-message detail table so you can read every prompt and assistant reply with its cost

**Claude Code Cost & Usage Explorer** (explore) — slice and drill
- Filter by project, model, role, entrypoint (cli / vscode), permission mode, tool name, sub-agent flag, and more
- Group by any dimension; the time series follows your filter
- Pivot to a raw-message table and inspect single API calls

## Quick start

You need Rill installed locally. If you don't have it:

```bash
curl https://rill.sh | sh
```

Then:

```bash
rill start https://github.com/rilldata/claude-usage.git
```

That's it. Open `http://localhost:9009`. The first time you run, Rill reads every JSONL file under `~/.claude/projects/` and materializes a DuckDB table — takes a few seconds. After that, navigate to **Claude Code Usage Overview** or **Claude Code Cost & Usage Explorer** in the sidebar.

Whenever you want fresher data, save any file in the project (or click "Refresh model" in the UI) and Rill reloads the JSONL files.

## What's in the project

```
models/
  claude_messages.yaml         Reads ~/.claude/projects/*/*.jsonl, extracts every
                               user/assistant message with full text, tool calls,
                               thinking, tokens, and cost.
  claude_model_pricing.yaml    Per-model price table (input/output/cache rates).
                               Edit this if Anthropic changes prices.
metrics/
  claude_usage.yaml            ~25 dimensions and ~17 measures including
                               total_cost_usd, max_cost_per_message,
                               avg_cost_per_user_prompt.
dashboards/
  claude_usage_canvas.yaml     The overview canvas.
  claude_usage_explore.yaml    The drill-down explore.
```

## Insights you can answer

- **What did I actually spend?** Total Cost (USD) KPI, defaults to all time.
- **Which project was most expensive?** Top Projects leaderboard, sorted by cost.
- **What's my single most expensive turn ever?** Sort the per-message table by cost descending.
- **Which tool calls drive cost?** "Cost by Tool Combination" leaderboard, or filter `tool_names` in the explore.
- **How does Opus 4.7 compare to Sonnet 4.6 for my workload?** Group by `model` in the explore.
- **What was I asking when costs spiked?** Filter `role = user` and `is_user_prompt = true` in the per-message table.
- **How much of my usage is sub-agents vs main thread?** Filter `is_sidechain`.

## Make it your own

- **Update pricing** — edit `models/claude_model_pricing.yaml`. Each row is one model prefix with input/output/cache rates per million tokens.
- **Add a chart** — open `dashboards/claude_usage_canvas.yaml`, add a row with one of the available component types (bar, line, donut, heatmap, table). Save, dashboard reloads.
- **Add a measure** — `metrics/claude_usage.yaml` already has counts and cost per session/message/user-prompt; add your own SQL-aggregate measure.

## Caveats on cost accuracy

The cost figures match Anthropic's actual API charge within a few percent. Known gaps:

- **1M-context Opus** is billed at 2× standard rates above 200k input tokens, but the model name in the transcript (`claude-opus-4-7`) doesn't tell us the context size used.
- **Cache write TTL** — Anthropic charges 1.25× input for 5-minute writes and 2× for 1-hour writes. The transcript only stores the combined total, so we assume 5-minute (the Claude Code default).
- **Subscription billing** — if you're on Claude Pro/Max via OAuth, you pay the flat subscription fee, not the dollar amount shown here. This dashboard reports the *API-equivalent* cost (what the same usage would have cost on the pay-per-token plan).

## How it works (one paragraph)

Claude Code writes one JSONL file per session under `~/.claude/projects/<project-slug>/<session-id>.jsonl`, with one event per content block. The `claude_messages` model reads all of them with DuckDB's `read_json` glob, extracts message content, tokens, and metadata, joins to a small pricing lookup table on a longest-prefix match, computes cost per row, then collapses streaming-chunk duplicates by grouping on `request_id` (merging text and summing block counts, taking cost once per API call). The metrics view sits on top of the model and powers both dashboards.
