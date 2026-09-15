# cali-mcp-sync

Private data mailbox between **CALI** (Anthony's desktop calendar app) and **Claude**,
used by a local MCP server. There is no live tunnel or server exposed to the internet —
this repo *is* the connection. Both sides only ever read/write plain files here over
normal git operations.

## Why this exists

CALI already has a full local REST API (`sync_server.py`, port 8765) for its phone app,
but that API only exists on Anthony's home network. This repo lets an MCP server running
on the same desktop answer Claude's questions using the latest synced data, and lets
Claude queue changes that CALI applies the next time it's running — without needing
either side to be reachable from the internet at the same time.

**Design principle:** a stale-but-available answer is correct, not a failure. GitHub is
always reachable; CALI's desktop is not always on. Reads should never block on "is the
desktop currently running" — they answer from the last successful sync.

## Layout

- `data/events.json` — full CALI events export (all fields except internal DB row IDs
  are still included for reference, but note `id` is a *live* CALI database id, not
  stable across a delete+recreate).
- `data/tasks.json` — full CALI tasks export, same shape as the phone-sync API's
  `/api/tasks` response.
- `requests/` — pending write-back requests from Claude (one JSON file per request).
  CALI's watcher polls this folder, applies each request through its own local
  `sync_server.py` REST API (never direct DB access), then moves the file to
  `processed/` with the result recorded.
- `processed/` — completed requests, kept as an audit trail. A request is never
  re-applied once it's been moved here.

## Who writes what

| Direction | Writer | Files |
|---|---|---|
| CALI → Claude | CALI (`app/github_sync.py`) | `data/events.json`, `data/tasks.json` |
| Claude → CALI | the local MCP server | `requests/*.json` |
| Applying a request | CALI (`app/github_sync.py`) | moves `requests/*.json` → `processed/*.json` |

Nothing in this repo is meant to be edited by hand. It's machine-to-machine plumbing.
