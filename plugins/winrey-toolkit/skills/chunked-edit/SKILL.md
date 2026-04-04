---
name: chunked-edit
description: Use when editing a long file (100+ lines of changes) or when an Edit call freezes, times out, or silently fails - requires splitting large edits into multiple smaller sequential Edit calls
---

# Chunked Edit

Split large edits into small sequential batches to avoid freezing.

## The Problem

The Edit tool can freeze or timeout when applying large changes to a long file in a single call. This is a known platform bug.

**Symptoms:** Edit call hangs, connection reset, repeated silent failures, or the tool never returns.

## The Rule

**Never edit more than ~50-100 lines in a single Edit call. Each call should stay under ~1000 tokens.**

When you need to make large changes to a file:

1. **Plan edit boundaries** - identify natural split points (between functions, blocks, sections)
2. **Edit top-down** - start from the top of the file and work downward, so line numbers don't shift for upcoming edits
3. **One batch per Edit call** - each call handles one cohesive chunk (~50-100 lines)
4. **Wait for confirmation** - ensure each Edit succeeds before the next

## When This Applies

- Refactoring or rewriting a large portion of a file
- Adding/replacing 100+ lines of code in one file
- Any time a previous Edit call froze or timed out
- Generating or regenerating long files

## When This Does NOT Apply

- Small edits (under 50 lines) - just edit normally
- Creating new files with Write - use the chunked Write approach from CLAUDE.md instead

## Quick Reference

| Scenario | Approach |
|----------|----------|
| Change < 50 lines | Single Edit, no splitting needed |
| Change 50-100 lines | Single Edit is OK, split if it fails |
| Change 100-300 lines | Split into 2-4 Edit calls |
| Change 300+ lines | Split into many calls, ~50-100 lines each |
| Edit freezes | Retry with a smaller chunk |

## Red Flags

- You're about to paste 200+ lines into `new_string` - STOP, split it
- An Edit just timed out - split the remaining work into smaller chunks
- You're rewriting most of a file - consider multiple targeted edits instead of one giant replacement
