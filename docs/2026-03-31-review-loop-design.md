# Review Loop Skill Design

A local global Claude Code skill that performs iterative multi-round code review using isolated subagents, with verification of findings and parallel review after fixes, looping until no Critical or Important issues remain.

## Motivation

Existing `requesting-code-review` is single-shot: dispatch reviewer, get feedback, done. It lacks:

- **Iteration** — no built-in loop to re-review after fixes
- **Verification** — no mechanism to challenge reviewer conclusions
- **Fix regression detection** — no review of the fix itself
- Relationship: complementary. Single quick review uses `requesting-code-review`; iterative review-until-clean uses `review-loop`.

## Overall Flow

```
Round 1:
  1. Dispatch 1~N independent reviewer subagents (fully isolated)
  2. Collect results → controller initial screening
     - Clearly valid → add to fix list
     - Clearly false positive → skip with reason
     - Uncertain → dispatch verifier subagent
  3. Report round summary to user
  4. No Important+ issues → PASS, exit
  5. Fix confirmed issues:
     - Simple → controller fixes directly
     - Complex single issue → 1 fixer subagent
     - Multiple independent issues → N parallel fixer subagents
  6. Dual-path parallel review:
     - Path A: 1 subagent reviews fix diff only
     - Path B: 1~N subagents do full review (fully isolated, from scratch)
  7. Merge and deduplicate results from both paths
  8. Report round summary to user
  9. No Important+ → PASS, exit
  10. Exceeded MAX_ROUNDS → pause, ask user
  11. Otherwise → back to step 2 with merged results
```

## Reviewer Subagent Design

### Reviewer Count Decision

The controller decides reviewer count based on change scope and complexity:

| Scenario | Suggested count | Example focus areas |
|----------|----------------|---------------------|
| Small change (<100 lines, 1-2 files) | 1 | Full coverage |
| Medium change (100-500 lines, multi-file) | 2 | A: logic/security/edge cases; B: architecture/maintainability/conventions |
| Large change (>500 lines, cross-module) | 2-3 | A: logic/security; B: architecture/patterns; C: spec compliance/test coverage |

Focus area assignment is flexible — controller adapts to code characteristics. For example, heavy database code might get a reviewer focused on SQL injection and transaction safety.

### Reviewer Prompt Template

Each reviewer receives (via `reviewer-prompt.md` template):

- **Round number** — "Round N review" (no history details)
- **Diff range** — BASE_SHA..HEAD_SHA
- **Change description** — brief summary from controller (no prior review info)
- **Focus areas** — what this reviewer should emphasize
- **Project conventions** — extracted from CLAUDE.md etc.

Required output format:

```
### Strengths
- ...

### Issues
For each issue:
- Severity: Critical / Important / Minor
- Location: file:line
- Description: what the problem is
- Rationale: why this is a problem
- Suggested fix: concrete recommendation

### Assessment
Pass / Fail + summary
```

**Completeness requirement:** Every review MUST exhaustively list ALL discovered issues — not just the top few. Reviewers must not omit issues because they seem minor or because there are "too many". List everything; the controller decides priority. This is essential for the loop to converge efficiently.

### Fix Diff Reviewer (Path A)

Uses `fix-reviewer-prompt.md` template. Similar to full reviewer but:

- Diff range limited to fix commits only
- Told "this diff represents recent fixes; focus on correctness and regressions" (no mention of what the original issues were — avoids confirmation bias)
- Focus: fix correctness, new regressions, side effects

### Isolation Principles

| Dimension | Rule |
|-----------|------|
| Prior review conclusions | Never passed to next round |
| Prior fix content | Full reviewer doesn't know; fix reviewer only sees diff range |
| Conversation history | Not inherited; each subagent starts fresh |
| Round info | Only round number ("Round N"), no history details |

## Screening and Verification

### Controller Initial Screening

For each issue from reviewers:

- **Has concrete `file:line` reference?** — no reference = suspicious
- **Does the referenced code actually have this problem?** — controller reads the file to quick-check
- **Consistent with project context?** — e.g., reviewer may not know an architectural convention
- **Fix benefit > risk?** — is the fix worth the change

### Verifier Subagent

For uncertain issues, dispatch an independent verifier (via `verifier-prompt.md` template):

- Receives: issue description + involved file paths + project conventions
- Does NOT know: which reviewer raised it, controller's leaning
- Task: read source code, independently judge whether issue is real
- Output: Confirmed / Not confirmed / Exists but no fix needed + code evidence + fix direction if confirmed

### Multi-Path Result Merging

After dual-path parallel review returns:

1. Merge issue lists from all paths
2. Deduplicate by file:line (same location, same category → merge)
3. Different severity for same issue → take the higher severity
4. Deduplicated list enters same screening → verification flow

## Loop Control and Termination

### Round Counting

- Round 1 = first full review
- Each complete cycle of "screening → fix → dual-path review" = +1 round

### Termination Conditions

- **PASS:** No Critical or Important issues remain (Minor issues listed but don't block)
- **MAX_ROUNDS exceeded (default 5):** Pause and ask user with: remaining Important+ issue list, round history summary, recommendation (continue/stop/adjust)
- **User intervention points:** architectural trade-off in fix direction; verifier contradicts reviewer and controller can't judge; fix scope exceeds current task

### Per-Round User Report

After each complete cycle, controller reports to user:

```
## Round N Review
- Issues found: X (Critical: a, Important: b, Minor: c)
- Verification: confirmed Y, excluded Z false positives
- Fixed: W
- Remaining Minor (non-blocking): list
- Status: continuing / passed / paused at limit
```

Kept concise — user can ask for details.

### Final Output

When review-loop ends:

```
## Review Loop Result
- Total rounds: N
- Status: Passed / User terminated
- Issues per round: Round 1: X, Round 2: Y, ...
- Total issues fixed: Z
- Remaining Minor issues: (list, optional fix)
```

## Invocation Interface

### Parameters (All Optional)

| Parameter | Default | Description |
|-----------|---------|-------------|
| BASE_SHA | Auto-inferred | Review start point |
| HEAD_SHA | HEAD | Review end point |
| DESCRIPTION | Auto-generated from git log/diff | Brief change description |
| PLAN_OR_REQUIREMENTS | From context if available | Spec or requirements |
| MAX_ROUNDS | 5 | Maximum review rounds before pausing |

### Auto-Inference Logic

```
Has explicit commit range in conversation context?
  → Yes: use that range
Current branch has unmerged commits vs main?
  → Yes: BASE = fork point, HEAD = HEAD (includes uncommitted changes if any)
Has only uncommitted changes (no unmerged commits)?
  → Yes: review working tree diff
None of the above?
  → Ask user if this is a project-level review
```

## File Structure

```
~/.claude/skills/review-loop/
  SKILL.md                 # Main: flow, rules, decision logic
  reviewer-prompt.md       # Full reviewer template
  fix-reviewer-prompt.md   # Fix diff reviewer template
  verifier-prompt.md       # Verifier template
```

## Integration with Existing Skills

| Skill | Relationship |
|-------|-------------|
| `requesting-code-review` | Complementary. Single quick review vs. iterative loop |
| `subagent-driven-development` | Can use `review-loop` as the review step per task |
| `receiving-code-review` | Controller's screening applies its verify-before-act principles |
| `verification-before-completion` | Can still be used after `review-loop` passes for final verification |
