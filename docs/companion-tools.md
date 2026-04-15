# Companion Tools for Token Efficiency

This document reviews tools that complement cc-efficiency. Our goal is educational: help you understand the full landscape of token efficiency so you can make informed decisions.

cc-efficiency detects **behavioral waste** -- patterns where tokens were consumed but didn't produce value. The tools below operate at different layers of the efficiency stack.

## The Efficiency Stack

```
Layer 3: PREVENT waste    ->  Output compression (RTK, etc.)
Layer 2: DETECT waste     ->  Behavioral analysis (cc-efficiency)
Layer 1: TRACK spend      ->  Cost dashboards (ccusage, toktrack, etc.)
```

Each layer answers a different question:

- **Layer 1**: "How much did I spend?" (dashboards, cost trackers)
- **Layer 2**: "Where did I waste tokens, and why?" (cc-efficiency)
- **Layer 3**: "Can I get the same result with fewer tokens?" (output compression)

---

## RTK (rtk-ai/rtk)

**What it is**: A Rust CLI proxy that intercepts shell commands and compresses their output before it reaches the AI. Single binary, zero dependencies.

**Stars**: ~26.6K | **Language**: Rust | **License**: Apache 2.0

### How it works

RTK uses Claude Code's PreToolUse hook to transparently rewrite commands. When Claude runs `git status`, the hook rewrites it to `rtk git status`. RTK executes the original command, applies four compression strategies (filtering, grouping, truncation, deduplication), and returns compressed output. Claude never sees the rewrite.

### Claimed savings

| Operation | Without RTK | With RTK | Savings |
|-----------|------------|----------|---------|
| ls/tree (10x) | 2,000 | 400 | -80% |
| cat/read (20x) | 40,000 | 12,000 | -70% |
| grep/rg (8x) | 16,000 | 3,200 | -80% |
| git status (10x) | 3,000 | 600 | -80% |
| git diff (5x) | 10,000 | 2,500 | -75% |
| cargo/npm test (5x) | 25,000 | 2,500 | -90% |
| **30-min session total** | **~118,000** | **~23,900** | **-80%** |

### Platform support

| Environment | Works? | Notes |
|-------------|--------|-------|
| Claude Code (macOS/Linux) | Yes | PreToolUse hook, transparent |
| Claude Code (Windows) | Limited | No hook support; falls back to CLAUDE.md injection |
| Claude API (SDK) | No | CLI-only tool, no API/SDK integration |
| Other editors | Partial | Cursor, Copilot, Gemini CLI via platform hooks |

### Pros

- **Significant real savings**: 70-90% reduction on command output is substantial. For Bash-heavy sessions, this directly reduces the largest source of input tokens.
- **Zero behavior change required**: Works transparently after install. The AI doesn't know it's running.
- **Fast**: <10ms overhead per command. Written in Rust, single binary.
- **Broad command coverage**: 100+ commands across git, test runners, package managers, containers, AWS CLI, and more.
- **Complementary to cc-efficiency**: RTK compresses what goes in; cc-efficiency finds patterns that shouldn't happen at all. Using both covers Layers 2 and 3.

### Cons

- **Security: permission bypass (issue #260, P1-critical)**: The hook outputs `permissionDecision: "allow"`, which bypasses Claude Code's deny rules. If you've configured deny rules for dangerous commands like `git push --force`, RTK silently overrides them. As of April 2026, this is unresolved.
- **Security: shell injection risk (issue #640)**: A security audit found that `sh -c` commands accept unsanitized input, creating a potential prompt injection -> shell injection chain. Unresolved.
- **Telemetry enabled by default (issue #640, H-1)**: Sends device identifiers and command histories to external endpoints without install-time consent dialog. Users must explicitly opt out.
- **Information loss**: Output compression can remove details the AI needs:
  - GitHub issue content gets truncated (#188)
  - Output ordering changes (#187)
  - Native command flags break (#236, #163)
  - No streaming for long-running commands (#222)
- **The fundamental tradeoff**: RTK bets that compressed output is "good enough." Usually it is. But when the AI needs the full diff, the complete test output, or the exact error message -- and RTK filtered it -- the AI makes worse decisions, potentially causing more expensive retry loops than the tokens saved.
- **CLI-only**: No support for Claude API/SDK usage. Only works with AI coding assistants that execute shell commands.

### Scope: Bash output only

RTK only compresses **Bash command output** -- the text returned by shell commands (git, npm, cargo, ls, etc.). It has no effect on:

- **Read/Grep/Glob** -- Claude Code's built-in tools bypass the shell entirely
- **Edit/Write** -- output tokens (code the AI generates) are unaffected
- **Agent spawns** -- context duplication overhead is unchanged
- **Prompt cache** -- timing-based; unrelated to command output size
- **System prompt / MCP / skills** -- context overhead loaded every message

In a typical session, Bash is 15-25% of tool calls. Even with 80% compression on all Bash output, the overall session savings depend heavily on your Bash ratio. If your workflow is mostly Read/Edit/Grep (which many are), RTK's impact is limited.

### Recommendation

RTK delivers real, measurable token savings on the Bash portion of your workflow. The value scales with how Bash-heavy your sessions are.

**Install it if**: You use Claude Code on macOS/Linux, your sessions run lots of git/test/build commands (Bash >20% in your efficiency report), and you understand the security tradeoffs.

**Be cautious if**: You rely on Claude Code's deny rules for safety, run sensitive commands, or need complete unfiltered output for debugging.

**Skip it if**: You primarily use the Claude API/SDK, work on Windows without WSL, or your workflow is mostly Read/Edit/Grep (which RTK doesn't touch).

---

## Cost Tracking Tools (Layer 1)

These tools show you what you spent. They don't tell you why or how to spend less, but they provide the visibility that makes cc-efficiency's recommendations actionable.

### ccusage (~12.8K stars)
The most popular Claude Code cost tracker. Reads session JSONL files, calculates token counts and dollar costs. Tells you "you spent $X" but not "you wasted Y%."

### toktrack (82 stars)
Rust-based, ultra-fast token and cost tracker. Real-time monitoring.

### splitrail (156 stars)
Cross-platform, real-time token and cost monitor. Supports multiple CLI tools.

### claude-dashboard (356 stars)
Status line plugin with context usage, rate limits, and cost tracking.

### ClaudeUsageTracker (109 stars)
macOS menu bar app for tracking Claude Code API usage.

---

## How cc-efficiency fits

cc-efficiency is a **Layer 2 tool**: it analyzes behavioral patterns to find waste that no amount of output compression or cost tracking can fix.

RTK can't help with:
- Search churn (the AI searching for the wrong thing repeatedly)
- Model selection (using Opus for a task Haiku could handle)
- Cache efficiency (5-minute gaps expiring your prompt cache)
- Session thrashing (starting new sessions instead of continuing)
- Context bloat (unused MCP servers inflating every message)

Cost trackers can't help with:
- Understanding *why* costs are high
- Identifying which specific patterns to change
- Estimating how much each behavioral change would save

All three layers work together. The ideal setup:
1. **Track** spend with a cost dashboard (know what you're paying)
2. **Detect** waste with cc-efficiency (know where to improve)
3. **Prevent** waste with RTK (compress what can't be eliminated)

---

*Last reviewed: April 2026. Tool details may change -- check their repositories for current status, especially regarding open security issues.*
