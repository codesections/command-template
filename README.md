# $COMMAND

This repo implements $COMMAND, a command designed to be triggered via api.codesections.com/state.

Specifically, the program polls one or more mailboxes for incoming messages and claims those that trigger
$COMMAND. It then reads the body of the message for arguments to the command. When done, it sends its results
back to the sender's mailbox as a threaded reply (its `reply_to` set to the triggering message's id). Errors and warnings go to the
server-assigned supervisor via `state.py msg error` / `state.py msg report` (see command-docs).

