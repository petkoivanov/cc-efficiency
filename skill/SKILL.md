---
name: "efficiency"
description: "Analyze Claude Code usage efficiency. Detects wasteful patterns (Bash overuse, search churn, unnecessary re-reads, ToolSearch overhead), estimates token waste, tracks weekly trends, and gives actionable recommendations to save tokens and move faster."
---

# Claude Code Efficiency Report

Run the efficiency analyzer to detect wasteful patterns in your Claude Code usage.

## Steps

1. Run the analyzer script:

```bash
python ~/.claude/tools/cc_efficiency.py --all
```

Or for recent data only:

```bash
python ~/.claude/tools/cc_efficiency.py --days 7
```

2. Review the findings with the user. For each HIGH or MEDIUM finding, explain:
   - What the pattern is and why it wastes tokens
   - The concrete recommendation
   - Whether a CLAUDE.md rule would help

3. If the user wants to act on recommendations, help them:
   - Add rules to CLAUDE.md (e.g., "Do not re-read files after editing")
   - Update settings.json permissions (e.g., add `mcp__*` to allow list)
   - Adjust workflow habits based on the data

## Options

- `--all` -- analyze all historical data
- `--days N` -- analyze last N days (default: 7)
- `--json` -- machine-readable output for further processing

## What It Detects

| Pattern | What It Means | Token Cost |
|---------|--------------|------------|
| Bash Overuse | Using shell for grep/cat/find instead of Grep/Read/Glob | ~400/call |
| Search Churn | 4+ Grep/Read loops -- can't find what you need | ~1,200/loop |
| ToolSearch Overhead | Loading MCP schemas at runtime | ~350/call |
| Read-After-Edit | Re-reading files that were just edited | ~800/call |
| Rapid-Fire Edits | Many tiny edits instead of batched changes | ~200/edit |
| Agent Overuse | Spawning agents for simple lookups | ~6,000/spawn |
| WebSearch for Local | Searching web when answer is in codebase | ~1,500/call |
