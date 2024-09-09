# Future work

This chapter contains ideas and plans that have not yet made it into the specification.

## Non-pseudonymous homeserver/groups

It could be advantageous to have groups where the homeserver can identify individual members. This would be an option that could either be turned on or off on a homeserver or on a per-group basis. A variety of mechanisms would have to be adjusted for non-pseudonymous groups. It's also a question we want interoperability between pseudonymous and non-pseudonymous homeservers.

## Plaintext queue configs

Currently, all queue configs are [sealed](glossary.md#sealed-queue-config). This is measure to protect metadata is unnecessary in the case the DS and the client's QS are part of the same homeserver. As a result, it should be possible to store the queue config as plaintext in such an instance.

## Improved spam management for fan-out queues

Friends of a user can currently invite the user's clients to large, very active groups and thus create a large influx of messages for the user's clients. This can be used to spam the user and there is currently no mitigation other than the use of privacy pass tokens. It would be good to allow the QS to categorize incoming messages into SPAM/HAM messages, where in the background, clients only download HAM messages and wait until the application is in the foreground to retrieve SPAM messages as well. This would also allow other UX modes, where the user has to first actively accept joining a group before the client downloads the corresponding SPAM messages.

## Tighter authentication

The AuthTokens required for client-side authentication by QS and DS prove that the client owns a signature key and are bound loosely to the context in which they are used. However, having the signature cover the whole request would provide a tighter bind between authentication and context. One drawback is that we either have to re-serialize the request to sign/verify, or that we have to fix the general request serialization scheme to a deterministic one that accommodates signatures.

## Re-initialization of groups

MLS allows the re-initialization of groups to change ciphersuite or to update the MLS version. We definitely need this sooner or later.

## Protocol versioning

All messages of the homeserver protocol should contain a version number to allow clients and servers to process messages with the right versions. The versions of the protocol supported by a given client should be present in the client's KeyPackage and the version supported by the homeserver should be published in it's .well-known (or similar).

## Ciphersuite agility

It might be interesting to support ciphersuite agility in a similar way as MLS. An alternative would be to tie the ciphersuites to protocol versions.
