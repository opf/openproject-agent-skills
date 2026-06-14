# openproject-agent-skills

Reusable agent skills for working with OpenProject from the command line.

## Skills

### `openproject-workpackage-crud`

A `SKILL.md` that teaches agents to drive `openproject-cli` (`op`) for Work
Package CRUD: inspecting a Work Package and its direct children, creating Work
Packages, and updating the subject, type, assignee, description, or running a
workflow action — confirming intent before each write since the CLI has no
dry-run.

- Source: [`openproject-workpackage-crud/SKILL.md`](openproject-workpackage-crud/SKILL.md)
- Scope: Work Package CRUD only — not projects, notifications, time entries,
  budgets, search, attachments, or MCP integration.
- Validation: see [`validation/`](validation/) for the RED/GREEN scenarios and
  the loophole log.

#### Install

Drop the skill directory into your agent's skill location.

Example for Claude Code:

```sh
cp -r openproject-workpackage-crud ~/.claude/skills/
```

#### Prerequisites

The skill targets the `op work-package` command surface (resource-first,
hyphenated) with the global `--format json` flag. Probe with:

```sh
op work-package inspect --help
op work-package update --help
```

Expect `inspect` to take `--format json`, `--types`, and `--open`; `update` to
take `--subject`, `--type`, `--assignee`, `--description`, `--attach`, and
`--action`; and `list` to take `--parent-id`. If you instead see the older
`op <verb> workpackage` form with `--set`/`--dry-run`/`--status`, that build
predates this skill's contract.
