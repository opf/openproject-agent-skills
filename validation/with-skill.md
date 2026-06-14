# With-skill scenario (GREEN)

Same setup as `baseline.md` — same `op` build, same Work Package, same goal —
but with `openproject-workpackage-crud/SKILL.md` loaded. Each numbered failure
from the baseline is closed by a specific section of the skill.

## How each baseline failure is prevented

### 1. Reads collapse into fuzzy search → Reference Parsing

The "Reference Parsing" section enumerates exactly the accepted forms
(`WP#1234`, `work package 1234`, full WP URLs) and forbids title-based
fallback. URLs are not scraped; the numeric ID is extracted from the URL path
and then inspected via the CLI:

```
op work-package inspect 1234 --format json
op work-package list --parent-id 1234 --format json
```

### 2. Stale command syntax → resource-first commands

The skill's Read/Create/Update workflows use only the current
`op work-package <verb>` form with the global `--format json` flag. There is no
`inspect workpackage`, no `--children`, and no `--json`; children come from
`list --parent-id`.

### 3. `--set` custom-field / status updates → fixed-flag updates

The "Update Workflow" and "Not Yet Supported By The CLI" sections state that
`--set` does not exist and arbitrary custom fields and KPIs are not writable.
Updates use only the supported flags (`--subject`, `--type`, `--assignee`,
`--description`, `--attach`, `--action`). If the user asks for a custom field,
the agent says it is not available through this CLI rather than guessing flag
variants.

### 4. Action-name invention → exact-title-only action handling

"Workflow actions and status" forbids guessing action titles, which are not
exposed in JSON output. If the user gave the exact title, the agent uses it
verbatim and verifies with a follow-up inspect. If they gave only an intent
("claim it"), the agent asks one short follow-up for the exact title. It calls
out that on `Epic` and `Open Point` types the claim-equivalent is often
`Assign to me`.

### 5. Create assumes a parent flag → documented `--parent` gap

"Create Workflow" states plainly that the CLI has no `--parent` flag and cannot
link a parent at create time. Rather than silently creating an orphan, the
agent tells the user the parent cannot be set and confirms whether to create
the Work Package standalone in the same project.

### 6. Description overwritten → read-modify-write

"Update Workflow" requires inspecting the current `description` (raw markdown)
first, applying the change locally, then writing the full new markdown back via
`--description`. The existing body is amended, not replaced.

### 7. Live writes without confirmation → confirm-then-verify

"Write Safety" replaces the missing dry-run with a discipline: state the exact
intended change, confirm clear user intent, execute, then run
`op work-package inspect <id> --format json` to verify the post-state.

### 8. Fallback to raw `/api/v3` → structured-error reasoning

"Write Safety" is explicit: reason from the CLI's structured errors instead of
switching to raw API calls. There is no sanctioned raw-API escape hatch in this
contract.

## Outcome

- Reads land on the right Work Package every time, via the current command
  surface.
- No looping on removed flags or stale syntax; unsupported requests get a clear
  "not available through this CLI" instead of guesswork.
- No orphan Work Packages: the `--parent` gap is surfaced before creating.
- Descriptions round-trip cleanly through a read-modify-write.
- Action workflows use an exact user-provided title or stop for clarification.
- Every write is preceded by a stated intent and followed by a verifying
  inspect; the CLI stays the only interface.

The inspect + child-create + assignee + action workflow completes in one
scripted pass.
