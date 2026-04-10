# Git Tips

- This repo has a second clone at `~/.claude/plugins/marketplaces/winrey-toolkit` (used by the Claude Code plugin runtime) sharing the same remote. Always make changes and commit in the main project directory only; sync the other copy via `git pull`. Never commit the same change in both directories — this creates duplicate commits with different SHAs and causes conflicts.

# Plugin Release Tips

- **Always bump `version` in [plugins/winrey-toolkit/.claude-plugin/plugin.json](plugins/winrey-toolkit/.claude-plugin/plugin.json) when adding, removing, or meaningfully modifying any skill.** Claude Code's plugin manager compares the manifest `version` against the installed cache; if the version is unchanged, `/plugins` update reports "already at the latest version" and the cache is **not** refreshed — meaning new skills silently fail to appear for users (this has happened in practice, see commit `f9b4022`). Use semver: minor bump (`0.x.0`) for new skills or breaking changes to existing ones, patch bump (`0.x.y`) for fixes/docs/tweaks.
- After bumping version and pushing, the local marketplace mirror at `~/.claude/plugins/marketplaces/winrey-toolkit` also needs `git pull` before `/plugins` update will see the new manifest. The marketplace mirror is itself a cached clone and does not auto-fetch.
