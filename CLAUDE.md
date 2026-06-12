# Client command template

This repo is a **template** for client commands — programs that run as
message-dispatched headless Claude Code sessions in the personal state
system (`api.codesections.com`). Copy it when creating a new command.

The full system is documented in `/home/fw13/Projects/api.codesection.com/api/docs/`. The
reference client script is `/home/fw13/Projects/api.codesection.com/api/client/scripts/state.py`.

---

## What a client command is

The personal state server includes a message queue (`/msg`) with a
claim/ack lifecycle. The **poller** (`api/client/poller/msg_poller.py`)
runs as a systemd timer, claims pending messages for its mailbox, and
dispatches them to `claude -p <prompt>` — spawning a headless Claude Code
session whose stdout becomes the ack result.

A **client command** is a defined contract for a class of those messages:
a named operation with a CLI-style argument structure (parsed from the
message body), a mailbox it subscribes to, and a documented result shape.
This lets non-agentic senders (Claude Chat, scripts) invoke a command by
name without encoding the full task as freeform prose.

---

## Two command shapes

A command is implemented in one of two dispatch styles (the poller's
`--arg-style` / `STATE_MSG_ARG_STYLE`):

- **LLM command** (`prompt`, the default): the poller builds a prompt
  from the message and spawns `claude -p`, usually with the command's
  `profile.md` appended via `--append-system-prompt-file`. The headless
  operating contract below applies, including the `FAILED:` sentinel.
- **Raw runner** (`raw-json`): a deterministic, non-LLM program. The
  poller writes the full claimed message object to a JSON file and
  invokes the runner with that path as its final argument. No prompt is
  built and the `FAILED:` sentinel does not apply — the runner's exit
  code is the failure signal (0 → `done`, nonzero → `failed`). stdout
  still becomes the ack result.

See `lifecycle.md` in the shared command-docs for the contract both
shapes follow.

---

## Operating contract (headless sessions)

These rules bind the dispatched **LLM** session (`prompt` style); a raw
runner replaces the sentinel with its exit code. Every session dispatched
by the poller runs **unattended**. No human can
approve interactive prompts. This creates binding rules:

- **Prefer shell tools over MCP tools.** `gh`, `git`, `python3`, `curl`
  don't require permission prompts. MCP tools do — and a stalled prompt
  exits 0 having done nothing, which looks like success.
- **Lead with `FAILED: <reason>` on any failure.** The poller trusts this
  sentinel more than exit codes. Use it when the task cannot be completed
  for *any* reason: ambiguous arguments, missing permissions, unreachable
  service, precondition not met.
- **Never act on a resource not explicitly named in the message.** If the
  task doesn't name a repo, directory, or remote, do not guess. Respond
  `FAILED: ambiguous target — <what you'd need to know>`.
- **Never ack the message yourself.** The poller owns the ack. If you call
  `state.py msg ack`, you'll race it and one will win a 409.
- **Keep stdout small.** The ack result is truncated to ~1500 chars. Link
  the artifact (PR URL, commit SHA, file path); don't embed it.

---

## Message lifecycle for a command

A message arrives at the command's mailbox (e.g. `code::fw13::cmd::$COMMAND`).
The poller:

1. Claims it (`POST /msg/{id}/claim`) — the server-side CAS; a 409 means
   another worker won, don't act.
2. Writes the body to a temp file.
3. Spawns: `claude -p "<prompt with body file: <path>>"` (prompt style;
   in raw-json style it writes the full message object as JSON and passes
   that file's path to the runner instead).
4. Acks with the session's stdout as result.

Inside the dispatched session (this command):

1. Read the body file for arguments: `cat <path>` or parse as needed.
2. Send results back to the sender's mailbox, threaded under the
   triggering message: `state.py msg send --to <sender>
   --reply-to <id> --subject "done" --body-file <result>`.
3. Output a compact result line to stdout. If anything went wrong,
   output `FAILED: <reason>` as the **first line of stdout**.

---

## Using state.py

All API interactions go through `state.py` — never direct HTTP. The
script lives at `/home/fw13/Projects/api.codesection.com/api/client/scripts/state.py`; in
deployed command packages it is included alongside the command.

Key commands:

```sh
# Read the full message details (metadata + body in a temp file)
state.py msg show <id>          # body is in the temp file at the path printed

# Send a reply
state.py msg send --to <mailbox> --reply-to <original-id> \
  --subject "done: <summary>" --body-file result.txt

# Check/update shared state
state.py get                    # fetches {map, data} to a temp file
state.py put --file <edited> --base <get-file> -m "<what changed>"
```

Secrets are at `~/.config/state/secrets.json`. The script finds them
automatically; never `cat` or print the secrets file.

---

## Mailbox subscription

Each command subscribes to a namespaced mailbox under the machine's
client-id, e.g. `code::fw13::cmd::$COMMAND`. Subscribe at startup
(idempotent):

```sh
state.py msg subscribe --as code::fw13::cmd::$COMMAND --note "$COMMAND handler"
```

The poller must be pointed at this mailbox:

```
STATE_MSG_MAILBOX=code::fw13::cmd::$COMMAND
```

or passed via `--mailbox code::fw13::cmd::$COMMAND` on the poller invocation.
Run one poller instance per mailbox.

---

## Result / error reporting

- **Success:** print the result (compact summary + artifact links) on
  stdout. Also send the result to the sender's mailbox, threaded via
  `--reply-to <triggering message id>`.
- **Failure:** first stdout line must be `FAILED: <reason>`. The poller
  acks `failed` and the sender sees the reason in `state.py msg replies`.
  The poller also derives an error report to the supervisor from that
  failed ack (one fact, one writer) — do **not** also report the failure
  yourself with `msg error`.
- **Rejections carry usage:** when the failure is a rejection of the
  message itself (bad arguments, malformed body), the rejection text
  must include the command's usage / expected body format — a raw
  runner prints the usage in the rejection before its nonzero exit; an
  LLM command includes the usage after the `FAILED: <reason>` line. The
  failed ack then documents the correct call for both the sender (via
  `msg replies`) and the supervisor (in the derived error report),
  making a corrected resend the cheap repair.
- **Partial:** send partial results to the sender as a threaded reply
  before the session ends; the ack captures whatever stdout remains.
- **Errors/warnings outside the dispatched task** (e.g. broken local
  setup noticed in passing): `state.py msg error` (actionable) or
  `state.py msg report` (informational). No
  destination is given — the server resolves the current supervisor from
  the owner-edited routing table. See `lifecycle.md` in the shared
  command-docs for the full error path.

---

## Pointers

| What | Where |
|------|-------|
| Full system docs | `/home/fw13/Projects/api.codesection.com/api/docs/` |
| Client script | `/home/fw13/Projects/api.codesection.com/api/client/scripts/state.py` |
| Poller (dispatch logic) | `/home/fw13/Projects/api.codesection.com/api/client/poller/msg_poller.py` |
| Poller README | `/home/fw13/Projects/api.codesection.com/api/client/poller/README.md` |
| Client guide (secrets, install) | `/home/fw13/Projects/api.codesection.com/api/docs/client-guide.md` |
| Messaging design | `/home/fw13/Projects/api.codesection.com/api/docs/messaging-design.md` |
| API reference | `/home/fw13/Projects/api.codesection.com/api/docs/api-reference.md` |
