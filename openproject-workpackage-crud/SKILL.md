---
name: openproject-workpackage-crud
description: Use when prompts reference OpenProject work packages such as WP#1234, work package URLs, or clearly labeled work package IDs and the task is to inspect, create, or update work packages through local openproject-cli workflows
---

# OpenProject Work Package CRUD

## Overview

Use `op work-package` commands as the machine interface for Work Package CRUD.
Reads can run automatically. The current CLI has no `--dry-run`, so confirm
intent before any write, then verify the result with a follow-up inspect.

Pass `--format json` for machine-readable output. `op` accepts either a numeric
ID (`1234`) or a project-based identifier (`PROJ-123`) anywhere an ID is taken.

## When to Use

- Prompt references `WP#1234`
- Prompt includes an OpenProject Work Package URL
- Prompt says `work package 1234`
- Prompt asks to inspect a Work Package and its children
- Prompt asks to create a Work Package in a project
- Prompt asks to update a Work Package subject, type, assignee, description, or
  run a workflow action

Do not use this skill for projects, notifications, time entries, budgets, or
general OpenProject help.

## Reference Parsing

Recognize only explicit Work Package references:

- `WP#1234` → `1234`
- `work package 1234` → `1234`
- full OpenProject Work Package URL → extract the numeric ID from the WP path
  segment, then use that ID in CLI calls (do not scrape the page)

Do not guess from titles or fuzzy search terms. If a URL has no unambiguous
numeric ID, ask one short follow-up. Use `op work-package search <query>...`
only when the user explicitly asks to search.

## Read Workflow

Inspect the parent, then list its direct children:

```bash
op work-package inspect <id> --format json
op work-package list --parent-id <id> --format json
```

`op work-package inspect <id> --types` lists the types available on the Work
Package (use before changing type). `--open` opens it in a browser.

The JSON inspect response is a flat object:

```json
{
  "id": 60586,
  "display_id": "AC-70",
  "subject": "...",
  "type": "...",
  "status": "...",
  "assignee": "...",
  "description": "<raw markdown>"
}
```

`description` is raw markdown — safe for read/modify/write round-trips. The CLI
does not return custom fields or a field-label schema, so only the fields above
are available to read.

## Create Workflow

```bash
op work-package create "<subject>" -p <project> --type <type> \
  --assignee <user-id> --description "<markdown>"
```

`-p/--project` is required and takes a project numeric ID or identifier. Subject
is the positional argument. `--assignee` takes a numeric user ID. If the prompt
lacks the project or enough detail to create safely, ask one short follow-up.

The CLI has no `--parent` flag: it cannot set a parent at create time. If the
user asks for a child under `WP#1234`, say the CLI cannot link a parent and
confirm whether to create it standalone in the same project instead.

## Update Workflow

Each flag is applied as its own update. Supported flags:

```bash
op work-package update <id> --subject "<text>"
op work-package update <id> --type <type>
op work-package update <id> --assignee <user-id>      # numeric user ID
op work-package update <id> --description "<markdown>"
op work-package update <id> --attach <filepath>
op work-package update <id> --action "<Action name>"  # workflow transition
```

To edit description text, inspect first to read the current `description`, apply
the change locally, then write the full new markdown back via `--description`.

### Workflow actions and status

Status is not a writable field on this CLI; status changes go through
`--action`/`-a`, which runs a named workflow transition. Action titles vary by
Work Package type, project, and the current user's role, and they are **not**
exposed in the JSON output, so they cannot be discovered machine-readably.

- If the user gave the exact action title, use it verbatim — capitalization and
  spaces included.
- If they gave only an intent ("claim it", "move to review"), ask one short
  follow-up for the exact title rather than guessing neighboring names.
- Action availability is state-dependent: a title valid in one status may be
  invalid in another. On non-`Implementation` types (`Epic`, `Open Point`) the
  claim-equivalent is often `Assign to me`, not a development-lifecycle action.

Common actions on internal `Implementation` tickets (examples, not defaults):

- `Claim`: assign to current user and move into the active development state
- `Developed`: move from in-development to review (often `needs review`)
- `Finish implementation`: close when implementation is fully complete

## Write Safety

- No `--dry-run` exists. State the exact intended change, confirm clear user
  intent, execute, then run `op work-package inspect <id> --format json` to
  verify the post-state.
- Prefer one short clarification question over a wrong mutation.
- Do not invent action titles or iterate through candidates against a shared
  Work Package. A rejected action is a stop condition: ask for the exact title.
- If the CLI returns structured errors, reason from those errors instead of
  switching to raw `/api/v3` calls.

## Not Yet Supported By The CLI

These were available in earlier CLI iterations but are absent from the current
binary. Do not attempt them; if a user asks, say they are not available through
this CLI today (they may return in a later release):

- arbitrary custom field / KPI updates (no `--set "Field=Value"`)
- reading custom fields or a field-label schema (`field_labels`)
- Epic body fields beyond `description` (e.g. `Acceptance criteria`,
  `Motivation and background information`, `Out of scope`)
- setting status by name (`--status`) — use `--action` instead
- setting a parent on create (`--parent`)
- `--dry-run` write previews

## Scope

Limited to Work Package CRUD via local `op work-package` commands. Does not
cover projects, notifications, time entries, budgets, search-driven flows,
attachment URL ingestion, or MCP/server integrations.
