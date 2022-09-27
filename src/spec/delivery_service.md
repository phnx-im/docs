# Delivery Service (DS)

The delivery service keeps track of groups and (pseudonymous) group membership.

TODO: (future work, i.e. long term) For each operation below, include a 'validation' subsection that includes the validation steps performed by the DS.

## DS configuration options

## DS state

The DS has a database of groups indexed by their group ID. Each group has the following pieces of data associated with it.


* **Public ratchet tree:** The public MLS ratchet trees of the group.
* **MLS GroupInfo:** The [GroupInfo](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#name-adding-members-to-the-group) of this group.
* **User profile:** For each user that has at least one client in the group, the DS keeps the following records.
  * **EID state (encrypted)**: The [evolving identity](authentication_service/evolving_identities.md) state of the user, encrypted using the [group state encryption key](delivery_service/group_state_encryption.md).
  * **Clients:** The leaf indices of each client belonging to the user.
  * **User auth key:** Public signature key known to all clients of a given user.
* **Client profile:** For each client in the group, the DS keeps the following records.
  * **Client index:** The client's leaf index in the public group tree.
  * **Intermediate Client Credentials:** The intermediate client credentials for each of the user's clients, encrypted using the group state encryption key.
  * **Encrypted Queue ID:** The queue ID of the client, encrypted by the client under the client's QS' [queue ID encryption key](queuing_service.md#fetch-queue-ID-encryption-key). For the HPKE encryption, the client has to use its own enqueue auth key to perform the HPKE in the asymmetrically authenticated mode. Also, the domain of the DS has to be in the AAD.
  * **Activity time:** A timestamp indicating either the time the client was added, or the last time the client has sent a commit (whatever is more recent).
  * **Activity epoch:** Epoch of the last time the client has sent a commit (see activity time).
* **Past group states:** Whenever a new KeyPackage is added to the group, the DS files a copy of the current epoch's group state and keeps it until all group members added in a given, past epoch have updated, or until all KeyPackages added in the given epoch have expired. The copies include the following data:
  * **Public ratchet tree**
  * **EID states (encrypted)**
  * **Intermediate client credentials (encrypted)**
  * **Joining clients:** List of the KeyPackagerefs of all clients that are expected to pick up this group state.

TODO: Rename (encrypted) queue id to something else (queue config). After all, it contains the encrypted pseudonymous client id, as well as (optionally) the push token key and has the domain attached.
TODO: Introduce push flag (or similar) as a per-user variable that gets forwarded to messages with each fan-out to clients of that user.
TODO: Future work: Allow local clients to replace their encrypted queue id by plaintext.
TODO: How do joining clients know the roles of other group members? They should probably be kept in a group extension.
TODO: Define AS ticket. What gets signed? The encrypted EID state? Or the real one?

### Correlation between group membership and EID state

* The set of a user's clients that's in a group has to be a subset of the clients that are in the user's EID.

## Create group

* Client sends the following pieces of data to the DS
  * KeyPackage (representing the creating client)
  * The user's encrypted EID state
  * The creating client's (encrypted) intermediate client credential that was used to sign the leaf credential in the public ratchet tree
  * The client's queue information
  * The initial EAR key (see [here](delivery_service/group_state_encryption.md))
* The DS returns the group ID.

### Future work: Obfuscate group creator

* A leaf index of 0 can be a strong indicator that the client with that index is the original creator of the group.
* It would be good to allow clients to start groups with them in another position than 0.

## Update queue information

* Allows a client to update its queue information.
* Sender has to provide the group's group ID and EAR key.
* Requires authentication via the sender's leaf key.
* Should be bound via the request's TBS to the given epoch, as well as a nonce.

## Get Welcome information

* Requires the group ID
* Requires the group's EAR key
* Requires the epoch for which the information should be fetched
* Requires the joiner's KeyPackageref and a signature using the signature key of the new joiner
* Returns the group's public tree, as well as encrypted ICCs and EID states

# Get External Commit information

* Requires the group ID
* Requires the group's EAR key
* Requires a signature of the joining user's user auth key.
* Returns the group's GroupInfo and public tree, as well as encrypted ICCs and EID states

TODO: Once we know how user profiles are indexed, the index should be included in the request as well.

## Deliver message

This endpoint allows clients to have MLS messages delivered to all other members of this group. The DS tracks membership of the group by analyzing the MLS commits distributed this way and performs a number of validation steps for each message.

To track group membership and to perform validation functions, the DS requires that messages are sent in the MLS plaintext format. Of course, this is with the exception of MLS application messages, which are sent encrypted as determined by the MLS specification.

To maintain a certain amount of simplicity, the only allows MLS commits that match certain patterns. Each patterns corresponds to a pre-determined high-level group operation such as adding a new user to the group, or removing an individual client of a user that is already a member of the group.

TODO: Note, how the DS can remove users based on expiring KeyPackages by sending a Remove proposal and then requiring the next commit (admin or not) to commit that remove.

### Validation

The DS aims to ensure that it only distributes MLS messages that are accepted by all group members as valid. It does this by performing the same validation steps as the clients using the unencrypted data in the MLS messages.

### Welcome messages

Whenever a client sends a commit that contains an Add proposal, it also has to include a Welcome to the message which the DS can forward to the added client.

If the added client(s) belong to a new user, the sender also has to attach the group ID, as well as the following data, encrypted under the new user's friendship encryption key:

* the adder's (non-pseudonymous) client ID
* the group's EAR key
* a signature over the whole package (Welcome, group ID and client ID using the adder's client credential)

TODO: If a failure to verify the sender's signature leads to the recipient blocking the group, then a user can permanently block another user from joining a given group.

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

#### Adding a new user to the group

Operation, where the commit contains one or more own Add proposals containing all clients of a new user. The sender has to additionally provide a signature of the user's QS to help the DS validate that the KeyPackage indeed all belong to one user, as well as a timestamp to prove that the KeyPackages were recently obtained.

This operation can only be performed by clients of users marked as *admin*.

Note that the receiving clients instead use the user's EID and the encrypted intermediate client credentials to validate this fact.

The sender also has to provide the new user's encrypted EID state, as well as the encrypted, intermediate client credentials of the clients the commit adds.

The DS then creates a new user profile for the user, leaving the user auth key blank until the user's first update.

#### Removing one or more users

#### Resync of an existing client