# cc-efficiency

Detects wasteful patterns in your Claude Code usage, estimates token overhead, and gives actionable recommendations to save tokens and move faster.

Zero dependencies -- Python 3.8+ stdlib only.

![Example report showing findings, severity levels, and token waste estimates](docs/report-example.png)

## What it does

- **19 anti-pattern detectors** across tool usage, session behavior, prompts, and context overhead
- **Token waste estimates** with conservative per-call heuristics
- **Weekly trends** to track if you're improving
- **Context audit** of MCP servers, skills, and plugins loaded but never used
- **Deep mode** that parses transcripts for compaction amnesia, subagent overkill, and round-trip waste

## What it doesn't do

It never reads your source code or conversation content. By default it only analyzes tool call metadata (which tool, when, which session). The `--deep` flag parses transcript structure but not message content. All data stays local.

## Quick Start

### 1. Install the hook (one-time)

Add this to `~/.claude/settings.json`. The enriched hook captures tool names, file paths, and search patterns for per-file analysis:

```jsonc
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "node -e \"process.stdin.resume();let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{try{const e=JSON.parse(d);const i=e.tool_input||{};const meta={};if(i.file_path)meta.file=i.file_path;if(i.pattern)meta.pattern=i.pattern;if(i.command)meta.cmd=String(i.command).slice(0,120);if(i.prompt)meta.prompt=String(i.prompt).slice(0,80);if(i.query)meta.query=String(i.query).slice(0,80);if(i.description)meta.desc=String(i.description).slice(0,80);const line=JSON.stringify({type:e.hook_event_name,tool:e.tool_name,sessionId:e.session_id,timestamp:Date.now(),...meta})+'\\n';require('fs').appendFileSync(require('path').join(require('os').homedir(),'.claude','.dashboard-events.jsonl'),line)}catch(err){}})\""
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node -e \"process.stdin.resume();let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{try{const e=JSON.parse(d);const line=JSON.stringify({type:e.hook_event_name,tool:e.tool_name,sessionId:e.session_id,timestamp:Date.now()})+'\\n';require('fs').appendFileSync(require('path').join(require('os').homedir(),'.claude','.dashboard-events.jsonl'),line)}catch(err){}})\""
          }
        ]
      }
    ]
  }
}
```

> See [`hooks.json`](hooks.json) for the full config including optional enhanced hooks for error tracking, permission denials, and session tracking.

### 2. Run the analyzer

```bash
# Last 7 days (default)
python cc_efficiency.py

# Last 30 days
python cc_efficiency.py --days 30

# All time
python cc_efficiency.py --all

# Full report with context audit
python cc_efficiency.py --all --context-audit --project /path/to/your/project

# Deep analysis (parses transcripts -- slower)
python cc_efficiency.py --all --deep

# Machine-readable JSON
python cc_efficiency.py --all --json --context-audit
```

### 3. Read the report

```
================================================================
  CLAUDE CODE EFFICIENCY REPORT
================================================================
  Period:      All time
  Sessions:    54
  Tool calls:  10,006

FINDINGS
----------------------------------------------------------------

  1. [!!] Search Churn (Floundering)  (HIGH)
     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
     Churn loops: 162  |  Wasted calls: 1,690
     Est. token waste: ~194,400
     >>  When you can't find something in 2 searches, use the
        Explore agent or ask Claude to search more broadly.

  2. [!!] ToolSearch Overhead  (HIGH)
     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
     ToolSearch calls: 190 (1.9% of all tools)
     Est. token waste: ~66,500
     >>  Add frequently-used MCP tools to permissions.allow
        in settings.json so schemas load upfront.

  3. [! ] Bash Overuse  (MEDIUM)
     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
     Bash calls: 2,459 / 10,006 (24.6%)
     Healthy range: 10-15%
     Est. token waste: ~383,600
     >>  Use Grep instead of grep/rg. Use Read instead of
        cat/head/tail. Use Glob instead of find/ls.

  4. [! ] Edit Without Prior Read  (MEDIUM)
     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
     Edit-before-read failures: 97
     Est. token waste: ~145,500
     >>  Claude sometimes tries to edit a file it hasn't
        read, fails, then reads and retries.

  + 8 low-severity patterns (no action needed)

----------------------------------------------------------------
  TOTAL ESTIMATED TOKEN WASTE:     ~970,400
  PER SESSION AVERAGE:             ~17,677
  POTENTIAL SAVINGS:               ~35% per session
----------------------------------------------------------------
```

## What It Detects (19 Patterns)

### Core Patterns (always run)

| # | Pattern | Signal | Token Cost | Fix |
|---|---------|--------|------------|-----|
| 1 | **Bash Overuse** | Bash >20% of tool calls | ~400/call | Use Grep/Read/Glob instead |
| 2 | **Search Churn** | 4+ Grep/Read loops | ~1,200/loop | Use Explore agent; add paths to CLAUDE.md |
| 3 | **ToolSearch Overhead** | Frequent schema lookups | ~350/call | Pre-allow MCP tools in settings.json |
| 4 | **Read-After-Edit** | Read within 5s of Edit | ~800/call | Edit confirms success; skip re-read |
| 5 | **Rapid-Fire Edits** | 4+ Edits in 10s | ~200/edit | Batch related changes in one request |
| 6 | **Agent Overuse** | Agent >5% of tool calls | ~6,000/spawn | Use Grep/Read for simple lookups |
| 7 | **WebSearch for Local** | WebSearch then local search | ~1,500/call | Search codebase first |

### Tier 1 -- High Impact (always run)

| # | Pattern | Signal | Token Cost | Fix |
|---|---------|--------|------------|-----|
| 8 | **Redundant Re-reads** | Same file Read multiple times/session | ~800/re-read | Add "don't re-read" to CLAUDE.md |
| 9 | **Session Thrashing** | Many sessions with <5 tool calls | ~15,000/session | Batch work into fewer sessions |
| 10 | **Retry Storms** | Same tool fails 3+ times in 30s | ~2,000/retry | Read before Edit; check paths exist |
| 11 | **Speculative Reading** | Files Read but never edited | ~600/unused read | Grep first, then Read only matches |

### Tier 2 -- Behavioral (always run)

| # | Pattern | Signal | Token Cost | Fix |
|---|---------|--------|------------|-----|
| 12 | **Vague Prompt Penalty** | Short prompt -> 8+ exploration calls | ~500/explore call | Include file paths in prompts |
| 13 | **Repeated Discovery** | Same Grep/Glob pattern in 3+ sessions | ~1,000/repeat | Add key paths to CLAUDE.md |
| 14 | **Edit Without Read** | Edit fails -> Read -> Edit retry | ~1,500/failure | Always Read before Edit |
| 15 | **Permission Friction** | Same tool denied 3+ times | ~1,500/denial | Auto-allow in settings.json |

### Tier 3 -- Deep Analysis (`--deep` flag)

| # | Pattern | Signal | Token Cost | Fix |
|---|---------|--------|------------|-----|
| 16 | **Compaction Amnesia** | Repeated work after context compaction | ~20,000/event | Break into shorter sessions |
| 17 | **CLAUDE.md Bloat** | Large CLAUDE.md re-sent every message | cache overhead | Trim stale sections |
| 18 | **Subagent Overkill** | Agents >10% of session tools | ~8,000/excess | Direct tools for simple tasks |
| 19 | **Conversational Round-Trips** | 3+ prompts with no tool calls | ~5,000/round | Front-load requirements |

## Context Audit

The `--context-audit` flag analyzes the hidden token cost of everything loaded into your system prompt -- even when you never use it:

```
CONTEXT OVERHEAD (tokens loaded per message)
----------------------------------------------------------------
  MCP tool listings               (~242 tools)         ~  3,630
  MCP server instructions         (6 servers)          ~    900
  Skill listings                  (~93 skills)         ~  2,790
  CLAUDE.md                                            ~  6,853
  MEMORY.md                                            ~  2,035
  ----------------------------------------------------------------
  TOTAL                                                ~ 16,208

  UNUSED MCP SERVERS (0 calls, still loaded):
    playwright                ~30 tools  = ~450 tokens/msg

  ESTIMATED CONTEXT WASTE:     ~2,402 tokens/msg
  (15% of context overhead is estimated unused)
```

Cross-references configured MCP servers against actual usage, counts installed skills and plugins, measures CLAUDE.md size. Use `--project /path` to scan a specific project.

## Weekly Trends

Track your efficiency metrics week-over-week:

```
WEEKLY TRENDS
----------------------------------------------------------------
  Week        Sess   Tools   Bash%  Agent%  TSrch%
  2026-W12      22    3411   26.3%    1.8%    2.7%
  2026-W13       4    1335   21.8%    1.2%    1.3%
  2026-W14      12    2176   23.7%    1.1%    1.4%
  2026-W15       6     779   27.9%    1.5%    1.2%

  Bash trend: worsening (23.7% -> 27.9%)
```

## Enhanced Hooks (Recommended)

For the full 19-pattern analysis, add error, denial, and session tracking hooks. See [`hooks.json`](hooks.json) for the complete config. This enables:

- **PostToolUseFailure** -- retry storm detection, edit-without-read failures
- **PermissionDenied** -- permission friction analysis
- **SessionStart** -- session thrashing detection

The enriched PostToolUse hook also captures file paths and search patterns, enabling per-file redundant re-read breakdowns and repeated discovery detection.

## Claude Code Skill (Optional)

To use as a `/efficiency` slash command:

```bash
cp -r skill ~/.claude/skills/efficiency
mkdir -p ~/.claude/tools
cp cc_efficiency.py ~/.claude/tools/
```

## How It Works

1. The PostToolUse hook writes one JSON line per tool call to `~/.claude/.dashboard-events.jsonl`
2. Each line: `{"type": "PostToolUse", "tool": "Read", "sessionId": "abc-123", "timestamp": 1713000000000, "file": "/path/to/file.py"}`
3. The analyzer reads this file, groups events by session, and runs 19 pattern detectors
4. Token waste estimates use conservative per-call heuristics (see `TOKEN_COSTS` in source)
5. `--deep` mode additionally parses transcript JSONL files from `~/.claude/projects/` for structural analysis (compaction events, round-trips) without reading message content
6. All data stays local -- nothing is sent anywhere

## Comparison with Existing Tools

| Tool | Focus | Data Source |
|------|-------|-------------|
| `/insights` (built-in) | Qualitative AI analysis | Full transcripts |
| `/stats` (built-in) | Activity grid, streaks | Session metadata |
| `ccusage` (12.8K stars) | Cost tracking, token counts | Session JSONL |
| **cc-efficiency** | **Waste pattern detection + fixes** | **Tool event stream** |

`ccusage` tells you "you spent $X". `cc-efficiency` tells you "you wasted 35% on search churn and edit retries -- here's how to fix each one."

## License

MIT
