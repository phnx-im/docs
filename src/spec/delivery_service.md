# Delivery Service (DS)

The delivery service keeps track of groups and (pseudonymous) group membership.

TODO: For each operation below, include a 'validation' subsection that includes the validation steps performed by the DS.

## DS configuration options

## DS state

The DS has a database of groups indexed by their group ID. Each group has the following pieces of data associated with it.

TODO: The DS needs to keep old public trees around for new members.

TODO: The DS has to keep track of indices in the user profile.

TODO: Put domain into group ID.

* **Public ratchet tree:** The public MLS ratchet tree of the group.
* **MLS GroupInfo:** The [GroupInfo](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#name-adding-members-to-the-group) of this group.
* **User profile:** For each user that has at least one client in the group, the DS keeps the following records.
  * **EID state**: The [evolving identity](authentication_service/evolving_identities.md) state of the user, encrypted using the [group state encryption key](delivery_service/group_state_encryption.md).
  * **Clients:** The leaf indices of each client belonging to the user.
  * **Role:** The user's role, i.e. either member or admin.
  * **User auth key:** Public signature key known to all clients of a given user.
* **Client profile:** For each client in the group, the DS keeps the following records.
  * **Intermediate Client Credentials:** The intermediate client credentials for each of the user's clients, encrypted using the group state encryption key.
  * **Queue information:** Information needed to deliver messages to the client's queue.
    * **Encrypted Queue ID:** The queue ID of the client, encrypted by the client under the client's QS' [queue ID encryption key](queuing_service.md#fetch-queue-ID-encryption-key). For the HPKE encryption, the client has to use its own enqueue auth key to perform the HPKE in the asymmetrically authenticated mode. Also, the domain of the DS has to be in the AAD.

## Create group

* Client sends the following pieces of data to the DS
  * An initial public ratchet tree that can contain only the creating client.
  * The user's encrypted EID state
  * The intermediate creating client's intermediate client credential that was used to sign the leaf credential in the public ratchet tree
  * The client's queue information
  * The initial EAR key (see [here](delivery_service/group_state_encryption.md))

## Update queue information

* Allows a client to update its queue information.
* Requires authentication via the sender's leaf key.

## Deliver message

TODO: Do we allow non inline proposals at all? If so, the DS has to store them along with the group state.

* A message can be any MLS message.
* Commits and proposals have to be in the MLS plaintext framing
* If it's the first commit of a user since it has been added to the group, it attaches a user auth key.
* Generally if new clients are added, the adder needs to attach their [encrypted intermediate client credentials](delivery_service/group_state_encryption.md#credential-encryption).
* The DS updates the public group state according to the information in the commit, as well as the various encrypted pieces of data attached to the message.
* The DS sends the messages to its local QS to forward to the various QS' of the group members and attaches the (encrypted) queue information.

### Adding a new user

TODO: When adding a bunch of clients from different users, how does the adder signal what subset of the clients belong to which user. Similarly, what if the adder (an admin) adds a new client of themselves, as well as a new user? We should probably restrict this for the PoC.

* This is the case when the commit contains adds that contains key packages that are not associated with an existing group member
* Only group admins are allowed to do this.
* Adder creates a commit that adds the new user via a KeyPackage.
* Adder attaches to the commit the encrypted EID state of the user, along with the encrypted client credentials for each of the user's clients.
* Adder sends the Welcome in their the connection group with the new user.
* The DS leaves the user auth key blank
* The DS retrieves the queue information from the corresponding key package extension.

### Adding a new client of an existing user

* This is the case if the commit contains an add that adds a KeyPackage for an existing user. The sender signals that by indicating the change in the user profile.
* Needs auth via user auth key (not possible of the user auth key is blank)
* New clients of an existing user can join via External Commit.

### Removing own client

* Needs auth via user auth key (not possible of the user auth key is blank)
* Whenever an own client is removed, the user auth key needs to be cycled.
* The new key is derived separately for each group. It is derived by combining the base user secret and an exported secret from the first epoch after the client was removed
* Future work: Rotate the base user secret

TODO: Create a documentation page that keeps track of all the state that clients need to sync and keep track of.

### Adding new own client

* Needs auth via user auth key

TODO: The user profile also has to be updated with every add.

