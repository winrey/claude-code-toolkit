# Git Tips

- This repo has a second clone at `~/.claude/plugins/marketplaces/winrey-toolkit` (used by the Claude Code plugin runtime) sharing the same remote. Always make changes and commit in the main project directory only; sync the other copy via `git pull`. Never commit the same change in both directories — this creates duplicate commits with different SHAs and causes conflicts.
