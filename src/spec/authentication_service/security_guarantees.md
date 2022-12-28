# Authentication security guarantees

In this document, we outline how the Phoenix homeserver protocol facilitates end-to-end authentication in this first phase of the project. This first approach will provide a baseline of authentication guarantees for users that is comparable to commonly used messaging applications. In a future version, we intend to provide more extensive authentication guarantees that go significantly beyond the current state-of-the-art in secure messaging. Among other things, [evolving identity](evolving_identities.md) will be a central part of v2 of the Phoenix homeserver authentication system.

## Overview

The authentication system consists of the authentication services (AS’, one operated by each homeserver operator), which act as a trusted third party for their respective homeserver, and a validation module run on each client. We will go into more detail on how much users have to trust their AS [here](./security_guarantees.md#threat-model-and-security-guarantees).

The AS of a homeserver issues client credentials for each of its users’ clients. The clients can then use the credentials to sign the leaf credentials used in each of their groups or in their pre-published MLS KeyPackages. The leaf credentials are then used to sign messages as per the MLS specification.

When a client receives a message from another client, it performs the following validation steps:

1. The signature on the message is valid w.r.t. the credential in the sender’s leaf.
2. The signature on the sender’s leaf is valid w.r.t. the sender’s client credential.
3. The signature on the sender’s client credential is valid w.r.t. one of the credentials published by the AS.

If all signatures are indeed valid, the recipient can conclude that the message came from the user indicated by the user name in the sending client’s client ID.

Similarly, a joiner in a new group performs steps 2. and 3. to authenticate each member of the new group.

### Out-of-band validation

To ensure security even against a compromised AS, users can validate the client credentials of their contacts out-of-band. The validation process is as follows:

- Alice sends a hash of her current client credentials to Bob out-of-band
- Bob hashes the credentials of Alice’s clients in their shared connection group and compares the result.
- Bob compares the two hash values to see if his view of Alice’s composed user identity is correct.

The process above ensures that Bob’s current view of Alice’s identity is correct. Note that users have to re-validate out-of-band if a user adds one or more clients.

### Connection groups and multi-client identity

As detailed [here](./connection_establishment.md), users have a connection group with each of their contacts (but not necessarily with each user they are in a group with). The connection group acts as a reference for users w.r.t. what clients represent an individual contact. If a user is invited into a new group by one of its contacts, but the inviting client is not part of the corresponding connection group, the invited user must decline the invitation.

For this to work reliably, users have to first add new clients to all of their connection groups before adding them to any other groups. Similarly, users have to remove clients first from all other groups before removing them from their connection groups.

This requires some care in the add/remove process, where clients have to coordinate through the user’s all-client group s.t. there the addition and removal process is robust.

## Threat model and security guarantees

Generally, the threat model of end-to-end authentication is an adversary that controls everything but the users’ clients, which includes the network, as well as infrastructure components such as the homeserver. However, in this first version of the Phoenix homeserver authentication system, we rely on the AS to perform some security-critical operations. As a consequence, we examine two scenarios in this section: One in which the AS is not malicious/compromised and one where it is.

### Security under an honest AS

If the AS can be trusted to behave honestly, it will issue client credentials only to the user registered under the user name contained in the client ID. As a consequence, performing the verification steps outlined above will ensure that

- a given message originates from the specific client that signed the message,
- the sending client is associated with the user name in the client ID and
- that user is registered with the signing homeserver under the claimed user name.

### Security under a compromised AS

In case a homeserver (or the AS specifically) is compromised or its operator malicious, security guarantees are limited, as the AS can create its own credentials for arbitrary users. However, there are still some security guarantees that the authentication system (in conjunction with MLS) provides in that scenario.

If two users have a connection (that was established while the AS was honest), both users cannot be impersonated towards one another by the AS. This is ensured by users adding clients to their connection groups before all other groups, thus effectively performing cross-signatures that all of their contacts can verify.

The adversary can generally impersonate a user towards non-contacts by creating a fresh client credential with a client ID containing the user’s user name. With this credential, the adversary can create connection groups with any non-contact. Theoretically, the adversary can even impersonate a user towards non-contacts in groups that the user itself is already in. However, this requires the adversary to learn which group to target (since group membership is hidden from the homeserver). Additionally, after joining such a group, the adversary must effectively block the user from that group to avoid detection (although the block itself might alert the user to the attack).