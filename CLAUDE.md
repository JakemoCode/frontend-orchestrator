# CLAUDE.md

frontend-orchestration: a Claude Code plugin with UI requirements interviews, TDD build orchestration, and design audits for frontend projects.

## Versioning

- Bump `version` in `.claude-plugin/plugin.json` (and `marketplace.json`) on every meaningful change. Jake installs this plugin from a directory marketplace, and Claude Code re-copies the source into `~/.claude/plugins/cache/` only when the version changes. Without a bump, `claude plugin update` reports "already at the latest version" and serves stale code.
- To force a resync without a bump, reinstall at user scope:
  ```sh
  claude plugin uninstall frontend-orchestration@frontend-orchestration-local -s user -y --keep-data
  claude plugin install frontend-orchestration@frontend-orchestration-local -s user
  ```
- Before trusting an update, run `diff -rq` between the cache directory and this source.
