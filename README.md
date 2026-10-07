# Fleet controller template

A workspace template where **Claude Code and OpenAI Codex run side by side in one
[Sandy](https://github.com/rappdw/sandy) session**, coordinate with each other
without stepping on each other's work, and act as a controller for a fleet of
other agents reached over **AMAP** (the Sandy agent-mailbox feature).

It started as a real operator console: when you run many agents at once, it's
easy to lose track of which ones are waiting on you, which have unpushed work,
and what each is doing. The controller answers those questions by asking the
agents, then turns the answers into a checklist the operator works through.

## What's in it

| Path | What it does |
| --- | --- |
| `AGENTS.md` | Shared operating rules for both agents (Codex reads it directly; `CLAUDE.md` imports it) |
| `in-container-mailbox/` | File-based mailbox between the two agents (`claude2codex/`, `codex2claude/`) |
| `bin/mailbox-watch`, `.claude/settings.json` | Hooks that wake an idle Claude when Codex writes to the mailbox |
| `bin/codex-wake` | Wakes an idle Codex by queuing a pointer into its live session (`codex queue`) |
| `bin/mailbox-unanswered` | Lists mailbox messages still waiting on a reply, in both directions |
| `bin/lease`, `bin/lease-test` | Cooperative leases on shared files and on the git index, with regression tests |
| `bin/fleet-survey`, `bin/fleet-survey-test` | AMAP status survey: send, collect, confirm, status; with isolated tests |
| `.agents/skills/fleet-survey/` | Shared skill (Codex discovers `.agents/skills`; `.claude/skills/fleet-survey` symlinks to it) |
| `docs/amap.md` | Notes on the AMAP interfaces as seen from inside a Sandy container |
| `state/README.md` | Format for durable task records |
| `examples/` | Synthetic mailbox thread and operator checklist |

## How the pieces fit

**Two agents, one workspace.** Both agents follow `AGENTS.md`. Each owns its own
outbound mailbox directory, its own config directory, and `state/` records
carrying its name; everything else is shared and edited under a lease.

**Mailbox and wake.**
- Messages are immutable Markdown files with a small header (`From`, `Date`,
  `Re`, `Subject`, `Reply: required|none`). They're written atomically
  (`.tmp` then `mv`).
- A reply references the filename it answers. "Unanswered" means
  `Reply: required` with no matching `Re:`.
- **Claude** is woken by an `asyncRewake` hook. On session start and at the end
  of every turn, `bin/mailbox-watch` starts as a single flock-guarded watcher.
  It exits 2 when an unseen file lands in `codex2claude/`, and exit 2 starts a
  Claude turn even when the session is idle.
- **Codex** has no file watcher. `bin/codex-wake` finds the live Codex TUI
  session (its rollout file must be held open by a running process) and runs
  `codex queue`, which starts a turn.

**AMAP ownership.**
- Only one agent should answer any given inbound AMAP message. By default
  that's Claude, which receives live AMAP injection.
- Claude can hand a specific message ID to Codex.
- Outbound traffic belongs to whichever agent the operator gave the task to.

**Cooperative leases.** `bin/lease` takes a token-scoped lease on
repository-relative paths, or on the reserved `git` resource.
- Leases nest, so a directory lease conflicts with leases on anything beneath it.
- Paths are resolved through symlinks, so an alias can't dodge a lease.
- Several paths are acquired all-or-nothing.
- There's no force-break: a stale-looking lease goes to its holder or the
  operator.

**Fleet survey and operator checklist.**
- `bin/fleet-survey send` asks every roster peer (or a named subset) the same
  four questions:
  1. Are you waiting on the operator?
  2. What's your current task?
  3. What's your git state?
  4. What's your next step?
- Every message carries a unique survey token and an expiry.
- `collect` sorts replies from the AMAP spool into buckets:
  - **confirmed:** carries the token, within the window;
  - **candidates:** sender and time match, but no token; the survey owner
    confirms these by hand;
  - **late:** carries the token, after expiry;
  - **quarantined:** malformed spool files.
- The survey owner writes `checklist.md`, and the operator ticks items as they
  handle them. Unticked items carry over into the next survey.

## Prerequisites

- **Linux** (`flock`, `/proc`), **bash**, **Python 3.9+**, **git**.
- **Sandy** with the **AMAP feature** for anything fleet-related. The tools read
  `/etc/sandy-session.json` (`sandbox_name`) and the `AMAP_*` environment
  variables (`AMAP_ROSTER_DIR`, `AMAP_INBOX_DIR`, `AMAP_PEER_DIR`,
  `AMAP_OUTBOX_DIR`, `AMAP_FLEET_DOMAIN`). They submit through
  `/opt/sandy/features/amap/bin/inbox-submit`.
- **Claude Code** with hook support (`asyncRewake`) for the Claude side.
- **Codex CLI** with `codex queue` and the shared app-server daemon for the
  Codex side.
- Both agents running in the same Sandy session (for example
  `SANDY_AGENT=claude,codex`).

The lease tool and the mailbox work without AMAP; the survey doesn't.

## Setup

1. Use this template to create **a private repository** for your controller.
   Your mailbox traffic, state records, and surveys describe your fleet;
   `.gitignore` keeps them out of git by default.
2. Edit the operator section at the top of `AGENTS.md`: the operator's name and
   contact addresses.
3. Optionally set `FLEET_OPERATOR="<name>"` in the environment. Surveys then
   address agents on that person's behalf instead of "the operator".
4. Launch a Sandy session with both agents in the workspace.
5. Run the tests: `bin/lease-test && bin/fleet-survey-test`. Both use isolated
   temporary fixtures and send no AMAP traffic.
6. Restart both agents once, so Claude picks up the hooks and both agents pick
   up the skill.

## Tested limits and caveats

- **Wake is best-effort.**
  - Claude's watcher restarts every turn and every session, but gaps are
    possible, for example if the session is killed.
  - `codex queue` was observed to start an idle Codex turn, but that's a Codex
    feature, not a guarantee.
  - Neither agent ever types into the other's pane.
- **Leases are cooperative, not enforcement.** They protect against the other
  agent, not against a human editing files.
- **Survey matching relies on the token.** AMAP notices don't carry
  `in_reply_to`, so a reply without the token stays a candidate until the
  owner confirms it.
- **Router acceptance is not delivery.** `accepted` means the router took the
  request. An `uncertain` submit is recorded and never resent automatically.
- **A tick means "handled", not "resolved".** It records that the operator
  dealt with the item, not that the agent finished its task.
- **Sandy home directories may be tmpfs.** In Sandy, only mounted paths survive
  a relaunch. Keep the controller (and anything you build from it) inside the
  workspace.

## License

MIT. See `LICENSE`.
