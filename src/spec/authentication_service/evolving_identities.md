# Future work: Evolving Identity (EID)

On a high level, the evolving identity represents a user's identity, which in turn is composed of the cryptographic identities of all of the user's devices.

More concretely, this is implemented using an MLS group that includes all of a user's clients, with the group's [ratcheting tree](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#name-ratchet-tree-concepts) representing the user's EID state.

Users update their EID by performing commits in the group using [MLS plaintexts](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#name-message-framing) as their framing.

## The EID state

The EID state is the ratcheting tree of an MLS group that contains all of a user's devices. It defines the user's cryptographic identity, but it's also the place of publication for the [client credentials](credentials.md#client-credentials) of all of the user's clients.

The leaves in the EID state thus don't contain Leaf Credentials like the leaves in other groups, but instead contain the client credential of the client represented by the leaf.

The EID state is managed by the client, but published by the AS and held by all clients that communicate with the user.


## Client management

The EID is the authoritative list of the user's clients. Any change to the user's line-up of clients thus needs to be first made in the EID.

Since the EID is at its core an MLS group, changes to a user's device can be made using MLS membership management operations such as commits to update, add and remove proposals.

Any commit to such a change needs to be sent to the AS for publication, but also to all of the user's groups. In order for all these parties to be able to track changes to the user's identity, the commit needs to be in an MLS plaintext message. Any recipient of such a commit can validate the visible information in the commit, and in particular verify the signature against the current state of the EID.

### Cross-signing

One goal of the EID is to implement two-directional client cross signing, i.e. have both the adding client and the added client confirm the addition with their signatures. To achieve this, recipients of a commit to the EID that contains a client addition MUST wait for a second commit sent by the new client that only contains an update by that client.

## Future work: User ID changes

In a future iteration, users will be able to change their user name by terminating their existing EID and starting a new one. The old and the new EID group will be cross-linked to one-another cryptographically to ensure that the user name change is fully authenticated.

## Future work: AS tickets

There is currently no way for other clients to judge how recent a given EID state is. Having a strict bound on how recent an EID is required to be is important to ensure prompt propagation of client removal. Adding a "ticket", i.e. a signed message from the AS containing a time stamp and the hash of the EID state could help with this.

The protocol would then be extended by requiring clients to regularly upload a ticket attesting the validity and recency of the EID state currently uploaded to the group. Clients receving a message from another client could then always require that the client's AS ticket be recent (for a configurable definition of "recent").