# $COMMAND

This repo implements $COMMAND, a command designed to be triggered via api.codesections.com/state.

Specifically, the api repo's message poller claims messages on this command's mailbox and dispatches each one
to the command — either a headless Claude Code session carrying this repo's profile (an LLM command) or this
repo's runner invoked with the claimed message as a JSON file (a raw-json command). The command reads the
message body for its arguments and, when done, sends its results back to the sender's mailbox as a threaded
reply (its `reply_to` set to the triggering message's id). A failed dispatch is reported to the server-assigned
supervisor by the poller, from the failed ack; `state.py msg error` / `state.py msg report` cover errors
outside a dispatched task (see command-docs).

