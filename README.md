# winrey-toolkit

Winrey's personal [Claude Code](https://claude.ai/code) plugin — a collection of custom skills.

## Skills

### winrey-review-loop

Iterative multi-round code review with isolated subagents.

Dispatches independent reviewer subagents → screens and verifies findings → fixes confirmed issues → parallel dual-path re-review (fix diff + full review) → repeats until no Critical or Important issues remain.

- **Isolated subagent reviewers** — each round starts fresh, no anchoring on prior findings
- **Verification layer** — uncertain findings are independently verified before fixing
- **Parallel dual-path re-review** — after fixes, review both the fix diff AND the full codebase simultaneously
- **Flexible reviewer count** — 1-3 reviewers based on change complexity, with customizable focus areas
- **Per-round user reporting** — concise summary after every cycle
- **Configurable max rounds** — default 5, pauses and asks user when exceeded

## Installation

```bash
claude plugin install /path/to/winrey-toolkit
```

Or from GitHub:

```bash
claude plugin install winrey/winrey-toolkit
```

## License

MIT
