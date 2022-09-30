# Future work: Broken state detection and reaction

Clients can provision their own encrypted [ICC](../authentication_service/credentials.md#intermediate-client-credentials) on the DS, as well as the owning user's [EID state](../authentication_service/evolving_identities.md). If a client provides broken values in this way, users that join the group can't authenticate the client or user, thus effectively breaking group joining. Since the state is encrypted and the DS can't (and shouldn't) learn the plaintext as to not fully identity the user or client, it cannot verify the validity of the encrypted ICC or EID state.

However, other clients can detect broken ICC or EID states of other clients and prove to the DS that an encrypted value is indeed invalid. With such a proof, the DS can allow the removal of the offending user, even if the reporting client is not a client of an admin user.

## Broken states and hidden invalid updates

As a general rule, if an Admin breaks the group state in such a way that the DS can't validate, but that all group members can detect, all group members must consider the group closed.

* Clients can send a valid-looking commit (from the DS' point of view), where the update path includes invalid (encrypted) path secrets for one or more subtrees of the committer's direct path.
    * Proposed mitigation: The affected clients can resync. It's not clear if it makes sense for affected clients to react differently depending on the role of the attacking client.
* Clients can upload an encrypted, but broken client credential chain. This will be detected by all other clients. The two sections below propose a detection + reaction strategy, as well as a prevention strategy.
* Clients can corrupt the user profile information of the DS when adding new clients, either by Welcome or by External commit(see the "Join group with new client" endpoint description)

## Future work (stage 1): Detection and removal of the offending user

TODO: Read up on franking and describe the scheme here. It's non-trivial, but we know how to do it.

## Future work (stage 2): Prevention using Zero-Knowledge Proofs

Better than the detection plus reaction step would be to allow the DS to verify the encrypted ICC and EID using ZKPs provided by the client.