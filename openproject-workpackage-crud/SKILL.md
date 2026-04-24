---
name: openproject-workpackage-crud
description: Use when prompts reference OpenProject work packages such as WP#1234, work package URLs, or clearly labeled work package IDs and the task is to inspect, create, or update work packages through local openproject-cli workflows
---

# OpenProject Work Package CRUD

## Overview

Use `openproject-cli` JSON commands as the machine interface for Work Package
CRUD workflows. Read automatically, but use `--dry-run --json` before writes.

## When to Use

- Prompt references `WP#1234`
- Prompt includes an OpenProject Work Package URL
- Prompt says `work package 1234`
- Prompt asks to inspect a Work Package and its subitems
- Prompt asks to create a child Work Package under an existing Work Package
- Prompt asks to update Work Package fields such as KPIs

Do not use this skill for projects, notifications, time entries, or general
OpenProject help.

## Reference Parsing

Recognize only explicit Work Package references in v1:

- `WP#1234`
- full OpenProject Work Package URLs
- clearly labeled numeric references like `work package 1234`

Do not guess from titles or fuzzy search terms.

## Read Workflow

For reads, fetch the parent Work Package plus direct children:

```bash
op inspect workpackage <id> --children --json
```

Summarize:

- the parent Work Package
- direct children only
- relevant custom fields from `fields`
- field mappings from `field_labels` when they matter for follow-up updates

## Create Workflow

Treat "associated with `WP#1234`" as "create a child under `WP#1234`" in this
workflow.

First run:

```bash
op create workpackage --parent <id> --type <type> "<subject>" --dry-run --json
```

After a valid dry-run and clear user intent, execute the real create:

```bash
op create workpackage --parent <id> --type <type> "<subject>" --json
```

If the prompt does not specify enough detail to create the child safely, ask one
short follow-up question.

## Update Workflow

First run:

```bash
op update workpackage <id> --set "Field=Value" --dry-run --json
```

After a valid dry-run and clear user intent, execute the real update:

```bash
op update workpackage <id> --set "Field=Value" --json
```

Prefer human field labels first. Raw API fields like `customField130` are the
fallback when labels are ambiguous or already known.

For supported workflow actions such as `Claim`, `Developed`, and
`Finish implementation`, use:

```bash
op update workpackage <id> --action "<Action name>"
```

If the current CLI does not support `--dry-run --json` for `--action`, treat
actions as a documented exception to the normal write guardrail and verify the
result with a follow-up inspect.

For `Claim`, expect the action to set the assignee and move the Work Package to
the appropriate in-progress status in one step when that action is available
for the Work Package type.

For internal software-development workflows, these action names are especially
common on `Implementation` tickets:

- `Claim`: assign the ticket to the current user and move it into the active
  development state
- `Developed`: move the ticket from an in-development state to review, often
  `needs review`
- `Finish implementation`: close the ticket when implementation work is fully
  complete

Always verify the available actions on the current Work Package state before
executing one, because the action list changes with status and type.

## Body fields and Epic schemas

Field labels are per-project and per-type — the same label may be present on
one Work Package and absent on another. Always read the schema from
`field_labels` on the live response, never assume.

Custom fields that hold markdown bodies come back as objects, not strings:

```json
"customField401": {"format": "markdown", "html": "...", "raw": "..."}
```

Use the `raw` value for any read/modify/write round-trip. The `html` field is
OpenProject's rendered output and will not survive being echoed back as new
content.

The `Epic` type commonly exposes three body fields alongside the standard
`description`:

- `Acceptance criteria` — the detailed specification body for the Epic. If a
  user asks to update a label like "Detailed Specification" on an Epic and the
  schema has no such label, propose `Acceptance criteria` as the likely target
  before declaring `unknown_field`.
- `Motivation and background information` — the "why" behind the Epic.
- `Out of scope` — explicit non-goals.

Epic schemas also commonly expose people/metadata labels such as `Designer`,
`Developers`, `Requested by`, `Roadmap`, `Module`, `List`, `Mockups`, and
`Votes`. Resolve via `--set "<Label>=<value>"` the same way as any custom field.

## Ambiguous Fields

If `--dry-run --json` returns an ambiguous or invalid field resolution, do not
guess. Ask one short follow-up question that names the conflicting field label
or the exact missing detail.

## Write Safety

- Never create or update immediately when `--dry-run` is available.
- Never infer a custom field name if schema resolution is unclear.
- Prefer one short clarification question over a wrong mutation.
- If the CLI returns structured JSON errors, reason from those errors instead of
  switching to raw API calls.
- For `--action` workflows that do not yet support `--dry-run --json`, execute
  the action only when the requested action is explicit, then verify with
  `op inspect workpackage <id> --json`.
- For workflow actions, prefer exact action titles as exposed by the current
  Work Package, including capitalization and spaces.

## Scope

This skill is intentionally limited to Work Package CRUD in the local
`openproject-cli` workflow. It does not cover:

- projects
- notifications
- time entries
- search flows
- attachment URL ingestion
- MCP/server integrations
