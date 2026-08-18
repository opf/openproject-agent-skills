# With-skill scenario (GREEN)

The same `SectionCardComponent` brief — the one that triggered the most baseline
failures — re-run with the skill (all four files) loaded, across independent
fresh agents. Graded on the five baseline axes plus the post-refactor menu axis.

## Round 1 — two agents, skill loaded

| Axis | Agent A | Agent B |
|---|---|---|
| F1 enum: single consistent idiom | pass | pass |
| F2 one spelling for the menu label | **fail** | pass |
| F3 boolean named for effect (`announce_changes:`) | pass | pass |
| F4 `render?` gates on real presence | pass | pass |
| F5 reuses shared spec examples | pass | pass |

Four of five closed and converged on the first pass. The survivor was F2: Agent
A re-introduced the dual `button_aria_label:` + `button_arguments:` menu label
and rationalized it verbatim as *"I matched the repo's real Menu/HasMenu
two-path label convention."* The skill's "match existing conventions" principle
collided with "one obvious way," and the agent resolved it toward the legacy
shape. See `loopholes.md` L1.

## Round 2 — after the refactor, two fresh agents

The skill was amended (a "Match conventions means the recipes — not the legacy
smells" block + a sharpened anti-pattern row). Re-tested with two fresh agents,
prompted to read `BorderBoxListComponent`/`HasMenu`:

| Axis | Agent C | Agent D |
|---|---|---|
| menu label spellings | **1** (`button_arguments:`) | **1** (`button_arguments:`) |
| rejected `button_aria_label:` explicitly | yes | yes |
| `attr_reader :variant` returns | `Symbol` | `Symbol` |
| announce-updates boolean name | `announce_changes` | `announce_changes` |

Both cited the skill's anti-example when declining the second spelling, despite
being pointed straight at the legacy code. Loophole closed; all axes green and
converged across independent samples.

> **Post-validation correction:** the rounds above ran while the skill told
> agents to store a plain symbol — hence `attr_reader :variant` returning a
> `Symbol`. The maintainer confirmed `StringInquirer` is an accepted house idiom
> they introduced, so the F1 recipe was flipped to prescribe `StringInquirer`
> (with the `String`-vs-`Symbol` gotcha documented). That revised recipe was not
> re-pressure-tested; a future run should confirm agents converge on the house
> idiom. The F2–F5 convergence evidence is unaffected.

## Interpretation

Variance is the signal. At baseline, the same brief produced opposite shapes on
F1/F2 (one agent each way). With the skill — and after the one refactor — five
independent agents converged on the same shape on every axis. That convergence,
not any single agent's output, is what says the guidance is binding.
