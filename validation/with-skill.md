# With-skill scenario (GREEN)

Same setup as `baseline.md` — same CLI build, same Epic, same goal — but with
`openproject-workpackage-crud/SKILL.md` loaded. Each numbered failure from the
baseline is closed by a specific section of the skill.

## How each baseline failure is prevented

### 1. Reads collapse into fuzzy search → Reference Parsing

The skill's "Reference Parsing" section enumerates exactly three accepted
forms (`WP#1234`, full WP URLs, `work package 1234`) and forbids title-based
fallback. The agent inspects via:

```
op inspect workpackage 1234 --children --json
```

and parses `field_labels` + `fields` from the JSON response. URLs are not
scraped; the numeric ID is extracted from the URL path and then inspected via
the CLI.

### 2. `--set "Status=..."` → Status transitions

The "Status transitions" section names this exact pitfall: `Status` is not a
custom field and will return `unknown_field` if pushed through `--set`. The
agent prefers a custom action (`Claim`, `Developed`) when the exact action
title is already known, and the dedicated `--status <name>` flag otherwise.

### 3. Probing candidate status values → form endpoint fallback

A `PropertyConstraintViolation` on `--status` is a stop condition, not a
prompt to retry. The skill instructs the agent to fetch allowed values from:

```
POST /api/v3/work_packages/<id>/form
```

read `_embedded.schema.status._embedded.allowedValues`, report them back to
the user, and ask which to apply. `in progress`, `needs clarification`, and
`decided` are explicitly called out as non-interchangeable.

### 4. Action-name invention → exact-title-only action handling

The skill forbids parsing human CLI output to discover valid action titles.
If the user already provided the exact action title, the agent can execute it
and verify with a follow-up inspect. If the prompt only gives an intent such
as "claim it", the agent asks one short follow-up question for the exact title
or reports that the current CLI contract does not expose actions safely. It
also calls out that on `Epic` and `Open Point` types the valid title is often
something like `Assign to me`, not the development-lifecycle actions used on
`Implementation` tickets.

### 5. Formattable fields treated as objects → CLI-normalized contract

The "Body fields and Epic schemas" section describes the post-#74418 contract:
Formattable custom fields read as raw markdown strings and write through
`--set "Label=<markdown>"`, which the CLI wraps as `{"raw": ...}`
transparently. The agent reads and writes raw markdown only — never the
rendered `html` and never an explicit `{"raw": ...}` wrapper from its own
side.

The round-trip checklist in the skill mandates:

1. capture current `raw` to a recovery file,
2. modify markdown,
3. `--dry-run --json` to validate label resolution,
4. apply, then re-inspect and assert on the new string's length and trailing
   content.

### 6. Epic body schema assumed → field_labels read first

The five-section Epic body convention is documented as a **convention**, not
a stable schema. The agent always reads `field_labels` on the live response
first. If a user asks for "Detailed Specification" and the schema doesn't
expose that label, the agent proposes `Acceptance criteria` (or a subheading
inside it, like `### Alternatives considered and rejected`) before declaring
`unknown_field`.

### 7. Live writes on first attempt → dry-run guardrail

"Update Workflow", "Create Workflow", and "Write Safety" all require
`--dry-run --json` first. The live mutation runs only after the dry-run
returns a clean resolution and the user's intent is unambiguous.

### 8. Fallback to raw `/api/v3` → structured-error reasoning

"Write Safety" is explicit: "If the CLI returns structured JSON errors, reason
from those errors instead of switching to raw API calls." The form-endpoint
fallback in (3) is the *only* sanctioned raw-API call, and it's read-only.

## Outcome

- Reads land on the right Work Package every time, with `field_labels` and
  custom-field IDs in context.
- No stray status writes; rejected transitions surface allowed values to the
  user instead of triggering a probe loop.
- Action workflows do not rely on parsing human CLI output; they use an exact
  user-provided title or stop for clarification.
- Formattable body edits round-trip cleanly with a recovery copy and a
  post-apply inspect assertion.
- Epic body fields write to the correct label after a live `field_labels`
  read.
- Writes are gated by dry-run.
- The CLI stays the primary interface; raw `/api/v3` is reserved for the
  read-only allowed-values lookup.

The Epic-edit + child-create + claim + transition workflow completes in one
scripted pass, matching the WP#74316 intent.
