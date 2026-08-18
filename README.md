# openproject-agent-skills

> [!WARNING]
> **Please read before installing or using these skills.**
>
> - **Maturity:** These skills were developed at a rapid pace — including with the assistance of AI coding tools — primarily for internal use. They have not yet had the level of scrutiny required to consider them secure or stable enough for general usage. Use them with care and at your own risk.
> - **They cause writes:** The skills drive [`openproject-cli`](https://github.com/opf/openproject-cli) against a live OpenProject instance, and the CLI has no dry-run. An agent following them can create and update real Work Packages. Point them at a test instance before trusting them with production data.
> - **Moving target:** They target a specific `op` command surface (see [Prerequisites](#prerequisites)); a different CLI version can silently change the meaning of a command.
> - **Support:** This is unsupported software with no official technical support from OpenProject GmbH.

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

## Installation

Install with the [`skills`](https://github.com/vercel-labs/skills) CLI, which
resolves this repository and writes the skill into whichever coding agents you
select:

```sh
npx skills add opf/openproject-agent-skills
```

Run without flags, it prompts for the target agents and the install scope. Use
`npm i -g skills` if you would rather have a persistent `skills` binary than
go through `npx` each time.

To skip the prompts, name the agents:

```sh
# Claude Code, globally
npx skills add opf/openproject-agent-skills -a claude-code -g

# Codex and Pi
npx skills add opf/openproject-agent-skills -a codex -a pi

# every detected agent, no prompts
npx skills add opf/openproject-agent-skills --agent '*' -y
```

Around 76 agents are supported — see
[Supported Agents](https://github.com/vercel-labs/skills#supported-agents) for
the full list of `--agent` values.

By default the skill is installed into the current project (`.claude/skills/`,
`.agents/skills/`, and so on, depending on the agent). Pass `-g` to install
globally instead — `~/.claude/skills/` for Claude Code, `~/.codex/skills/` for
Codex, `~/.pi/agent/skills/` for Pi.

### Manual install

Without `npx`, copy the skill directory into your agent's skill location:

```sh
cp -r openproject-workpackage-crud ~/.claude/skills/
```

Substitute the skills directory of whichever agent you are using.

## Releasing

Releases are driven by [changesets](https://github.com/changesets/changesets),
following the OpenProject org convention (`primer_view_components`,
`openproject-octicons`, `rubocop-openproject`). When your PR changes skill
behaviour, run `bunx changeset` and commit the generated file alongside your
change. On merge to `main`, a "Release Tracking" PR collects pending changesets;
merging that PR bumps the version, updates `CHANGELOG.md`, and tags the release
on GitHub. Nothing is published to npm — the package is private, and the
`skills` CLI installs straight from this repository.
