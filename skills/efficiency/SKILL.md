---
name: "efficiency"
description: "Analyze Claude Code usage efficiency. 21 detectors for wasteful patterns (Bash overuse, search churn, redundant re-reads, edit-without-read, ToolSearch overhead, session thrashing, retry storms, vague prompts, model selection, cache efficiency, and more). Estimates token waste, tracks weekly trends, audits context overhead, and gives actionable recommendations."
---

# Claude Code Efficiency Report

Run the efficiency analyzer to detect wasteful patterns in your Claude Code usage.

## Steps

1. Run the analyzer script:

```bash
python ~/.claude/tools/cc_efficiency.py -A
```

This runs the full analysis: all historical data + deep transcript parsing + context audit.

Or for a quick recent check:

```bash
python ~/.claude/tools/cc_efficiency.py --days 7
```

2. Review the findings with the user. For each HIGH or MEDIUM finding, explain:
   - What the pattern is and why it wastes tokens (and dollars)
   - The concrete recommendation
   - Whether a CLAUDE.md rule would help

3. **Always include an "Understanding Your Costs" section** at the end of the report, regardless of finding severity. This tool's primary goal is education. Use a markdown heading and keep each topic to 2-3 sentences. Cover:
   - **Prompt caching**: How Claude Code's automatic caching works (5-min TTL, 0.1x reads vs 1.25x writes), what the user's hit rate means, what expiries cost.
   - **Model pricing**: The cost difference between Opus/Sonnet/Haiku, that output tokens are never cached and cost 5x input, what the session complexity breakdown shows.
   - **Context overhead**: If --context-audit ran, what MCP servers/skills/plugins cost per message and why unused ones still cost even when cached.
   - **How to read the dollar estimates**: These use the input token price as a baseline. Output-heavy waste (code generation, agent spawns) costs more. The real currency is tokens; dollars are for reference.

4. If the user wants to act on recommendations, help them:
   - Add rules to CLAUDE.md (e.g., "Do not re-read files after editing")
   - Update settings.json permissions (e.g., add `mcp__*` to allow list)
   - Remove unused MCP servers from .mcp.json
   - Adjust workflow habits based on the data

## Options

- `-A` -- full analysis (all time + deep + context-audit)
- `--all-time` -- analyze all historical data (also `--all`)
- `--days N` -- analyze last N days (default: 7)
- `--deep` -- parse transcripts for compaction, bloat, round-trips
- `--context-audit` -- audit MCP servers, skills, plugins loaded but unused
- `--model MODEL` -- model for dollar estimates: opus, sonnet, haiku (default: opus)
- `--json` -- machine-readable output for further processing
- `--project PATH` -- scan a specific project for .mcp.json and CLAUDE.md

## What It Detects (21 Patterns)

| # | Pattern | Token Cost |
|---|---------|------------|
| 1 | Bash Overuse | ~400/call |
| 2 | Search Churn (Grep/Read loops) | ~1,200/loop |
| 3 | ToolSearch Overhead | ~350/call |
| 4 | Read-After-Edit | ~800/call |
| 5 | Rapid-Fire Edits | ~200/edit |
| 6 | Agent Overuse | ~6,000/spawn |
| 7 | WebSearch for Local Answers | ~1,500/call |
| 8 | Redundant File Re-reads | ~800/re-read |
| 9 | Session Thrashing | ~15,000/session |
| 10 | Retry Storms | ~2,000/retry |
| 11 | Speculative Reading | ~600/read |
| 12 | Vague Prompt Penalty | ~500/explore call |
| 13 | Repeated File Discovery | ~1,000/repeat |
| 14 | Edit Without Prior Read | ~1,500/failure |
| 15 | Permission Friction | ~1,500/denial |
| 16 | Compaction Amnesia (--deep) | ~20,000/event |
| 17 | CLAUDE.md Bloat (--deep) | cache overhead |
| 18 | Subagent Overkill (--deep) | ~8,000/excess |
| 19 | Conversational Round-Trips (--deep) | ~5,000/round |
| 20 | Model Selection Inefficiency | ~2-3K/session |
| 21 | Prompt Cache Efficiency | ~11.5K/expiry |
