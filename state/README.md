# Durable fleet context

No fleet observations have been recorded yet. Create a record when performing
a task; leave unknown fields explicit rather than inventing an initial inventory.

Use `YYYY-MM-DDTHHMMSSZ-<controller>-<topic>.md` with a unique suffix if needed.
Each controller owns its record; reference related records instead of
concurrently rewriting them. Controller identity should distinguish sessions.

Suggested record contents:

```markdown
# Task or survey title

- Owner: controller and session identity
- Started / last updated: UTC timestamps
- Request and scope: the operator's task; hosts/agents/repos covered
- Status: active / waiting / complete
- Related records: paths

## Observations

| Agent/session | Host/environment | Task/repo/branch | Observed at UTC | Evidence | Finding |
| --- | --- | --- | --- | --- | --- |

## Operator actions

Exact action, requester, reason, age, and resolution path; or no actions found
within the stated coverage.

## Communications

Recipient, purpose, incoming message ID, outgoing request ID, observed router
result, acknowledgement, and next follow-up. Record only messages actually made.

## Coverage and uncertainty

Checked members, nonresponders, stale evidence, missing upstreams, remote/PR
checks not performed, and unresolved identity or route questions.

## Decisions and handoff

Decisions made, work owner, remaining steps, and evidence needed to close them.
```

Record the time evidence was observed as well as the time notes were written.
Keep prior observations recognizable when superseded. A pending operator action
closes only on evidence of resolution, cancellation, or an explicit decision.
