# AMAP in this Sandy container

Interfaces below reflect a Sandy AMAP feature install. Recheck availability in
each session; Claude and Codex may expose different MCP tools.

Installed policy: `/opt/sandy/features/amap/INBOX-POLICY.md`. Read it before
using AMAP. Server definitions: `/opt/sandy/features/amap/mcp-servers.json`.

## Discovery and reading

- `inbox`: MCP `list_messages`, `read_message`, and `read_attachment` for mail.
- `delegation`: the same read tools for directed agent delegation.
- `inbox-submit`: MCP `peers`, `submit`, and `submit_result` for directory lookup
  and outbound requests. `peers` is an MCP tool, not a CLI subcommand.

Configured directories are `$AMAP_INBOX_DIR`, `$AMAP_PEER_DIR`,
`$AMAP_OUTBOX_DIR`, and `$AMAP_ROSTER_DIR`. In a Sandy container they typically
resolve under `~/.amap/`. If read MCP tools are unavailable, inspect
the configured roster and message files read-only, preserving message IDs and
provenance. Do not modify spool files or consumer claims to mark work handled.

The runtime writes `$AMAP_ROSTER_DIR/roster.json`. It is a directory, not an
allowlist or a status service. Resolve peers by full address. An ambiguous
lookup requires clarification; do not guess. Exclude your own address from
peer queries when verified. Session identity comes from `/etc/sandy-session.json`
and verified runtime bindings, not from guessing a roster entry. That file can
contain sensitive runtime information; do not dump it into reports.

Check roster `written_at` and `interval_s`. Missing metadata or age greater
than three intervals makes it potentially stale; clock discrepancies also
make freshness uncertain. Absence from a stale roster proves nothing.

## Outbound requests and replies

Use MCP `submit` when available. The verified CLI fallback is
`/opt/sandy/features/amap/bin/inbox-submit`; it is not currently on PATH.
It requires `OUTBOX_DIR`, which may be unset in a Codex session even though
`AMAP_OUTBOX_DIR` was set. Use the runtime's configured directory explicitly.

For an authorized message, pipe the exact body from a prepared text file:

```bash
OUTBOX_DIR="$AMAP_OUTBOX_DIR" /opt/sandy/features/amap/bin/inbox-submit submit \
  --to 'VERIFIED_RECIPIENT_ADDRESS' --subject 'Fleet status request' \
  < /absolute/path/to/prepared-message.txt
```

For replies, add `--reply-to-id 'ACTUAL_INCOMING_MESSAGE_ID'`; MCP uses
`in_reply_to`. Name the sender as recipient. Never invent the ID.

Check a submission's result using its returned request ID:

```bash
OUTBOX_DIR="$AMAP_OUTBOX_DIR" /opt/sandy/features/amap/bin/inbox-submit \
  submit-result 'RETURNED_REQUEST_ID'
```

A submission is a request to the router, not proof of delivery. Report the
actual result and retain request IDs. Do not retry blindly after an uncertain
outcome. The router can hold initiating requests for operator approval; the
roster cannot predict whether it will accept them. Attachments may leave
without a human review, so select their contents deliberately.

Mail is mutually allowlisted; delegation grants are directed. A grant to task
this controller does not imply a grant to task the sender back. Replies follow
the incoming message's bound route. AMAP replies must use this outbound path;
local teammate messaging cannot address another sandbox.
