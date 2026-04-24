# Baseline scenario (RED)

The reference workflow for this skill is the kind of end-to-end ticket-filing
and triage scripting described in WP#74316: an agent driving `op` to inspect a
parent Work Package, file follow-up children with full body content, populate
fields, and transition status — all without falling back to the web UI or raw
`/api/v3` calls.

This file captures how a naive agent fails that workflow when the skill is
**not** loaded. The failure modes here are not hypothetical; each one motivated
a hardening commit on this branch and is reflected in `loopholes.md`.

## Setup

- Loaded skills: none specific to OpenProject.
- CLI: `openproject-cli` build that exposes `--json`, `--children`,
  `--dry-run --json`, `--set`, `--action`, `--status`, and Formattable-field
  read/write normalization (#74415, #74418).
- Target: an Epic Work Package with a populated `Acceptance criteria`
  Formattable body and several `Implementation` children.
- Goal: read the Epic, edit `Acceptance criteria`, file a child `Implementation`
  ticket under it, claim the child, and later move it forward in the workflow.

## Failure modes observed

### 1. Reads collapse into a fuzzy search

Without the reference-parsing rules, the agent treats `WP#1234` and "work
package 1234" as ambiguous and falls back to title-based search, opening the
wrong record or asking clarifying questions where the prompt is already
unambiguous. URLs are sometimes scraped via WebFetch instead of inspected via
the CLI, so `field_labels` and custom-field IDs never enter the agent's
context.

### 2. Status changes via `--set "Status=..."`

The agent infers from the field-label pattern that `Status` is just another
custom field and runs:

```
op update workpackage <id> --set "Status=in progress"
```

The CLI returns:

```
unknown_field: Status
```

Without guidance, the agent retries with `--set "status=..."`, then
`--set "State=..."`, mutating shared state with each guess until one happens
to land or a workflow constraint rejects it.

### 3. Status changes by probing candidate values

Once the agent learns about `--status`, it runs through plausible names in
sequence:

```
op update workpackage <id> --status "in progress"
op update workpackage <id> --status "needs clarification"
op update workpackage <id> --status "decided"
```

OpenProject returns:

```
PropertyConstraintViolation: no valid transition exists from old to new
status for the current user's roles
```

Each rejected attempt is a write against a real Work Package. The agent has no
notion that allowed values must be fetched from the form endpoint
(`POST /api/v3/work_packages/<id>/form`) rather than guessed.

### 4. Action names get invented

For an Epic, the agent runs:

```
op update workpackage <id> --action "Claim"
```

and gets:

```
No unique available action from input Claim found
```

The CLI prints the actual action list (often a single `Assign to me` for
non-Implementation types). Without guidance, the agent ignores that list and
tries `Take`, `Start`, `Begin work`, etc., none of which exist on this
workflow.

### 5. Formattable body fields treated as objects

On older CLI builds the agent saw `customField401` as
`{"format": "markdown", "html": "...", "raw": "..."}` and learned to send
the same shape back on write. The current CLI (#74418) returns the field as a
raw markdown string and accepts a raw markdown string on `--set`. Without an
updated contract the agent either:

- echoes the rendered `html` back and overwrites the body with a flattened,
  link-broken version of itself, or
- wraps the new content as `{"raw": "..."}` inside `--set`, which now
  double-wraps and produces malformed payloads.

There is no automatic recovery copy of the pre-update content, so a silent
truncation is unrecoverable from the agent's side.

### 6. Epic body schema assumed, not read

The agent assumes a fixed three-field Epic schema (`description`, `Motivation`,
`Acceptance criteria`) and tries to write `Detailed Specification` directly:

```
op update workpackage <id> --set "Detailed Specification=..." --dry-run --json
```

returns `unknown_field`. The agent declares the field missing and either
creates a new custom field (out of scope) or asks the user — instead of
recognizing that on this project the content lives under
`Acceptance criteria`, optionally inside an `### Alternatives considered and
rejected` subheading.

### 7. Writes execute without dry-run

Updates and creates run as live mutations on first attempt. When schema
resolution is ambiguous, the agent discovers the problem only after the
mutation lands.

### 8. Fallback to raw `/api/v3` on first CLI error

When the CLI prints a structured error, the agent abandons the CLI and starts
hand-crafting `curl` calls against `/api/v3/work_packages/<id>`, parsing HAL
JSON, and managing `lockVersion` itself. The CLI's error message — which
usually names the offending field or the available actions — is discarded.

## Outcome

- Multiple stray status writes against a shared ticket.
- One Formattable body field overwritten with rendered HTML and no recovery
  copy.
- Action probing leaves the Work Package in an unintended assignee state.
- The original goal (file a child, claim it, move it forward) takes several
  supervised rounds rather than one scripted pass.

This is the gap WP#74413 was filed to close.
