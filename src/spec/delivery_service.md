# Delivery Service (DS)

The delivery service keeps track of groups and (pseudonymous) group membership.

TODO: (future work, i.e. long term) For each operation below, include a 'validation' subsection that includes the validation steps performed by the DS.
TODO: Remove mention of EID (which is future work for now) and replace mention of the ICC with an ICC + CC bundle.
TODO: Rename subsections to be actual endpoint names.

## DS configuration options

* Maximal KeyPackageBatch age: Maximal difference between a KeyPackageBatch timestamp and the current time when a new user is added to a group.
  * Default: 1h
* Maximal DSAuthToken age: Maximal age of an [DSAuthToken](delivery_service.md#authentication) presented to the DS for client authentication.
  * Default: 1h
* Maximal duration of client commit inactivity: Maximal duration between two commits of an individual client before the removal of the client is proposed by the DS.
  * Default: 90d

## DS state

The DS has a database of EAR encrypted group states indexed by their group ID.

```rust
struct GroupStateDbEntry {
  encrypted_group_state: Vec<u8>,
  timestamp: Timestamp,
  deleted_queues: Vec<SealedQueueConfig>,
}
```

The plaintext group state contains the following data:

* **Public ratchet tree:** The public MLS ratchet trees of the group.
* **MLS GroupInfo:** The [GroupInfo](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#name-adding-members-to-the-group) of this group.
* **User profile:** For each user that has at least one client in the group, the DS keeps the following records.
  * **Clients:** The leaf indices of each client belonging to the user.
  * **User auth key:** Public signature key known to all clients of a given user.
* **Client profile:** For each client in the group, the DS keeps the following records.
  * **Client index:** The client's leaf index in the public group tree.
  * **Client credential chain:** The intermediate client credentials for each of the user's clients, as well as the client's client credential, encrypted using the group state encryption key.
  * **Client queue config:** The [client queue config](glossary.md#sealed-queue-config) of the client.
  * **Activity time:** A timestamp indicating either the time the client was added, or the last time the client has sent a commit (whatever is more recent).
  * **Activity epoch:** Epoch of the last time the client has sent a commit (see activity time).
* **Past group states:** Whenever a new KeyPackage is added to the group, the DS files a copy of the current epoch's group state and keeps it until all group members added in a given, past epoch have updated, or until all KeyPackages added in the given epoch have expired. The copies include the following data:
  * **Public ratchet tree**
  * **EID states (encrypted)**
  * **Intermediate client credentials (encrypted)**
  * **Joining clients:** List of the KeyPackagerefs of all clients that are expected to pick up this group state.
* **Proposal store:** List of proposals sent in this group in this epoch. Gets cleared upon every epoch change.

## Proposal store

* Proposal store is emptied after each successful commit
* Proposals are added to the store either through user self removals, 
* If the proposal store is non-empty, the next commit must be a client update that contains all proposals in the proposal store. The DS must reject all other operations containing commits.
* For all requests containing non-update commits, or commits that do not contain all proposals in the store, the DS will return an error message indicating the the proposal store is non-empty.

## Authentication

Messages from the client to the DS are authenticated by the client by providing a `DsAuthToken`, where the `DsSenderId` in the token depends on the endpoint the client is querying.

```rust
enum DsSenderId {
  LeafIndex(u32),
  KeyPackageRef(KeyPackageRef),
  UserKeyHash(Vec<u8>),
}

struct DsAuthToken {
  group_id: GroupId,
  timestamp: Timestamp,
  sender_id: DsSenderId,
  // TBS: group_id, timestamp and sender_id
  signature: Signature,
}
```

The verification key used to create the token depends on the `sender_id`:

* LeafIndex: Signature key in the leaf's credential
* KeyPackageRef: Signature key of the credential in the KeyPackage with the given KeyPackageRef
* UserKeyHash: Signature key in the user profile indicated by the UserProfileHash

## Report of queue deletion

* Used by the local QS if a fan-out fails due to a deleted queue (either local or federated)
* When receiving such a report, the DS stores the `sealed_queue_config` in the corresponding group state
* Duplicate `sealed_queue_config`s are ignored
* The entries are then later processed by the DS as described [here](./delivery_service.md#removal-due-to-queue-deletion)

```rust
struct {
  group_id: GroupId,
  sealed_queue_config: SealedQueueConfig,
}
```

## Create group

* Client sends the following pieces of data to the DS
  * KeyPackage (representing the creating client)
  * The user's encrypted EID state
  * The creating client's (encrypted) intermediate client credential that was used to sign the leaf credential in the public ratchet tree
  * The client's queue information
  * The initial EAR key (see [here](delivery_service/group_state_encryption.md))
* The DS returns the group ID.

### Authentication

* DSSenderId: LeafIndex

### Future work: Obfuscate group creator

* A leaf index of 0 can be a strong indicator that the client with that index is the original creator of the group.
* It would be good to allow clients to start groups with them in another position than 0.

## Update queue information

* Allows a client to update its queue information.
* Sender has to provide the group's group ID and EAR key.

### Authentication

* DsSenderId: LeafIndex

## Get Welcome information

* Requires the group ID
* Requires the group's EAR key
* Requires the epoch for which the information should be fetched
* Returns the group's public tree, as well as encrypted ICCs and EID states

### Authentication

* DSSenderId: KeyPackageRef

## Get External Commit information

* Requires the group ID
* Requires the group's EAR key
* Returns the group's GroupInfo and public tree, as well as encrypted ICCs and EID states

### Authentication

* DSSenderId: UserKeyHash

## Deliver message

This endpoint allows clients to have MLS messages delivered to all other members of this group. The DS tracks membership of the group by analyzing the MLS commits distributed this way and performs a number of validation steps for each message.

To track group membership and to perform validation functions, the DS requires that messages are sent in the MLS plaintext format. Of course, this is with the exception of MLS application messages, which are sent encrypted as determined by the MLS specification.

To maintain a certain amount of simplicity, the only allows MLS commits that match certain patterns. Each patterns corresponds to a pre-determined high-level group operation such as adding a new user to the group, or removing an individual client of a user that is already a member of the group.

TODO: Note, how the DS can remove users based on expiring KeyPackages by sending a Remove proposal and then requiring the next commit (admin or not) to commit that remove.

### Validation

The DS aims to ensure that it only distributes MLS messages that are accepted by all group members as valid. It does this by performing the same validation steps as the clients using the unencrypted data in the MLS messages.


### Valid group operations

The following group operations and associated commit patterns are considered valid by the DS. Most membership operations also require updates to associated group information such as user profiles or encrypted credentials and EID states. If this is the case, the sender needs to attach the data either in plaintext (user profile updates), or encrypted (EID update commits, EID states and intermediate client credentials).

If not otherwise mentioned, all proposals must be inline proposals.

A newly added user has to send an empty commit with at least one of its clients with a user auth key attached before they can perform other group operations.

#### Adding new users to the group

Operation, where the commit contains one or more inline Add proposals containing clients of one or more new users. The sender has to additionally provide a signature of the user's QS to help the DS validate that the KeyPackages indeed all belong to one user, as well as a timestamp to prove that the KeyPackages were recently obtained.

```rust
struct AddUserParams {
  commit: Commit,
  welcome: Welcome,
  welcome_attribution_info: Vec<WelcomeAttributionInfo>,
  key_package_batches: Vec<KeyPackageBatch>,
}

struct AddUserParamsAad {
  encrypted_credential_information: Vec<Vec<u8>>
}
```

The `commit` must include the `AddUserParamsAad` of all added users in the AAD of the MLSContent, where the ciphertexts in the `encrypted_credential_information` are sorted in the same way as the Add proposals in the `commit`.

This operation can only be performed by clients of users marked as *admin* and all KeyPackages have to contain an extension that contains a [ClientQueueConfig](./glossary.md#sealed-queue-config).

Upon reception, the DS hashes the KeyPackages in all Add proposals contained in the commit. The KeyPackageBatches indicate, which KeyPackages belong to which user. If there are KeyPackages for which there is no matching KeyPackageRef in any KeyPackageBatch, or if there is a KeyPackageRef in a batch that has no corresponding Add proposal, the commit is invalid. Otherwise, the DS creates a user proile for each batch with the leaf indices of the KeyPackages referenced within. The user auth key of the new user remains empty until the first update of one of the user's clients.

The DS also has to verify that the timestamp is not older than the DS' configured maximal KeyPackageBatch age.

Finally, the DS sends the `commit` to the group members by sending them on to its local QS, either for it to forward the the client's federated QS or to a local queue. It also sends [WelcomeBundles](./glossary.md#welcomebundle) to the newly added clients.

##### Authentication

* DsSenderId: UserKeyHash

##### Future work: Tighten up DS validation using Zero-Knowledge proofs

* ZKPs could allow us to verify that the sender of a Welcome sends the correct EID and credential encryption keys
* Alternatively, the recipient of the Welcome could let the DS know that it received a bogus Welcome. The problem here is how the recipient can prove that the Welcome is indeed bogus.

#### Remove users

```rust
struct RemoveUserParams {
  commit: Commit,
}
```

* The commit must exclusively contain Remove proposals
* The commit must contain Remove proposals for all clients of all evicted users
* The sending client must be a client of an admin
* The DS validates the commit and updates its public tree
* The DS removes the user profiles for the evicted users and the encrypted credential information of all of their clients
* Note, that a user can't remove itself due to MLS constraints
* Finally, the DS sends the `commit` to the group members by sending them on to its local QS, either for it to forward the the client's federated QS or to a local queue.

##### Authentication

* DsSenderId: UserKeyHash

#### Updating the sending client's own key material

```rust
struct ClientUpdateParams {
  commit: Commit,
}

struct ClientUpdateParamsAad {
    option_encrypted_credential_information: Option<Vec<u8>>,
}
```

* DS validates the commit and changes its public tree
  * The commit must contain an update path, as well as all pending proposals
  * If the credential in the sender's KeyPackage has changed, there must be encrypted credential information in the AAD
* If there is encrypted client credential information in the commit's AAD, the DS also updates its corresponding state
* Finally, the DS sends the `commit` to the group members by sending them on to its local QS, either for it to forward the the client's federated QS or to a local queue.

##### Authentication

* SenderId: UserKeyHash

##### Future work: Base user secret rotation

* Although the group secret should provide sufficient PCS guarantees after a client removal, users should also be able to rotate their base user secret.
* This is tricky, because a rotation would affect all groups simultaneously.

#### Join group with new client

```rust
struct JoinClientParams {
  external_commit: Commit,
}

struct AddClientParamsAad {
  existing_user_clients: Vec<LeafIndex>,
  encrypted_credential_information: Vec<u8>,
}
```

* The DS verifies that the `existing_user_clients` correspond to the leaf indices in the user profile of the sending user (identified via the UserKeyHash used for authentication)
* An adversary could make the state of clients and DS diverge if it has two users in a group and sends the commit with a client of one user while using the user auth key of the other. The existing_user_clients allows other group members to verify that the adder didn't falsely claim to be another user towards the DS. If the adder did indeed misbehave, other group members can rectify the error as described [here](./delivery_service/broken_state_detection.md).

##### Authentication

* SenderId: UserKeyHash

#### Add own clients

```rust
struct AddClientsParams {
  commit: Commit,
  welcome: Welcome,
  welcome_attribution_info: WelcomeAttributionInfo,
}

struct AddClientsParamsAad {
  encrypted_credential_information: Vec<u8>,
}
```

* The commit must contain exclusively Add proposals
* This endpoint brings the same potential to break the group state as the "Join with new client" endpoint. See there for more information.
* Finally, the DS sends the `commit` to the group members by sending them on to its local QS, either for it to forward the the client's federated QS or to a local queue.
* It also sends [WelcomeBundles](./glossary.md#welcomebundle) to the newly added clients.

##### Authentication

* SenderId: UserKeyHash

#### Remove own clients

```rust
struct RemoveClientParams {
  commit: Commit,
}
```

* The commit must exclusively contain Remove proposals
* The DS validates the commit and updates its public tree
* The DS removes the encrypted credential information of all of removed clients
* Note, that a user can't remove itself due to MLS constraints
* Finally, the DS sends the `commit` to the group members by sending them on to its local QS, either for it to forward the the client's federated QS or to a local queue.

* After removing a client and rotating the user secret of the group, the remover should issue a second commit to add all missing clients to the group

##### Authentication

* SenderId: UserKeyHash

#### ReSync

```rust
struct ResyncClientParams {
  external_commit: Commit,
}
```

* The commit must contain exactly one Add and one Remove proposal referencing the same leaf
* The DS validates the commit and updates its public tree
* The leaf credential of the resynced client must remain the same
* Finally, the DS sends the `commit` to the group members by sending them on to its local QS, either for it to forward the the client's federated QS or to a local queue.

##### Authentication

* SenderId: UserKeyHash

#### Client self remove

```rust
struct ClientSelfRemoveParams {
  remove_proposal: Proposal,
}
```

* The proposal must be a Remove proposal for the sending client
* The sending client must not be the last client of the user
* The DS validates the proposal and stores it in this epoch's proposal store
* Finally, the DS sends the `remove_proposal` to the group members by sending them on to its local QS, either for it to forward the the client's federated QS or to a local queue.

##### Authentication

* SenderId: UserKeyHash

#### User self remove

```rust
struct SelfRemoveParams {
  remove_proposals: Vec<Proposal>,
}
```

* The proposals must be a Remove proposals for all clients of the user
* The DS validates the proposals and stores them in this epoch's proposal store
* Finally, the DS sends the `remove_proposals` to the group members by sending them on to its local QS, either for it to forward the the client's federated QS or to a local queue.
* Once the proposals are committed, the DS performs the same clean up as for the Remove User endpoint

##### Authentication

* SenderId: UserKeyHash

##### Future work: Batch remove proposals

With multiple individual proposals all parties have to verify multiple signatures. Ideally, it would be possible to batch remove proposals such that multiple clients can be removed with one proposal.

## DS-induced removals

* Every time a group is EAR-decrypted, the DS performs the following operations
  * Check if the activity time of one of the clients indicates that the client has passed the maximal duration of client commit inactivity. If that is the case, the DS sends `ClientInactivityRemoval` to all group members that propose the removal of all clients that have passed that duration. It also puts the Proposals into its `proposal_store`.

  ```rust
  struct ClientInactivityRemoval {
    proposals: Vec<Proposal>
  }
  ```

  * Check if the `sealed_queue_configs` vector in the `GroupStateDbEntry` is non-empty. If that is the case, the DS searches the `ClientQueueConfig`s of all clients for matching `SealedQueueConfig`s and distributes the following to all group members for each match. It also puts the Proposals into its `proposal_store`.

  ```rust
  struct QueueDeletionRemoval {
    proposals: Vec<Proposal>
  }
  ```

### Future work: Removal due to client misbehaviour

See [here](./delivery_service/broken_state_detection.md).
