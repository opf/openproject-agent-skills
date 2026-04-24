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

Action names vary by workflow, project, and Work Package type. The same intent
may surface as `Claim`, `Assign to me`, `Take`, or something project-specific.
If the CLI returns `No unique available action from input X found`, read the
action list the CLI prints and pick from it — do not invent names. In
particular, on Work Package types that are not `Implementation` tickets (e.g.
`Epic`, `Open Point`), the list is often a single item like `Assign to me` and
will not include development-lifecycle actions.

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

## Status transitions

`Status` is not a custom field — it will not resolve through `--set`. Do not
attempt `--set "Status=<name>"`; it returns `unknown_field`.

Prefer, in order:

1. A custom action that transitions status (e.g. `Claim`, `Developed`), when
   the action list exposes one.
2. The dedicated `--status <name>` flag on `op update workpackage`, when the
   CLI build supports it.

Status transitions are gated by the current user's role **and** the workflow
defined for the Work Package's type. A transition that looks obvious may still
be denied with
`PropertyConstraintViolation: no valid transition exists from old to new status
for the current user's roles`.

When that happens, do not probe by trying other status names in sequence —
that mutates shared state with agent-guessed values. Keep the CLI as the
default interface: try the explicit `--status` update first, and only if it is
rejected and the CLI does not expose the allowed transitions, fetch them from
the form endpoint:

```
POST /api/v3/work_packages/<id>/form
```

and read `_embedded.schema.status._embedded.allowedValues` (names) or
`_embedded.schema.status._links.allowedValues` (hrefs). The plain schema GET
at `_links.schema.href` does **not** populate `allowedValues` — only the form
endpoint does, because allowed transitions are computed in the context of the
current user and current state.

Report the allowed values back to the user and ask which to apply. Do not
choose a replacement status yourself; `in progress`, `needs clarification`,
and `decided` are not interchangeable intents.

## Body fields and Epic schemas

Field labels are per-project and per-type — the same label may be present on
one Work Package and absent on another. Always read the schema from
`field_labels` on the live response, never assume.

Formattable custom fields are normalized to raw markdown strings in the JSON
inspect output, the same shape as `description`:

```json
"customField401": "### Heading\n\nBody text with a [link](...)..."
```

Read and write this value directly as markdown. Do not treat it as an object
with `format` / `html` / `raw` keys — older CLI builds exposed that shape, but
recent versions collapse Formattable custom fields to the raw string on read
and wrap them back into `{"raw": ...}` on PATCH transparently inside
`--set "Label=<markdown>"`.

For a read/modify/write round-trip:

1. Capture the current string value into a local file so you have a recovery
   copy if the patch is later observed to have truncated content.
2. Modify the markdown.
3. `op update workpackage <id> --set "<Label>=<new markdown>" --dry-run --json`
   to validate schema resolution.
4. Apply, then re-`inspect` and assert on the new string's length and
   expected trailing content.

`--dry-run --json` validates label resolution and value coercion, not the
server's acceptance of the payload, the `lockVersion`, or any server-side
coercion of the raw content. Post-apply inspect remains the authoritative
check on any body-field write.

The `Epic` type is typically authored as five logical body sections — one
standard field and four markdown sections carried in custom fields. The exact
schema varies per project, so always confirm via `field_labels` on the live
response before writing; the list below is a content convention, not a
guarantee.

- `description` — the short framing paragraph. Standard field, serialized as a
  raw string.
- `Motivation and background information` — the "why" behind the Epic,
  including key limitations of the status quo.
- `Acceptance criteria` — the detailed specification body. If a user asks to
  update a label like "Detailed Specification" on an Epic and the schema has
  no such label, propose `Acceptance criteria` as the likely target before
  declaring `unknown_field`.
- `Out of scope` — explicit non-goals.
- `Alternatives considered and rejected` — options that were weighed and
  discarded, with brief rejection rationale. In many projects this is **not a
  separate custom field** but a heading-level subsection at the end of
  `Acceptance criteria` (e.g. `### Alternatives considered and rejected`).
  Before declaring `unknown_field`, check whether the existing
  `Acceptance criteria` markdown already contains such a heading, and
  prefer a read/modify/write on that field to proposing a new one.

This is a convention, not a stable schema. Content conventions age quickly
and OpenProject has no machine-readable "body layout" for Epics today, so
verify each Epic's actual `field_labels` on read and treat missing labels
as "not on this template" rather than "wrong call".

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
- A successful `--dry-run --json` is necessary but not sufficient: it validates
  label resolution and value coercion, not server acceptance of the patch. For
  Formattable body fields, for any first-time use of `--set` on a field type
  you haven't previously round-tripped, and after any status or link change,
  follow up with `op inspect workpackage <id> --json` and assert on the
  post-state.
- Do not iterate through candidate statuses to find one the workflow accepts.
  A rejected transition on a shared Work Package means: stop, fetch allowed
  values from the form endpoint, and ask the user.

## Scope

This skill is intentionally limited to Work Package CRUD in the local
`openproject-cli` workflow. It does not cover:

- projects
- notifications
- time entries
- search flows
- attachment URL ingestion
- MCP/server integrations
