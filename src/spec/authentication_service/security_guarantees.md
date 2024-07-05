# Authentication security guarantees

This section outlines how the Phoenix homeserver protocol facilitates end-to-end authentication.

## Overview

The authentication system consists of the authentication services (AS’, one operated by each homeserver operator), which act as a trusted third party for their respective homeserver, and a validation module run on each client. For more information on the consequences of a malicious or compromised AS, see [here](../../threat_model/qualitative/authentication_service.md).

The AS of a homeserver issues client credentials its users’ clients. The clients can then use the credentials to sign the leaf credentials used in each of their groups or in their pre-published MLS KeyPackages. The leaf credentials are then used to sign messages as per the MLS specification.

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

The process above ensures that Bob’s current view of Alice’s identity is correct.

### Connection groups and multi-client identity

As detailed [here](./connection_establishment.md), users have a connection group with each of their contacts (but not necessarily with each user they are in a group with). The connection group acts as a reference for users w.r.t. what clients represent an individual contact. If a user is invited into a new group by one of its contacts, but the inviting client is not part of the corresponding connection group, the invited user must decline the invitation.
