# Future work: Broken state detection and reaction

Clients can provision their own encrypted [credential chain](../authentication_service/credentials.md#intermediate-client-credentials) on the DS. If a client provides broken values in this way, users that join the group can't authenticate the client or user, thus effectively breaking group joining. Since the state is encrypted and the DS can't (and shouldn't) learn the plaintext as to not fully identity the user or client, it cannot verify the validity of the encrypted credential chain.

However, other clients can detect broken credential chains of other clients and prove to the DS that an encrypted value is indeed invalid. With such a proof, the DS can allow the removal of the offending user, even if the reporting client is not a client of an admin user.

## Broken states and hidden invalid updates

As a general rule, if an Admin breaks the group state in such a way that the DS can't validate, but that all group members can detect, all group members must consider the group closed.

* Clients can send a valid-looking commit (from the DS' point of view), where
  the update path includes invalid (encrypted) path secrets for one or more
  subtrees of the committer's direct path.
  * Proposed mitigation: The affected clients can resync. It's not clear if it
    makes sense for affected clients to react differently depending on the role
    of the attacking client.
* Clients can upload an encrypted, but broken client credential chain. This will
  be detected by all other clients. The two sections below propose a detection +
  reaction strategy, as well as a prevention strategy.
* Clients can corrupt the user profile information of the DS when adding new
  clients, either by Welcome or by External commit(see the "Join group with new
  client" endpoint description)
* A user can add a client from another group member as one of its own.
  * Proposed mitigation: Add proof-of-ownership for KeyPackages when adding a
    client of the same user

## Future work (stage 1): Detection and removal of the offending user

One way of dealing with bad updates is to use franking and allow users prove to the DS that a given update to the group state was broken. The DS can then allow ordinary members to remove the member. Note that it's proabably a reasonable trade-off that the identity of the offender is revealed to the DS in the reporting process. However, revealing group secrets is not.

## Future work (stage 2): Prevention using Zero-Knowledge Proofs

Better than the detection plus reaction step would be to allow the DS to verify the encrypted credential chain using ZKPs provided by the client.
