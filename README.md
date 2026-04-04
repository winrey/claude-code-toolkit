# cc-skill-review-loop

A [Claude Code](https://claude.ai/code) skill for iterative multi-round code review using isolated subagents.

## What it does

Dispatches independent reviewer subagents → screens and verifies findings → fixes confirmed issues → parallel dual-path re-review (fix diff + full review) → repeats until no Critical or Important issues remain.

Each reviewer is fully isolated: no prior review conclusions, no conversation history, no knowledge of previous fixes.

## Key features

- **Isolated subagent reviewers** — each round starts fresh, no anchoring on prior findings
- **Verification layer** — uncertain findings are independently verified before fixing
- **Parallel dual-path re-review** — after fixes, review both the fix diff AND the full codebase simultaneously
- **Flexible reviewer count** — 1-3 reviewers based on change complexity, with customizable focus areas
- **Per-round user reporting** — concise summary after every cycle
- **Configurable max rounds** — default 5, pauses and asks user when exceeded

## Installation

Copy the skill files to your Claude Code skills directory:

```bash
# Clone
git clone https://github.com/winrey/cc-skill-review-loop.git

# Copy to Claude Code skills directory
cp -r cc-skill-review-loop ~/.claude/skills/review-loop
```

Or manually copy the 4 files into `~/.claude/skills/review-loop/` (or any name you prefer).

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Main skill — flow, rules, decision logic |
| `reviewer-prompt.md` | Full reviewer subagent template |
| `fix-reviewer-prompt.md` | Fix diff reviewer subagent template |
| `verifier-prompt.md` | Issue verifier subagent template |

## Usage

Once installed, Claude Code will automatically discover the skill. You can invoke it with `/review-loop` (or whatever you named the directory), or it will be suggested when iterative code review is appropriate.

## License

MIT
