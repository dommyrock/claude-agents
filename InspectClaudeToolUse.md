# Inspecting Claude Code Tool Usage

Real-time (during a session)
┌────────────────┬──────────────────┬─────────────────────────────────────────────────────────┐
│     Method     │       How        │                      What it shows                      │
├────────────────┼──────────────────┼─────────────────────────────────────────────────────────┤
│ Verbose toggle │ Ctrl+O           │ Every tool call with inputs and results as they execute │
├────────────────┼──────────────────┼─────────────────────────────────────────────────────────┤
│ Verbose flag   │ claude --verbose │ Same as above, enabled from session start               │
├────────────────┼──────────────────┼─────────────────────────────────────────────────────────┤
│ Debug command  │ /debug           │ Reads session debug log, helps troubleshoot execution   │
├────────────────┼──────────────────┼─────────────────────────────────────────────────────────┤
│ Cost command   │ /cost            │ Token usage and cost breakdown for the session          │
└────────────────┴──────────────────┴─────────────────────────────────────────────────────────┘

After the fact (past sessions)

┌────────────────────┬──────────────────────────────────────────┬─────────────────────────────────────────────────────────────────┐
│       Method       │                   How                    │                          What it shows                          │
├────────────────────┼──────────────────────────────────────────┼─────────────────────────────────────────────────────────────────┤
│ Session JSONL logs │ ~/.claude/projects/<encoded-dir>/*.jsonl │ Complete audit trail — every tool call, parameters, and results │
├────────────────────┼──────────────────────────────────────────┼─────────────────────────────────────────────────────────────────┤
│ Resume a session   │ claude --resume                          │ Pick a past session to re-enter and inspect                     │
└────────────────────┴──────────────────────────────────────────┴─────────────────────────────────────────────────────────────────┘

Programmatic / CI

┌─────────────┬────────────────────────────────┬────────────────────────────────────────────────────────────┐
│   Method    │              How               │                       What it shows                        │
├─────────────┼────────────────────────────────┼────────────────────────────────────────────────────────────┤
│ JSON output │ claude -p --output-format json │ Full structured JSON with all tool invocations             │
├─────────────┼────────────────────────────────┼────────────────────────────────────────────────────────────┤
│ Monitoring  │ OpenTelemetry integration      │ Session counts, tokens, costs, tool decisions at org level │
└─────────────┴────────────────────────────────┴────────────────────────────────────────────────────────────┘

Querying session logs

## count tool usage by type in a session file

```sh
grep -o '"tool":"[^"]*"' ~/.claude/projects/<dir>/<session-id>.jsonl | sort | uniq -c
```

Session logs are retained for 30 days by default (configurable via cleanupPeriodDays in settings).

---

### Official references

- https://code.claude.com/docs/en/cli-reference.md
- https://code.claude.com/docs/en/interactive-mode.md
- https://code.claude.com/docs/en/costs.md
- https://code.claude.com/docs/en/data-usage.md
- https://code.claude.com/docs/en/monitoring-usage.md