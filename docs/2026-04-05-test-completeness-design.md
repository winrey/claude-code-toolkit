# Test Completeness Skill Design Spec

**Date:** 2026-04-05
**Skill name:** `test-completeness`
**Location:** `plugins/winrey-toolkit/skills/test-completeness/`

## Overview

A skill that audits test completeness for a project or PR, combining dynamic test execution (coverage data) with static scenario analysis (good case / bad case / boundary case / test layer coverage). When gaps are found, it proposes specific test additions and, upon user confirmation, generates the missing tests.

## Design Decisions

| Dimension | Decision |
|-----------|----------|
| Trigger | Manual (`/test-completeness`) + integrable by other skills |
| Scope | Default: diff (changed files only). Parameter: `full` for whole project |
| Detection | Phase 1: dynamic (run tests + coverage). Phase 2: static (Scenario Auditor subagent) |
| Remediation | Report → propose fix plan → user confirms → generate tests |
| Max iterations | 5 rounds of audit-fix cycle |
| Criteria | Good case, bad case, boundary cases, 100% coverage, unit/integration/e2e tests |

## File Structure

```
plugins/winrey-toolkit/skills/test-completeness/
├── SKILL.md              # Main skill definition (Controller)
└── auditor-prompt.md     # Scenario Auditor subagent template
```

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `scope` | `diff` | `diff` = changed files only, `full` = entire project |
| `base` | main branch | Base reference for diff mode (branch or SHA) |

## Architecture

### Two-Phase Design

**Phase 1 — Dynamic Data Collection (Controller)**
- Detect project language, framework, test runner, and coverage tool
- Determine scope (diff vs full) and collect target file list
- Run test suite and generate coverage report (line + branch)
- Parse uncovered lines and branches

**Phase 2 — Static Scenario Analysis (Scenario Auditor Subagent)**
- Receives: source code + coverage data + target files + project context
- Independently analyzes all 6 dimensions for each target file
- Outputs structured gap report

### Process Flow

```
Step 1: Project Detection
├─ Identify language/framework/test tool/coverage tool
├─ Determine scope (diff vs full)
└─ Collect target file list

Step 2: Dynamic Data Collection
├─ Run test suite
├─ Generate coverage report (line/branch)
└─ Parse uncovered lines and branches

Step 3: Dispatch Scenario Auditor (subagent)
├─ Input: source code + coverage data + target files
├─ Independent analysis:
│   ├─ Good case coverage
│   ├─ Bad case coverage (error/exception paths)
│   ├─ Boundary case coverage
│   ├─ Unit test existence
│   ├─ Integration test existence
│   └─ E2E test existence
└─ Output: structured gap report

Step 4: Summary Report
├─ Merge coverage data + auditor analysis
├─ Sort by severity (Critical → Important → Minor)
└─ Present to user

Step 5: Remediation Proposal (requires user confirmation)
├─ Propose specific test plan for each gap
├─ User selects which tests to add
└─ Generate test code

Step 6: Verification Loop (max 5 rounds)
├─ Re-run tests + coverage
├─ If 100% coverage and no Critical/Important gaps → pass, end
└─ Otherwise → re-dispatch auditor, back to Step 5
```

## Severity Definitions

| Severity | Criteria |
|----------|----------|
| **Critical** | Core business logic has zero tests; coverage at 0% for key files |
| **Important** | Missing bad case or boundary tests; missing a test layer (e.g., has integration but no unit tests); coverage < 100% |
| **Minor** | Tests exist but scenarios are not comprehensive enough; test naming/organization could improve |

## Scenario Auditor Subagent

### Input Placeholders

| Placeholder | Description |
|-------------|-------------|
| `{TARGET_FILES}` | Source file paths to audit |
| `{COVERAGE_DATA}` | Coverage report summary (uncovered lines/branches) |
| `{TEST_FILES}` | Existing test file paths |
| `{PROJECT_CONTEXT}` | Language/framework/test tool info |
| `{SCOPE}` | `diff` or `full` |
| `{DESCRIPTION}` | Change summary (diff mode only) |

### 6 Audit Dimensions

| Dimension | What to check | Pass criteria |
|-----------|---------------|---------------|
| **Good case** | Tests for normal/expected inputs | Every public function/method has at least 1 happy path test |
| **Bad case** | Tests for error/exception paths | Every error throw/catch/error return has a corresponding test |
| **Boundary cases** | Tests for edge values/extreme inputs | Null, zero, max value, empty array, empty string, type boundaries tested |
| **Unit tests** | Function/module-level isolated tests | Every function with logic has a unit test |
| **Integration tests** | Tests for cross-module interactions | Code involving external deps (DB, API, filesystem) has integration tests |
| **E2E tests** | End-to-end user flow tests | User-facing feature entry points have e2e coverage |

### Output Format

The auditor outputs a per-file structured report:

```markdown
## File: {path}

### Coverage
- Line coverage: X%
- Branch coverage: X%
- Uncovered lines: L10-L15, L30

### Good Case: {PASS/FAIL}
- ✅ functionA: has happy path test
- ❌ functionB: missing normal input test

### Bad Case: {PASS/FAIL}
- ❌ functionA: throw at L20 has no corresponding test
- ✅ functionB: error path covered

### Boundary Cases: {PASS/FAIL}
- ❌ functionA: missing empty array input test
- ❌ functionB: missing zero value boundary test

### Unit Tests: {PASS/FAIL}
### Integration Tests: {PASS/FAIL/N/A}
### E2E Tests: {PASS/FAIL/N/A}

### Issues
| # | Severity | Dimension | Description | Suggested test |
|---|----------|-----------|-------------|----------------|
| 1 | Critical | Good case | functionB has no tests at all | Test normal input returns expected result |
| 2 | Important | Boundary | functionA missing empty input test | Test empty array input throws TypeError |
```

**Key principle:** Auditor must list ALL gaps without filtering — the controller decides priority and whether to remediate. Integration and e2e dimensions allow N/A when the project lacks that infrastructure or the file has no external interactions.

## Summary Report Format

Controller merges coverage data + auditor results into a user-facing summary:

```markdown
# Test Completeness Report

## Summary
- Total files: 5 | Pass: 2 | Gaps found: 3
- Line coverage: 87% (target: 100%)
- Critical: 2 | Important: 5 | Minor: 3

## Issues by Severity

### Critical
1. [src/auth.ts] functionLogin has no tests at all
2. [src/payment.ts] line coverage at 0%

### Important
3. [src/auth.ts] validateToken missing expired token bad case
4. [src/utils.ts] parseInput missing empty string boundary test
...

### Minor
8. [src/auth.ts] hashPassword tests exist but missing different encoding inputs
...
```

## Remediation Flow

### Interactive Mode (manual trigger)

1. Present summary report to user
2. Propose remediation options:
   - (a) Fix all gaps
   - (b) Fix Critical only
   - (c) Fix Critical + Important
   - (d) Let user select specific gaps to fix
3. For selected gaps, generate test code in batches
4. Show generated tests to user for confirmation before writing
5. After writing, enter verification loop (max 5 rounds)

### Non-Interactive Mode (called by other skills)

- Skip remediation proposal
- Output report + pass/fail verdict only
- Calling skill decides next action

## Project Detection

Controller auto-detects project test infrastructure in Step 1:

| Detection target | Method |
|-----------------|--------|
| Language/framework | package.json / pyproject.toml / Cargo.toml / go.mod etc. |
| Test framework | jest / vitest / pytest / go test / cargo test etc. |
| Coverage tool | istanbul/c8 / coverage.py / go cover etc. |
| Coverage command | Infer from package.json scripts / Makefile / CI config |
| Existing test dirs | `__tests__/` / `tests/` / `*_test.go` / `*.spec.ts` etc. |
| Existing test layers | Infer unit / integration / e2e from directory structure and naming |

## Edge Cases

| Scenario | Handling |
|----------|----------|
| No test framework in project | Report status, suggest framework options, do not proceed with audit |
| Coverage tool not installed | Suggest install command, or degrade to static-analysis-only mode |
| Test suite fails to run | Report failure reason, suggest fixing tests before audit |
| No changed files in diff mode | Notify user, suggest switching to `full` mode |
| Files that don't need tests (type defs, constants) | Auditor marks as N/A, not counted as gaps |
| Integration/e2e infrastructure missing | Mark those dimensions as N/A, suggest setting up infrastructure in report |

## Integration with Other Skills

| Skill | Integration |
|-------|-------------|
| `review-loop` | Can invoke test-completeness after review-loop passes |
| `verification-before-completion` | Trigger test-completeness as verification step before claiming "done" |
| `feature-dev` | Suggest running test-completeness after feature development |
| `writing-plans` | Generated plans can include "run test-completeness" as acceptance step |
