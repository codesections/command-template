# $COMMAND

This repo implements $COMMAND, a command designed to be triggered via api.codesections.com/state.

Specifically, the program polls one or more mailboxes for incomming messages and claimes those that trigger 
$COMMAND. It then reads the body of the message for arguments to the command. When done, it sends its results 
via a message to the `reply_to` address specified in the message it recieved. It sends reports errors and logs
to its server-assigned supervisor.

