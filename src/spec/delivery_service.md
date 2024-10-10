# Delivery Service (DS)

The delivery service keeps track of groups and (pseudonymous) group membership and delivers messages to group members.

## DS configuration options

* Maximal DSAuthToken age: Maximal age of an [DSAuthToken](delivery_service.md#authentication) presented to the DS for client authentication.
  * Default: 1h
* Maximal duration of client commit inactivity: Maximal duration between two commits of an individual client before the removal of the client is proposed by the DS.
  * Default: 90d
* Padding target (in percentiles (1..10)): When padding the size of groups for encryption at rest, this determines the size of the padding. The largest group on the DS in the given percentile (ordered by number of members per group) determines the smallest size that groups are padded to before encryption. As a result, after encryption, all groups look at least as large as that group. Larger groups are padded to the closes multiple of that size. The actual padding is then computed from the group's ciphersuite, as well as the size of the credentials.
  * Default: 8

## DS state

The DS has a database of EAR encrypted group states indexed by their group ID. Alongside each encrypted group state, the DS stores some metadata such as the timestamp indicating when the group was last used, as well as a list of encrypted queue IDs.

```rust
struct GroupStateDbEntry {
  encrypted_group_state: Vec<u8>,
  timestamp: Timestamp,
  queue_id_encryption_key: QueueIdEncryptionKey,
  deleted_queues: Vec<Vec<u8>>,
}
```

The plaintext group state contains the following data:

* **Public ratchet tree:** The public MLS ratchet trees of the group.
* **MLS GroupInfo:** The [GroupInfo](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#name-adding-members-to-the-group) of this group.
* **User profile:** For each user in the group, the DS keeps the following records.
  * **Client:** The leaf index of the client belonging to the user.
* **Client profile:** For each client in the group, the DS keeps the following records.
  * **Client index:** The client's leaf index in the public group tree.
  * **Client credential (encrypted):** The client's client credential, encrypted using the group state encryption key.
  * **Client queue config:** The [client queue config](glossary.md#sealed-queue-config) of the client.
  * **Activity time:** A timestamp indicating either the time the client was added, or the last time the client has sent a commit (whatever is more recent).
  * **Activity epoch:** Epoch of the last time the client has sent a commit (see activity time).
* **Past group states:** Whenever a new KeyPackage is added to the group, the DS files a copy of the current epoch's group state and keeps it until all group members added in a given, past epoch have updated, or until all KeyPackages added in the given epoch have expired. The copies include the following data:
  * **Public ratchet tree**
  * **Joining clients:** List of the KeyPackagerefs of all clients that are expected to pick up this group state.
* **Proposal store:** List of proposals sent in this group in this epoch. Gets cleared upon every epoch change.
* **Queue ID decryption key:** The private key corresponding to the QueueIdEncryptionKey stored alongside the encrypted group state.

## Proposal store

* The Proposal store is emptied after each successful commit
* Proposals are added to the store either through user self removals,
* If the proposal store is non-empty, the next commit must be a client update that contains all proposals in the proposal store. The DS must reject all other operations containing commits.
* For all requests containing non-update commits, or commits that do not contain all proposals in the store, the DS will return an error message indicating the the proposal store is non-empty.

## Authentication

Messages from the client to the DS are authenticated by the client by providing a `DsAuthToken`, where the `DsSenderId` in the token depends on the endpoint the client is querying.

```rust
enum DsSenderId {
  LeafIndex(u32),
  KeyPackageRef(KeyPackageRef),
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

All endpoints additionally require the client to submit a privacy pass token valid under the privacy pass public key of the AS of the same domain as the DS.

## GroupInfo updates

When sending a commit to change the group state, the sender has to enable the DS to update the MLSGroupInfo of the group. Since the DS either already has most of the required data and can extract the rest from the message, the sender only has to include its signature over the new GroupInfo.

```rust
struct GroupInfoUpdate {
  signature: Signature,
}
```

## Message delivery

The majority of endpoints of the DS allow clients to send MLS messages to groups.

### Validation

Whenever receiving an MLS message at any endpoint, the DS checks if a local group state with the message's GroupId exists. If it does, the DS locks the corresponding database entry to prevent concurrent access. The DS then takes the GroupStateEarKey that is part of every message delivery request and decrypts the group state. Finally, the DS performs the same validation steps a receiving MLS client would perform, as well as the endpoint-specific validation steps.

The MLS client checks also includes checking that the epoch numbers match. In the case of commits, this comparison along with the lock on the database entry ensure that any conflicts between commits for the same epoch are resolved.

If all validation steps pass, the DS performs the endpoint and message-specific changes to its local group state.

### Message distribution

To distribute MLS messages the DS sends messages on to the local QS to enqueue in a local client's queue or to forward to a federated QS. For the message format see [here](glossary.md#fan-out-message).

In cases where the QS responds with a message indicating that a target queue doesn't exist, the DS assumes that the queue was deleted and that the associated group member should be removed. Since the group state has already been re-encrypted by the time the QS can respond, the DS encrypts any returned [SealedQueueConfigs](./glossary.md#sealed-queue-config) under the group's [QueueIdEncryptionKey](./glossary.md#queue-id-encryption-key). The corresponding private key is stored encrypted as part of the group state.

When the group state is next decrypted for message processing, so are the sealed queue configs. For each sealed queue config, the DS locates the associated group members and proposes their removal as specified [here](delivery_service.md#ds-induced-removals).

## Activity times

The DS tracks both the last time individual group members were active (encrypted at rest as part of the group state), as well as the last time the group in general was actively used. A timestamp consists only of the month and the year of last activity.
Whenever a client sends a commit as part of a query to an endpoint, the DS updates the *activity time* and *activity epoch* of the sender.

## Client endpoints

Endpoints meant to be accessed by clients registered with the homeserver via HTTP requests.

### Request group id

* Endpoint: `ENDPOINT_DS_GROUP_ID`

Request a fresh group id for use with the [create group](delivery_service.md#create-group) endpoint. The DS samples a fresh group id, checks for collisions and, if none are found, enters the group id as a placeholder into the database. If a collision is found, the DS re-samples until there are no collisions.

```rust
struct RequestGroupIdResponse {
  group_id: GroupId,
}
```

#### Authentication

* None

### Create group

* Endpoint: `ENDPOINT_DS_CREATE_GROUP`

```rust
struct CreateGroupParams {
  group_id: GroupId,
  key_package: KeyPackage,
  encrypted_credential_chain: Vec<u8>,
  creator_queue_config: ClientQueueConfig,
  group_info: GroupInfo,
  initial_ear_key: GroupStateEarKey,
}
```

* The DS checks if there is a placeholder in the group database for this group id. If there is, it creates the GroupStateDbEntry.

#### Authentication

* DsSenderId: LeafIndex

### Update queue information

* Endpoint: `ENDPOINT_UPDATE_QUEUE_INFO`

```rust
struct UpdateQueueInfoParams {
  group_id: GroupId,
  group_state_ear_key: GroupStateEarKey,
  new_queue_config: ClientQueueConfig,
}
```

#### Authentication

* DsSenderId: LeafIndex

### Get Welcome information

* Endpoint: `ENDPOINT_DS_WELCOME_INFO`

```rust
struct WelcomeInfoParams {
  group_id: GroupId,
  group_state_ear_key: GroupStateEarKey,
  epoch: Epoch,
}

struct WelcomeInfoResponse {
  public_tree: MlsRatchetTree,
  credential_chains: Vec<u8>,
}
```

#### Authentication

* DSSenderId: KeyPackageRef

### Get External Commit information

* Endpoint: `ENDPOINT_DS_EXTERNAL_COMMIT_INFO`

```rust
struct ExternalCommitInfoParams {
  group_id: GroupId,
  group_state_ear_key: GroupStateEarKey,
}

struct WelcomeInfoResponse {
  group_info: GroupInfo,
  public_tree: MlsRatchetTree,
  credential_chains: Vec<u8>,
}
```

#### Authentication

* DSSenderId: LeafIndex


### Adding new users to the group

* Endpoint: `ENDPOINT_DS_ADD_USERS`

Operation, where the commit contains one or more inline Add proposals containing the client(s) of one or more new users.

```rust
struct AddUsersParams {
  commit: MlsMessage,
  group_state_ear_key: GroupStateEarKey,
  group_info_update: GroupInfoUpdate,
  welcome: Welcome,
  welcome_attribution_info: Vec<WelcomeAttributionInfo>,
  key_package: Vec<KeyPackage>,
}

struct AddUsersParamsAad {
  encrypted_credential_information: Vec<Vec<u8>>
}
```

The `commit` must include the `AddUserParamsAad` of all added users in the AAD of the MLSContent, where the ciphertexts in the `encrypted_credential_information` are sorted in the same way as the Add proposals in the `commit`.

This operation can only be performed by clients of users marked as *admin* and all KeyPackages have to contain an extension that contains a [ClientQueueConfig](./glossary.md#sealed-queue-config).

The DS also has to verify that the timestamp is not older than the DS' configured maximal KeyPackage age.

Finally, the DS sends the `commit` to the group members by sending them on to its local QS, either for it to forward the the client's federated QS or to a local queue. It also sends [WelcomeBundles](./glossary.md#welcomebundle) to the newly added clients.

#### Authentication

* DsSenderId: LeafIndex

### Remove users

* Endpoint: `ENDPOINT_DS_REMOVE_USERS`

```rust
struct RemoveUserParams {
  commit: MlsMessage,
  group_state_ear_key: GroupStateEarKey,
  group_info_update: GroupInfoUpdate,
}
```

* The commit must exclusively contain Remove proposals
* The sending client must be a client of an admin
* The DS validates the commit and updates its public tree
* The DS removes the user profiles for the evicted users and the encrypted credential information of all of their clients
* Note, that a user can't remove itself due to MLS constraints
* Finally, the DS sends the `commit` to the group members by sending them on to its local QS, either for it to forward the the client's federated QS or to a local queue.

#### Authentication

* DsSenderId: LeafIndex

### Updating the sending client's own key material

* Endpoint: `ENDPOINT_DS_UPDATE_CLIENT`

```rust
struct UpdateClientParams {
  commit: MlsMessage,
  group_state_ear_key: GroupStateEarKey,
  group_info_update: GroupInfoUpdate,
}

struct UpdateClientParamsAad {
    option_encrypted_credential_information: Option<Vec<u8>>,
}
```

* DS validates the commit and changes its public tree
  * The commit must contain an update path, as well as all pending proposals
  * If the credential in the sender's KeyPackage has changed, there must be encrypted credential information in the AAD
* If there is encrypted client credential information in the commit's AAD, the DS also updates its corresponding state
* If a remove proposal is committed as part of the commit, the DS removes the associated client and user profiles.
* If the KeyPackageRef of the updating client (prior to applying the update) is in one of the *joining clients* vectors in the group's storage of old group states, the DS removes that KeyPackageRef from the vector. If this leaves the vector empty, the DS removes this particular copy of the group state.
* Finally, the DS sends the `commit` to the group members by sending them on to its local QS, either for it to forward the the client's federated QS or to a local queue.

#### Authentication

* SenderId: LeafIndex

### Join connection group

* Endpoint: `ENDPOINT_DS_JOIN_CONNECTION_GROUP`

```rust
struct JoinConnectionGroupParams {
  external_commit: MlsMessage,
  group_state_ear_key: GroupStateEarKey,
  group_info_update: GroupInfoUpdate,
}

struct JoinConnectionGroupParamsAad {
  encrypted_credential_information: Vec<u8>,
}
```

* The DS checks if the group contains only a single user. Every single-user group can be joined as a connection group.

#### Authentication

No additional authentication is required for this endpoint. The knowledge of the group's EAR key effectively authenticates the joining client.

### ReSync

* Endpoint: `ENDPOINT_DS_RESYNC_CLIENT`

```rust
struct ResyncClientParams {
  external_commit: MlsMessage,
  group_state_ear_key: GroupStateEarKey,
  group_info_update: GroupInfoUpdate,
}
```

* The commit must contain exactly one Add and one Remove proposal referencing the same leaf
* The DS validates the commit and updates its public tree
* The leaf credential of the re-synced client must remain the same
* Finally, the DS sends the `commit` to the group members by sending them on to its local QS, either for it to forward the the client's federated QS or to a local queue.

#### Authentication

* SenderId: LeafIndex

### User self remove

* Endpoint: `ENDPOINT_DS_SELF_REMOVE_USER`

```rust
struct SelfRemoveUserParams {
  remove_proposal: MlsMessage,
  group_state_ear_key: GroupStateEarKey,
}
```

* The proposal must be a Remove proposal for the clients of the user
* The DS validates the proposal and stores it in this epoch's proposal store
* Finally, the DS sends the `remove_proposal` to the group members by sending it on to its local QS, either for it to forward the the client's federated QS or to a local queue.
* Once the proposal are committed, the DS performs the same clean up as for the Remove User endpoint

#### Authentication

* SenderId: LeafIndex

### Send application message

* Endpoint: `ENDPOINT_DS_SEND_MESSAGE`

```rust
struct SendMessageParams {
  application_message: MlsMessage,
  group_state_ear_key: GroupStateEarKey,
}
```

* The DS sends the `application_message` to the group members by sending them on to its local QS, either for it to forward the the client's federated QS or to a local queue.

#### Authentication

* SenderId: LeafIndex

### Delete group

* Endpoint: `ENDPOINT_DS_DELETE_GROUP`

```rust
struct DeleteGroupParams {
  commit: MlsMessage,
  group_state_ear_key: GroupStateEarKey,
}
```

* The commit must contain Remove proposals for all group members except for the sending client
* Finally, the DS sends the `commit` to the group members by sending them on to its local QS, either for it to forward the the client's federated QS or to a local queue.
* After sending out the commit, the DS deletes the group state.

#### Authentication

* SenderId: LeafIndex

## DS-induced removals

In some situations, the DS will mandate the removal of a given group user by adding a remove proposal to the group's [proposal store](delivery_service.md#proposal-store). Every time, a group state is EAR-decrypted during to process a request, the DS performs the following operations:

Activity time: If the activity time of one of the clients indicates that the client has passed the maximal duration of client commit inactivity, the DS sends a `ClientInactivityRemoval` to all group members that proposes the removal of all clients that have passed that duration. It also puts the Proposals into the `proposal_store` of the group.

```rust
struct ClientInactivityRemoval {
  proposals: Vec<MlsMessage>
}
```

Removed queues: If the `sealed_queue_configs` vector in the `GroupStateDbEntry` is non-empty, the DS searches the `ClientQueueConfig`s of all clients for matching `SealedQueueConfig`s and distributes the following to all group members for each match. It also puts the Proposals into the `proposal_store` of the group.

```rust
struct QueueDeletionRemoval {
  proposals: Vec<MlsMessage>
}
```
