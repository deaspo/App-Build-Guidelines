# App Build Guidelines

Rules for coding agents (Claude Code, Cowork, Codex, Copilot CLI, etc.) building apps: design-first workflow, decision escalation, pre-release web checklist, free-tier / AI-credit abuse controls, and agent token economy.

The full rule set is in [app-build-guidelines.md](app-build-guidelines.md). Section 0 explains how an agent should apply it; section 1 lists when each section applies.

## Use

Global, every project (Claude Code):

```sh
mkdir -p ~/.claude/rules
curl -fsSL -o ~/.claude/rules/app-build-guidelines.md \
  https://raw.githubusercontent.com/deaspo/App-Build-Guidelines/devel/app-build-guidelines.md
```

Per project, committed: copy the file to `.claude/rules/app-build-guidelines.md`.

Other agents: reference or paste it into `AGENTS.md`.

See "Install locations" in section 0 of the guidelines for the other options.

## Status

This repository is read-only. Issues and pull requests are not accepted. Fork it if you want to adapt the rules.

## License

[MIT](LICENSE)
