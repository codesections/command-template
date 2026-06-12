# $COMMAND

This repo implements $COMMAND, a command designed to be triggered via api.codesections.com/state.

Specifically, the program polls one or more mailboxes for incoming messages and claims those that trigger
$COMMAND. It then reads the body of the message for arguments to the command. When done, it sends its results
via a message to the `reply_to` address specified in the message it received. Errors and warnings go to the
server-assigned supervisor via `state.py msg report error|warn` (see command-docs).

