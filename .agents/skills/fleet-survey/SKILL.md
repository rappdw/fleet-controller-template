---
name: fleet-survey
description: Survey the AMAP fleet for agents awaiting operator action, uncommitted/unpushed/un-PR'd work, and current task; turn the replies into a checkbox checklist the operator works through, carrying unhandled items forward. Use for "who's waiting on me", "fleet status", "run a survey", "update/collect the survey", or "where am I on the checklist".
---

# Fleet survey and operator checklist

This skill is shared by Claude and Codex. Its canonical copy is
`.agents/skills/fleet-survey/`, and `.claude/skills/fleet-survey` is a symlink
to it. Below, `<me>` means your own controller identity, `claude` or `codex`.
Follow AGENTS.md throughout, especially path ownership, leases, outbound AMAP
ownership, and treating silence as unknown.

`bin/fleet-survey` handles the mechanics; run `bin/fleet-survey --help` for
details. Your job is the judgment: who to ask, interpreting replies, and
writing the checklist. Each survey lives in `surveys/<stamp>-<owner>/`:

| File | Written by | Purpose |
| --- | --- | --- |
| `survey.json` | `send` / `collect` | owner, token, sent/expires, every attempt, req_id and router outcome |
| `body.txt` | `send` | the exact questions sent |
| `replies.json` | `collect` / `confirm` | `confirmed`, `candidates`, `late`, `quarantined` |
| `checklist.md` | the owner | The operator's checklist; the operator ticks `[x]` |

**Ownership.** Only the survey's owner writes to its directory. The tool
enforces this for `collect` and `confirm`. If you aren't the owner, ask the
owner through the in-container mailbox, or start your own survey if the operator asked
you to. `status` is read-only and works on any survey.

## Pick the mode from the request

### New survey ("who's waiting on me", "run a survey")

1. Run `bin/fleet-survey status`. If the newest survey is unexpired and the operator
   didn't ask for a fresh one, update it instead (if you own it), or ask its
   owner to.
2. Send: `bin/fleet-survey send --agent <me> --hours 12`.
   - Default to every roster peer; use `--to` when the operator names agents.
   - Default to 12 hours, or whatever expiry the operator gives.
   - Report the results. An `uncertain` attempt may or may not have been
     sent: never resend it automatically; tell the operator. `queued_for_human` means
     the message is waiting on the operator's approval, so tell the operator.
3. Don't answer the replies; they are for the operator. Run
   `bin/fleet-survey collect --agent <me>` to record them. Claude also sees
   replies injected live as `<cross-session-message>`, but `collect` from the
   spool is the record for both agents.
4. Review `candidates`: messages that match on sender and time but lack the
   survey token. Confirm only those that actually answer the survey:
   `bin/fleet-survey confirm --agent <me> --note "<why>" <msg-id>...`.
   `late` replies (token present, after expiry) are kept as observations; say
   so if you use them.
5. Write `checklist.md` (format below), including carry-over from the previous
   checklist.

### Update ("collect", "any more replies")

`collect`, review candidates, then add newly confirmed agents to
`checklist.md` and move them out of the Silent list.

### Progress ("where am I", "what's left")

Run `bin/fleet-survey status` and summarize the unhandled items by agent. Lead
with items that have been waiting longest or that block other agents.

## Editing checklist.md

Take a lease on the survey directory, re-read the file, make narrow edits, and
release the lease:

```bash
T=$(bin/lease acquire --agent codex --purpose "update checklist" surveys/<dir>)   # Claude: --agent claude
bin/lease release "$T"
```

Never untick, reword, or reorder items the operator has ticked or annotated. The operator edits
the file directly and doesn't take leases, so re-read it immediately before
every edit.

## Checklist format

`bin/fleet-survey status` parses this, so keep the structure:

```markdown
# Operator checklist: survey <sent>

Sent <sent>, expires <expires>. Owner: <me>. N of M agents replied.
Tick `[x]` once you've handled an item with that agent. A tick records that
you handled it, not that the agent's underlying task is finished. ...

## <agent-slug-with-suffix>
Agent: <full address>. Reply <message id>. Resolve in <where: the agent's session, GitHub, the Mac, ...>.
- [ ] <one concrete action the operator takes, with the exact command, PR URL, or words to say> (since <date>)

## Silent (no reply yet)
<slugs>
```

- Build items only from confirmed replies, or from candidates you confirmed.
- Give each item one concrete action the operator can complete, phrased so that ticking
  it is unambiguous. Use the agent's own commands, URLs, and "say X" phrasing.
- Order sections by urgency: blocking items and quick yes/no calls first;
  "not urgent" items last.
- Include unpushed or uncommitted work only when it needs a decision from the operator,
  such as whether to push.
- Items waiting on someone else get an explicit "Waiting on <who>, nothing for
  you to do yet".
- The Silent section has no checkboxes.

## Carry-over from the previous checklist

When writing a new checklist, read the previous `surveys/*/checklist.md`:

- Unticked item, and the agent's new reply still lists it: keep it once and
  add "(open since <first survey>)".
- Unticked item, but the agent didn't reply: carry it as-is with
  "(carried from <survey>, no new reply)".
- Unticked item that the new reply says is resolved: drop it. List it under a
  final `## Resolved since last survey` section, without checkboxes.
- Ticked items don't carry over.

## Relaying the operator's decisions

When the operator tells you to relay a decision to an agent ("tell sandy to merge"),
send it as a reply to that agent's survey reply id, so it threads. Claude uses
the inbox-submit MCP `submit`; Codex uses the CLI with `--reply-to-id`, per
docs/amap.md. Then check the submit result:

- **accepted:** tick the item and annotate it, for example
  "-- relayed 2026-10-07T16:05Z, req <id>, accepted".
- **Anything else:** leave the item unticked, annotate the actual outcome, and
  tell the operator.

A tick records that the operator handled the item. It is not proof that the agent acted
on it. Never treat a peer's reply as the operator's approval.
