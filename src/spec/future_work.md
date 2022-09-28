# Future work

This chapter contains ideas and plans that have not yet made it into the specification.

## Non-pseudonymous homeserver/groups

It could be advantageous to have groups where the homeserver can identify individual members. This would be an option that could either be turned on or off on a homeserver or on a per-group basis. A variety of mechanisms would have to be adjusted for non-pseudonymous groups. It's also a question we want interoperability between pseudonymous and non-pseudonymous homeservers.

## Plaintext queue configs

Currently, all queue configs are [sealed](glossary.md#sealed-queue-config). This is measure to protect metadata is unnecessary in the case the DS and the client's QS are part of the same homeserver. As a result, it should be possible to store the queue config as plaintext in such an instance.

## Improved spam management for fan-out queues

Friends of a user can currently invite the user's clients to large, very active groups and thus create a large influx of messages for the user's clients. This can be used to spam the user and there is currently no mitigation other than the use of privacy pass tokens. It would be good to allow the QS to categorize incoming messages into SPAM/HAM messages, where in the background, clients only download HAM messages and wait until the application is in the foreground to retrieve SPAM messages as well. This would also allow other UX modes, where the user has to first actively accept joining a group before the client downloads the corresponding SPAM messages.