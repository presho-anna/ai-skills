# Decision Log: `.pipeline/decisions.md`

The decision log is the pipeline's shared, append-only memory. It exists so that (a) no agent diverges from the ticket or the spec silently, (b) later agents and later runs don't re-litigate settled choices, and (c) the human can audit *why* the code looks the way it does without reading transcripts.

**Rules:**

- Append-only. Never edit or delete an entry — to reverse a decision, append a new entry that supersedes it and reference the old ID.
- Every agent reads the whole log at the start of its step, before touching code.
- Log the decision **at the moment it is made**, not retrospectively in a summary.
- One entry per decision. Small is fine; four lines that save a future argument are worth it.
- The Reviewer treats "diff differs from spec/ticket with no matching entry" as a Fail.

## File template (created at Step 0)

```markdown
# Decision Log — <ticket id / short title>
Baseline: <sha> | Started: <date>

<!-- Append entries below. Never edit existing entries. -->
```

## Entry format

```markdown
## D<NN> [<TYPE>] <one-line title>
- **Agent:** Planner | Coder | Tester | Reviewer | Orchestrator
- **Decision:** what was decided, concretely
- **Rationale:** why, in 1–3 sentences; name the alternative rejected if there was one
- **Impact:** files/concepts affected; anything the ticket owner should confirm
- **Supersedes:** D<NN> (only if applicable)
```

## Entry types

| Type | Use when | Example |
| ---- | -------- | ------- |
| `RENAME` | A concept from the ticket gets a different name in code, or an existing concept is renamed | Ticket says "user tags"; codebase already calls these "labels" — spec uses `Label` throughout |
| `DIVERGENCE` | The work deviates from the ticket's or spec's literal wording for any reason | Spec's target file was deleted last sprint; equivalent logic now lives in `services/`, implemented there instead |
| `ARCHITECTURE` | A pattern/approach was chosen where more than one was viable | Used the existing `Notifier` pub/sub rather than a direct call, matching how the other 3 event types work |
| `ASSUMPTION` | Proceeding required assuming something unverified | Assuming the CSV export is ≤ 10k rows since no ticket mention of pagination; flag if wrong |
| `HEREBEDRAGONS` | A hazard, trap, or surprising coupling was discovered — whether or not it was touched | `OrderMailer` is invoked reflectively from `admin/hooks.rb`; renaming its methods breaks admin silently |
| `DEFERRED` | Something in or near scope was consciously not done | Rate-limiting the new endpoint deferred; needs product input on limits |

## Worked example

```markdown
## D03 [DIVERGENCE] Validation moved from controller to model
- **Agent:** Coder
- **Decision:** Implemented the email-format check as an ActiveModel validation on `Subscriber`, not in `SubscriptionsController` as spec'd.
- **Rationale:** The controller is also hit by the bulk-import job, which the spec's placement would have bypassed. Model-level covers both paths.
- **Impact:** `app/models/subscriber.rb`, `spec/models/subscriber_spec.rb`; spec.md §2 now inaccurate on file path.
```
