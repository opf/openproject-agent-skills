# Baseline scenario (RED)

The reference workflow for this skill is end-to-end ticket inspection and
triage scripting: an agent driving `op` to inspect a parent Work Package, read
its children, file a follow-up Work Package, update its fields, and run a
workflow action — all without falling back to the web UI or raw `/api/v3`
calls.

This file captures how a naive agent fails that workflow when the skill is
**not** loaded, against the current `op work-package` command surface. Each
failure mode is reflected in `loopholes.md`.

## Setup

- Loaded skills: none specific to OpenProject.
- CLI: current `op` build. Commands are resource-first and hyphenated
  (`op work-package inspect`, `op work-package update`), JSON is selected with
  the global `--format json` flag, and `update` takes a fixed set of flags
  (`--subject`, `--type`, `--assignee`, `--description`, `--attach`,
  `--action`). There is no `--set`, `--status`, `--dry-run`, or `--parent`, and
  inspect returns a flat DTO with no custom fields or field-label schema.
- Target: a parent Work Package with several children.
- Goal: read the parent and its children, file a follow-up Work Package, set
  its assignee, and move it forward with a workflow action.

## Failure modes observed

### 1. Reads collapse into a fuzzy search

Without the reference-parsing rules, the agent treats `WP#1234` and "work
package 1234" as ambiguous and falls back to title-based search, opening the
wrong record or asking clarifying questions where the prompt is already
unambiguous. URLs are sometimes scraped via WebFetch instead of having their
numeric ID extracted and inspected via the CLI.

### 2. Stale command syntax

The agent reaches for an earlier CLI shape that no longer exists:

```
op inspect workpackage <id> --children --json
op create workpackage --parent <id> --type <type> "<subject>"
op update workpackage <id> --set "Field=Value" --dry-run --json
```

The current binary rejects these with `unknown command` / `unknown flag`. The
agent loops on flag-name variants instead of switching to the
`op work-package <verb>` form with `--format json`.

### 3. Custom-field and status updates via `--set`

The agent assumes arbitrary fields are writable and runs:

```
op update workpackage <id> --set "Status=in progress"
op update workpackage <id> --set "Story points=5"
```

`--set` does not exist on this CLI, so every variant fails. The agent retries
lowercase keys, synonyms, and `--status`, none of which the binary exposes,
mutating nothing but burning the turn — or, worse, falls through to raw API
calls (failure 7).

### 4. Action names get invented

To move the Work Package forward the agent runs:

```
op work-package update <id> --action "Claim"
```

and gets a no-unique-action error on a type where the intent surfaces as
`Assign to me`. The action list is not exposed in JSON output, so without
guidance the agent guesses `Take`, `Start`, `Begin work`, etc., none of which
exist on this workflow, instead of asking for the exact title.

### 5. Create assumes a parent flag

Asked to file a child under `WP#1234`, the agent runs:

```
op work-package create "Follow-up" -p <project> --parent 1234
```

`--parent` does not exist. The agent either errors out, or drops the flag and
silently creates an orphan Work Package with no link to the intended parent and
no note to the user that the parent was not set.

### 6. Description overwritten without reading first

Asked to amend the description, the agent writes new text straight through:

```
op work-package update <id> --description "<new text only>"
```

Because it never inspected the current `description` first, the existing body
is replaced rather than amended, silently dropping content. There is no
dry-run to surface the diff before it lands.

### 7. Writes execute without confirmation or verification

With no `--dry-run` available, updates and creates run as live mutations on the
first attempt, with no statement of intent beforehand and no `inspect`
afterward. When something is wrong, the agent discovers it only after the
mutation has landed — if at all.

### 8. Fallback to raw `/api/v3` on first CLI error

When the CLI prints a structured error, the agent abandons the CLI and starts
hand-crafting `curl` calls against `/api/v3/work_packages/<id>`, parsing HAL
JSON, and managing `lockVersion` itself. The CLI's error message — which
usually names the offending flag or the problem — is discarded.

## Outcome

- Reads land on the wrong Work Package or stall on needless clarification.
- Time is burned looping on removed flags (`--set`, `--status`, `--dry-run`,
  `--parent`) and stale command syntax.
- An orphan Work Package is created with no parent link and no warning.
- A description is overwritten instead of amended.
- Action probing leaves the Work Package in an unintended state, or the agent
  escapes to raw API calls on the first error.

This is the gap the skill is filed to close.
