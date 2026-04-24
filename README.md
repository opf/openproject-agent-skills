# openproject-agent-skills

Reusable agent skills for working with OpenProject from the command line.

## Skills

### `openproject-workpackage-crud`

A `SKILL.md` that teaches Claude/Codex agents to drive `openproject-cli` for
Work Package CRUD: inspecting a Work Package and its direct children, creating
child Work Packages, and updating fields with dry-run guardrails.

- Source: [`openproject-workpackage-crud/SKILL.md`](openproject-workpackage-crud/SKILL.md)
- Scope: Work Package CRUD only — not projects, notifications, time entries,
  search, attachments, or MCP integration.
- Validation: see [`validation/`](validation/) for the RED/GREEN scenarios and
  the loophole log.

#### Install

Drop the skill directory into your agent's skill location. For Claude Code:

```sh
cp -r openproject-workpackage-crud ~/.claude/skills/
```

#### Prerequisites

The skill requires an `openproject-cli` build that exposes the JSON, dry-run,
and workflow-action contract. Probe with:

```sh
op inspect workpackage --help
op update workpackage --help
```

Look for `--json`, `--children`, `--dry-run`, `--set`, `--action`, and
`--status`. If those flags are absent, the CLI build is too old for the skill;
see the `Prerequisites` section of `SKILL.md` for the full contract.
