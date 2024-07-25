# Security assumptions

The operator of a homeserver can leverage its conrol over the homeserver to influence if and how the homeserver serves its users. While the goal of the protocol should be to reduce the amount of control the operator has, there are a few assumptions that need to be made.

## Operators are trusted not to spam users

Homeserver operators can circumvent any server-side spam reduction mechanisms and will thus be able to send excessive amounts of messages to a user and their clients (or notifications if the user provides the homeserver with a means to notify them). While the client can impose its own spam reduction mechanisms that the homeserver operator can't circumvent, they will likely also impede the usability of the client.

## Operators are trusted not to deny services to individual users

The homeserver operator can control which user the homeserver provides its services to and to which ones it doesn't. Metadata minimalism already mitigates this problem to a certain degree, because in most cases, the operator shouldn't know who makes a certain request. However, some actions require explicit authentication, allowing the operator to lock out users in a targeted way. Similarly, the operator can simply delete records of a specific user from the database, or deny a given user the ability to publish KeyPackages.
