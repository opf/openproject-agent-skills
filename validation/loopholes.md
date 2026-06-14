# Loophole log

Iterative refinements made to the skill after the GREEN scenario revealed
gaps. Each entry maps a discovered failure mode to the rule in `SKILL.md` that
now carries it.

> **Contract change:** earlier iterations of this skill targeted a CLI that
> exposed `--set`, `--status`, `--dry-run`, `--children`, `--json`, `--parent`,
> and custom-field / Formattable read-write. The current `op` build dropped all
> of these in favour of a resource-first command surface
> (`op work-package <verb>` with global `--format json`) and a fixed update
> flag set. Loopholes tied to the removed features (status-as-field, the form
> endpoint allowed-values lookup, Formattable body objects, the five-section
> Epic schema, the flag-probe prerequisite) were retired with that contract;
> their underlying intents now live in the rules below. See L8.

## L1 — Action names are not stable across types and projects

**Discovered:** `--action "Claim"` failed on Work Package types where the
intent surfaces as `Assign to me` or `Take`, with the CLI returning a
no-unique-action error. The agent's response was to invent neighboring names.

**Fix:** Forbid guessing action titles. They are not exposed in JSON output, so
require an exact user-provided title; otherwise ask one short follow-up. Call
out that on `Epic` and `Open Point` types the valid title is often
`Assign to me` rather than a development-lifecycle action.

**Skill section:** Update Workflow → Workflow actions and status; Write Safety.

## L2 — Status is changed by action, not by a field

**Discovered:** Agents reasoned by analogy from other field updates and tried
`--set "Status=..."`, then `--status "..."`. Neither flag exists on this CLI,
so the attempts failed and the agent looped on variants.

**Fix:** State that status is not a writable field and changes go through
`--action`. List `--set` and `--status` under "Not Yet Supported By The CLI".

**Skill section:** Update Workflow → Workflow actions and status; Not Yet
Supported By The CLI.

## L3 — Create cannot set a parent

**Discovered:** Asked for a child under `WP#1234`, agents assumed a `--parent`
flag, then silently created an orphan Work Package with no link to the intended
parent and no warning to the user.

**Fix:** Document that the CLI has no `--parent` flag. Require the agent to tell
the user the parent cannot be set and confirm whether to create the Work
Package standalone in the same project.

**Skill section:** Create Workflow.

## L4 — Descriptions overwritten instead of amended

**Discovered:** With no dry-run to preview the change, agents wrote new
`--description` text straight through, replacing the existing body and silently
dropping content.

**Fix:** Require a read-modify-write: inspect the current `description` (raw
markdown), apply the change locally, then write the full new markdown back.

**Skill section:** Update Workflow.

## L5 — No dry-run means confirm-then-verify

**Discovered:** Updates and creates ran as live mutations on first attempt,
with no statement of intent and no check afterward. The earlier skill leaned on
`--dry-run --json` as its guardrail; that flag no longer exists.

**Fix:** Replace the dry-run guardrail with a discipline: state the exact
intended change, confirm clear user intent, execute, then `inspect --format
json` to verify the post-state.

**Skill section:** Write Safety.

## L6 — Custom fields and KPIs are not available

**Discovered:** Agents expected `field_labels` and custom-field IDs in the
inspect JSON and tried to write KPIs or Epic body fields (e.g. `Acceptance
criteria`) via `--set`. The current inspect returns a flat DTO and there is no
`--set`.

**Fix:** Document that inspect returns only the flat field set and that custom
fields, KPIs, and Epic body fields beyond `description` are not available. If a
user asks, say so rather than guessing a workaround.

**Skill section:** Read Workflow; Not Yet Supported By The CLI.

## L7 — No raw `/api/v3` escape hatch

**Discovered:** On the first structured CLI error, agents abandoned the CLI and
hand-crafted `curl` calls against `/api/v3`, managing `lockVersion` themselves.

**Fix:** Require reasoning from the CLI's structured errors. There is no
sanctioned raw-API call in this contract.

**Skill section:** Write Safety.

## L8 — CLI command surface migrated

**Discovered:** The skill targeted `op <verb> workpackage` with `--json` and a
flag-probe `Prerequisites` section. The current build uses resource-first,
hyphenated commands (`op work-package <verb>`) with the global `--format json`
flag, lists children via `list --parent-id`, and exposes a fixed update flag
set. Stale syntax fails with `unknown command` / `unknown flag`.

**Fix:** Rewrite all command examples to the `op work-package <verb>` form,
replace `--json` with `--format json`, replace `inspect --children` with
`list --parent-id`, and drop the flag-probe prerequisite in favour of a fixed
"Not Yet Supported By The CLI" list.

**Skill section:** Overview; Read / Create / Update Workflow; Not Yet Supported
By The CLI.
