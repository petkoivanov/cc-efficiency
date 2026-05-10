# cc-efficiency

Single-file Python analyzer that detects wasteful patterns in Claude Code usage. Installed as a plugin skill (`/efficiency`) at `~/.claude/tools/cc_efficiency.py`.

## Commands

- `python cc_efficiency.py -A` — full analysis (all-time + deep + context audit)
- `python cc_efficiency.py --days 7` — quick recent check
- `python cc_efficiency.py -A --json` — machine-readable output
- `python cc_efficiency.py -A --model sonnet` — dollar estimates for a different model
- No test suite — iterate by running the script against your own `~/.claude` data.

## Layout

- `cc_efficiency.py` — ~2,200 lines. Single file, zero dependencies, Python 3.8+ stdlib only. Contains all 22 detectors, argparse CLI, live-pricing fetch (LiteLLM, 1-day cache at `~/.claude/tools/.litellm_pricing_cache.json`), git-yield correlation, and report formatting. Use `Grep` to find detectors (`def detect_` or `FINDING_`) and `offset`/`limit` for sections rather than full rereads.
- `skills/efficiency/SKILL.md` — the slash-command entry point invoked via `/efficiency`.
- `hooks.json` — reference hook config users install into `~/.claude/settings.json` to emit tool-call events.
- `docs/` — `report-example.png`, `companion-tools.md` (RTK comparison).
- `README.md` — user-facing install + pattern list. Keep the 21-pattern table in sync with `cc_efficiency.py`.

## Data sources

- `~/.claude/.dashboard-events.jsonl` — one JSON line per tool call written by the PostToolUse hook. Fields: `type`, `tool`, `sessionId`, `timestamp`, `file`, `pattern`, `cmd`, `desc`, `prompt`, `query`.
- `~/.claude/.efficiency-events.jsonl` — optional enhanced events (errors, denials, session starts).
- `~/.claude/projects/*/*.jsonl` — full transcripts, parsed only under `--deep` and only structurally (never message content).

## Design rules

- Zero dependencies. If you're tempted to `import requests`/`pandas`/etc., stop — add to stdlib or skip the feature.
- Single file. Do not split into modules; easier to distribute and `curl`-install.
- Never read message content. Metadata (tool names, timestamps, file paths) only. The `--deep` parser inspects transcript structure, not text.
- Token-cost heuristics live in `TOKEN_COSTS` at the top of `cc_efficiency.py`. Adjust carefully — they flow into the dollar estimates shown to users.
- Model pricing is fetched live from LiteLLM's published JSON at runtime; falls back through stale cache → `MODEL_PRICING_FALLBACK` if offline. When Anthropic releases a new model, update `LITELLM_KEY_MAP` in `cc_efficiency.py` so it resolves the new id.
- Yield analysis (detector #22) reads session `cwd` from `~/.claude/.efficiency-events.jsonl` (enhanced SessionStart events written by `hooks.json`), NOT from PostToolUse events — the base `cwd` field isn't on those. If users see "Sessions with git repo: 0", they need the enhanced SessionStart hook installed.
- Yield's per-session token estimate is heuristic: `len(events) * 3,300` tokens/tool-call. Aggregate `tokens_per_commit` and the `>50K unproductive` threshold are approximate, not literal token counts. Don't quote them as exact figures.
- Output is a plain-text report by default; `--json` for machine consumption. Keep both formats stable — other tools may depend on them.

## Release workflow

- The plugin is registered via `extraKnownMarketplaces` → `github.com/petkoivanov/cc-efficiency`. Changes land by committing to `main` on that repo.
- After editing `cc_efficiency.py`, also copy it to `~/.claude/tools/` to pick up locally.
