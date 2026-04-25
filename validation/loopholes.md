# Loophole log

Iterative refinements made to the skill after the GREEN scenario revealed
gaps. Each entry maps a discovered failure mode to the commit that closed it
and the section of `SKILL.md` that now carries the rule.

## L1 — Action names are not stable across types and projects

**Discovered:** `--action "Claim"` failed on Work Package types where the
intent surfaces as `Assign to me` or `Take`, with the CLI returning
`No unique available action from input Claim found`. The agent's response was
to invent neighboring names.

**Fix:** Forbid parsing human CLI output to discover action titles. Require an
exact user-provided action title when using `--action`; otherwise ask one short
follow-up question or report that the current CLI contract does not expose
actions safely. Call out that on `Epic` and `Open Point` types the valid title
is often `Assign to me` rather than a development-lifecycle action.

**Commit:** `79bf63d [#74413] harden skill for real-world WP updates`
**Skill section:** Update Workflow → action-name variability paragraph; Write
Safety.

## L2 — `Status` is not a custom field

**Discovered:** Agents reasoned by analogy from `--set "Module=..."` and ran
`--set "Status=in progress"`, getting `unknown_field`, then probed lowercase
variants and synonyms, mutating shared state with each guess.

**Fix:** Add an explicit "Status transitions" section. `--set` is forbidden
for status. Prefer custom action first, then dedicated `--status` flag.

**Commit:** `79bf63d [#74413] harden skill for real-world WP updates`
**Skill section:** Status transitions.

## L3 — Allowed status transitions live on the form endpoint, not the schema

**Discovered:** When `--status <name>` was rejected with
`PropertyConstraintViolation: no valid transition exists from old to new
status for the current user's roles`, agents tried to fetch the allowed
values from the plain schema GET at `_links.schema.href` — which does not
populate `allowedValues`. They then probed candidate names sequentially.

**Fix:** Document that `POST /api/v3/work_packages/<id>/form` is the only
source of `allowedValues` (computed in the context of the current user and
state), and require the agent to surface allowed values to the user rather
than choose one. Explicitly enumerate that `in progress`, `needs
clarification`, and `decided` are not interchangeable intents.

**Commit:** `79bf63d [#74413] harden skill for real-world WP updates`
**Skill section:** Status transitions; Write Safety bullet on iteration.

## L4 — Formattable body fields lost content on round-trip

**Discovered (older CLI):** Formattable custom fields came back as
`{"format": "markdown", "html": "...", "raw": "..."}`. Agents either echoed
`html` back as new content (overwriting markdown with rendered HTML) or sent
a plain string where `{"raw": "..."}` was required, silently truncating the
field.

**Fix (first pass):** Require capturing the pre-update `raw` to a local file
for recovery; mandate post-apply `inspect` with length and trailing-content
assertions; document that `--dry-run --json` does not catch wire-format bugs.

**Commit:** `79bf63d [#74413] harden skill for real-world WP updates`
**Skill section:** Body fields and Epic schemas; Write Safety.

## L5 — Formattable contract changed under the skill

**Discovered:** The CLI was updated (#74415, #74418) so that Formattable
custom fields normalize to a raw markdown string on read and the CLI handles
`{"raw": ...}` wrapping on write transparently inside `--set`. The skill's
older guidance — to treat the field as a `{format,html,raw}` object — became
not just outdated but actively wrong: agents who applied it would
double-wrap their payload.

**Fix:** Rewrite the body-fields section to describe the new contract: read
and write raw markdown directly, never construct `{"raw": ...}` from the
agent side, never echo `html`. The pre-update recovery copy and post-apply
inspect remain mandatory because dry-run still does not validate server
acceptance, `lockVersion`, or server-side coercion.

**Commit:** `0ebd101 [#74418] align skill with Formattable field contract`
**Skill section:** Body fields and Epic schemas.

## L6 — Epic body schema assumed instead of read

**Discovered:** Agents assumed Epics had a fixed three-field body
(`description`, `Motivation and background information`, `Acceptance
criteria`) and tried to write `Detailed Specification` as a custom field.
When that returned `unknown_field` they declared the field missing rather
than recognizing that on this project the content lives under `Acceptance
criteria`, sometimes inside an `### Alternatives considered and rejected`
subheading.

**Fix:** Promote the schema description to a five-section content
**convention** (description, Motivation, Acceptance criteria, Out of scope,
Alternatives), explicitly flag it as not a stable schema, and require a
`field_labels` read on every Epic before writing. Call out that
`Alternatives` is often a heading-level subsection of `Acceptance criteria`
rather than a separate custom field.

**Commit:** `0bc55dd document the five-section Epic body convention`
**Skill section:** Body fields and Epic schemas → Epic five-section
convention.

## L7 — CLI contract dependency was undeclared

**Discovered (during code review of WP#74413):** The skill assumed CLI flags
(`--json`, `--children`, `--dry-run --json`, `--action`, `--status`) that
the released `openproject-cli` build does not yet expose. An agent invoking
the skill on a stock CLI hit `unknown flag` and had no documented recourse.

**Fix:** Add a `Prerequisites` section listing the required flags, a probe
instruction (`op inspect workpackage --help`, `op update workpackage --help`),
and a stop-and-report rule for the case where the contract is missing.
Explicitly defer pinning to a SHA (unmerged branches rebase) and to a semver
version (no release containing the contract exists yet); the flag list itself
is the contract until a release ships.

**Commit:** This branch — `[#74413] declare openproject-cli contract as
skill prerequisite`.
**Skill section:** Prerequisites.
