# Delivery Service (DS)

The delivery service keeps track of groups and (pseudonymous) group membership.

TODO: (future work, i.e. long term) For each operation below, include a 'validation' subsection that includes the validation steps performed by the DS.
TODO: Remove mention of EID (which is future work for now) and replace mention of the ICC with an ICC + CC bundle.

## DS configuration options

* Maximal KeyPackageBatch age: Maximal difference between a KeyPackageBatch timestamp and the current time when a new user is added to a group.
  * Default: 1h

## DS state

The DS has a database of groups indexed by their group ID. Each group has the following pieces of data associated with it.

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

TODO: Define AS ticket. What gets signed? The encrypted EID state? Or the real one?

### Correlation between group membership and EID state

* The set of a user's clients that's in a group has to be a subset of the clients that are in the user's EID.

### Authentication

```rust
enum SenderId {
  LeafIndex(u32),
  KeyPackageRef(KeyPackageRef),
  UserKeyHash(Vec<u8>),
}

struct AuthToken {
  group_id: GroupId,
  timestamp: Timestamp,
  sender_id: SenderId,
  // TBS: group_id, timestamp and sender_id
  signature: Signature,
}
```

All requests to the DS have to include a signature over the AuthToken struct, where the verification key depends on the sender_id:

* LeafIndex: Signature key in the leaf's credential
* KeyPackageRef: Signature key of the credential in the KeyPackage with the given KeyPackageRef
* UserKeyHash: Signature key in the user profile indicated by the UserProfileHash

## Create group

* Client sends the following pieces of data to the DS
  * KeyPackage (representing the creating client)
  * The user's encrypted EID state
  * The creating client's (encrypted) intermediate client credential that was used to sign the leaf credential in the public ratchet tree
  * The client's queue information
  * The initial EAR key (see [here](delivery_service/group_state_encryption.md))
* The DS returns the group ID.

### Authentication

* SenderId: LeafIndex

### Future work: Obfuscate group creator

* A leaf index of 0 can be a strong indicator that the client with that index is the original creator of the group.
* It would be good to allow clients to start groups with them in another position than 0.

## Update queue information

* Allows a client to update its queue information.
* Sender has to provide the group's group ID and EAR key.

### Authentication

* SenderId: LeafIndex

## Get Welcome information

* Requires the group ID
* Requires the group's EAR key
* Requires the epoch for which the information should be fetched
* Returns the group's public tree, as well as encrypted ICCs and EID states

### Authentication

* SenderId: KeyPackageRef

# Get External Commit information

* Requires the group ID
* Requires the group's EAR key
* Returns the group's GroupInfo and public tree, as well as encrypted ICCs and EID states

### Authentication

* SenderId: UserKeyHash

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

* Generally if new clients are added, the adder needs to attach their [encrypted intermediate client credentials](delivery_service/group_state_encryption.md#credential-encryption).
* The DS updates the public group state according to the information in the commit, as well as the various encrypted pieces of data attached to the message.
* The DS sends the messages to its local QS to forward to the various QS' of the group members and attaches the (encrypted) queue information.
* Generally if clients are added or removed, the commit needs to be accompanied by one or more different messages (depending on the scenario)
  * If a client's client credential or intermediate client credential is updated, or if a new client is added: a new encrypted intermediate client credential (also, the commit needs to contain the add (if it's a new client) or an update to the client's leaf)
  * If a client's client credential is updated: additionally one or more commits containing changes to the user's EID state (encrypted under a group key, maybe as application messages), as well as an updated, encrypted EID state for the user
  * If a client is removed: One or more commits containing changes to the user's EID state, as well as an updated, encrypted EID state for the user

TODO: How to encrypt commits that detail changes to a user's EID? They should be sent in the same message as the commit containing the corresponding change to the group's membership.

TODO: Note that the sender also has to include the signature over the GroupInfo.

#### Adding new users to the group

Operation, where the commit contains one or more own Add proposals containing clients of one or more new users. The sender has to additionally provide a signature of the user's QS to help the DS validate that the KeyPackages indeed all belong to one user, as well as a timestamp to prove that the KeyPackages were recently obtained.

```rust
struct AddUserParams {
  commit: Commit,
  welcome: Welcome,
  welcome_attribution_info: Vec<WelcomeAttributionInfo>,
  key_package_batches: Vec<KeyPackageBatch>,
  encrypted_credential_information: Vec<u8>,
}
```

This operation can only be performed by clients of users marked as *admin* and all KeyPackages have to contain an extension that contains a [ClientQueueConfig](./glossary.md#sealed-queue-config).

Upon reception, the DS hashes the KeyPackages in all Add proposals contained in the commit. The KeyPackageBatches indicate, which KeyPackages belong to which user. If there are KeyPackages for which there is no matching KeyPackageRef in any KeyPackageBatch, or if there is a KeyPackageRef in a batch that has no corresponding Add proposal, the commit is invalid. Otherwise, the DS creates a user proile for each batch with the leaf indices of the KeyPackages referenced within. The user auth key of the new user remains empty until the first update of one of the user's clients.

The DS also has to verify that the timestamp is not older than the DS' configured maximal KeyPackageBatch age.

##### Future work: Tighten up DS validation using Zero-Knowledge proofs

* ZKPs could allow us to verify that the sender of a Welcome sends the correct EID and credential encryption keys
* Alternatively, the recipient of the Welcome could let the DS know that it received a bogus Welcome. The problem here is how the recipient can prove that the Welcome is indeed bogus.

#### Updating the sending client's own key material

* If intermediate client cred changes, the sender has to provide the new cred in encrypted form.
* If the client cred changes, the sender has to provide the a new encrypted intermediate cred, as well as a new leaf cred, the EID update commit and the new, encrypted EID state.

##### Future work: Base user secret rotation

* Although the group secret should provide sufficient PCS guarantees after a client removal, users should also be able to rotate their base user secret.
* This is tricky, because a rotation would affect all groups simultaneously.

#### Managing own clients

Operation, where the commit contains one or more own Add and/or Remove proposals for own clients. The sender additionally authenticates the fact that the KeyPackages in the Add proposals, are indeed its clients and that it is authorized to remove the indicated clients by providing a signature using the sending user's user auth key.

The sender also has to provide the EID commits (encrypted, for forwarding to other group members), its new, encrypted EID state, as well as the encrypted, intermediate client credentials of the clients the commit adds.

Using the information in the commit, the DS updates its public group state and

##### Future work: Optimize distribution of EID states and changes

TODO: Do we want to require that the EID commits are sent into *every* group? That might be a bit excessive.

TODO: Instead of having a user auth key, we could also rely on a signature by the QS like when adding a user. This gives us slightly tighter guarantees, because a user can't just place someone else's client into the group claiming (towards the DS) that it's actually its own. (This doesn't quite work, but maybe we can make it work somehow.)

#### Removing one or more users

#### Resync of an existing client