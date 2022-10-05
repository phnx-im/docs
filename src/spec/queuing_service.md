# Queuing Service (QS)

The main purpose of the queuing service is to store-and-forward messages delivered by the [delivery service](./delivery_service.md), as well as the publication of KeyPackages. In this chapter we discuss the various concepts used by the QS to provide its functionalitites, as well as its individual endpoints.

## Overview

The store-and-forwards functionality of the QS is designed in such a way that the QS cannot link the state it maintains for each of the homeserver's users with the user's actual identity as maintained by the AS.

To avoid a connection with the user's actual identity, each user has a pseudonymous user record with a sub-record for each of the user's clients. The records are created by the user's clients when they register with the AS.

After creation, the user's clients can publish KeyPackages and fetch messages from their queue, as well as rotate the key material used for authentication, or for at-rest encryption of queued messages.

The user record also contains the user's [friendship token](glossary.md#friendship-token), which the QS uses to authenticate requests for batches of the user's KeyPackages.

The client-specific records contain the client's fan-out queue (for messages received from a local or a federated DS), as well as the KeyPackages published by the client. See [here](queuing_service/keypackage_publication.md) for more information on KeyPackage publication and how unlinkability between a user's real identity and its pseudonym is maintained.

The fan-out queue in turn contains an optional push-token (encrypted at-rest under the owning client's [push-token encryption key](glossary.md#push)), as well as key material used for the [queue's at-rest encryption](queuing_service/queue_encryption.md).

## QS state

The QS keeps the following state.

* **Pseudonymous user records:** Indexed by a [PUID](glossary.md#pseudonymous-user-id-puid), each record contains a number of sub fields.
  * **User record auth key:** [Public signature key](glossary.md#user-record-auth-key) used by a user to authenticate itself as the owner of this record.
  * **Friendship token:** Clients have to provide the friendship token to obtain a bundle of KeyPackages for a given user.
  * **Client specific records:** For each of the user's clients, the QS keeps the following state. Indexed by a [PCID](glossary.md#pseudonymous-client-id-pcid).
    * **Activity time:** Timestamp indicating the last time a client has fetched messages from the queue.
    * **KeyPackages:** Encrypted KeyPackages, each with an encrypted [client credential chain](./glossary.md#client-credential-chain) attached. At least one KeyPackage has to be marked as KeyPackage of last resort.
    * **Public client record key:** [Public signature key](glossary.md#client-record-auth-key). Authenticates the owner of this client record and authorizes them to dequeue (i.e. fetch and delete) messages in the queue, as well as to change the queue configuration such as the authentication keys, or to add entries to the block list. Also authorizes the client to upload KeyPackages.
    * **Optional push token:** The client's (optional) push token, stored encrypted at rest under the client's push token encryption key. Used by the QS to create a push notification for the client if a message is enqueued that includes the required encryption key.
    * **Fan-out queue:** A fan-out queue with a number of further record fields attached.
      * **Queue encryption key material:** Key material to perform [queue encryption](./queuing_service/queue_encryption.md).
        * **Queue encryption key:** [HPKE public key](glossary.md#queue-encryption-key) of the queue owner.
        * **Encryption ratchet key:** Symmetric key used to derive queue encryption keys.
      * **Current sequence number:** The current message sequence number.
      * **Queued messages:** A sequence of ciphertexts containing the messages in the queue. Each incoming message is [encrypted](./queuing_service/queue_encryption.md) and is assigned the current sequence number, after which the current sequence number is incremented..
      * **Blocklist salt:** Salt that is used when hashing values for the *Group blocklist*. Generated randomly by the QS upon queue creation.
      * **Blocklist:** List of salted group ID hashes that the QS should not accept messages from for this queue.
* **Queue ID encryption keypair:** [A public/private HPKE keypair](glossary.md#queueconfig-encryption-key) that clients can encrypt their queue ID under before providing it to a local or federated DS.
* **QS signing key:** [A public/private signature keypair](./glossary.md#qs-signing-key) that the QS uses to sign KeyPackage bundles before returning them upon request. Also used to sign messages when forwarding them from the local DS to a remote QS.
* **QS-to-QS queues:** A database of queues indexed by the remote QS' domain. Each queue has the same queue encryption key material attached as the client queues.

## QS configuration options

* Maximal InterQsAuthToken age: Maximal age of an [InterQsAuthToken](./queuing_service.md#inter-qs-authentication) presented to the QS for inter-QS authentication.
  * Default: 1h
* Maximal client record age: Maximal age of an inactive client record.
  * Default: 90d
* Maximal number of requested messages: Maximal number of messages that will be returned to a client requesting messages from a queue.


## Authentication

Messages from the client to the QS are authenticated by the client by providing an QSAuthToken, where the QSSenderId in the token depends on the endpoint the client is querying.

```rust
enum QsSenderId {
  Puid(Puid),
  Pcid(Pcid),
}

struct QsAuthToken {
  sender_id: QsSenderId,
  timestamp: Timestamp,
  // TBS: sender_id and timestamp
  signature: Signature,
}
```

The verification key used to create the token depends on the sender_id:

* Puid: [User record auth key](./glossary.md#user-record-auth-key)
* Pcid: [Client record auth key](./glossary.md#client-record-auth-key)

## Federation endpoints

Endpoints publicly accessible and meant to be accessed by federated homeservers.

### Fetch QS signing key

A DS or QS can fetch the [QS' signing key](./glossary.md#qs-signing-key) through this endpoint.

### Federated enqueue message

A remote QS can enqueue a message by providing the QS with an encrypted pseudonymous client ID, as well as a message to enqueue in the queue specified by the encrypted ID. It decrypts the ciphertext and checks if the group ID given in the message is in the queue's blocklist. If it isn't, it enqueues the message.

```rust
struct FederatedEnqueueMessageParams {
  client_queue_config: ClientQueueConfig,
  message: FanOutMessage,
}
```

#### Inter-QS Authentication

For each query the sending QS has to provide an InterQsAuthToken signed with its QS signing key.

```rust
struct InterQsAuthToken {
  timetamp: Timestamp,
  // TBS: timestamp
  signature: Signature,
}
```

## Client endpoints

Endpoints accessible to clients of the homeserver.

### Fetch queue config encryption key

Clients can fetch the QS' [queue config encryption key](./glossary.md#queueconfig-encryption-key) through this endpoint.

### User record management endpoints

Endpoints for management of pseudonymous and non-pseudonymous user records. Note that pseudonymous user record is deleted with its last client record.

#### Create new pseudonymous user record

Create a new pseudonymous user record, as well as a first client record.

```rust
struct CreateUserRecordParams {
  user_record_auth_key: SignaturePublicKey,
  friendship_token: FriendshipToken,
  client_record_auth_key: SignaturePublicKey,
  queue_encryption_key: HpkePublicKey,
}
```

The QS creates the user record and client record, indexed by a freshly sampled Puid and Pcid. The QS returns both Puid and Pcid.

#### Edit pseudonymous user record

Edit a given user record, overwriting the existing values with the one given in the message.

```rust
struct EditUserRecordParams {
  puid: Puid,
  user_record_auth_key: SignaturePublicKey,
  friendship_token: FriendshipToken,
}
```

##### Authentication

* QSSenderId: Puid

#### Get pseudonymous user record

Get the data associated with a given user record.

```rust
struct GetUserRecordParams {
  puid: Puid,
}
```

The QS returns the following.

```rust
struct GetUserRecordResponse {
  user_record_auth_key: SignaturePublicKey,
  friendship_token: FriendshipToken,
  client_records: Vec<Pcid>,
}
```

##### Authentication

* QSSenderId: Puid

#### Delete pseudonymous user record

Delete the given user record including all associated client records.

```rust
struct DeleteUserRecordParams {
  puid: Puid,
}
```

##### Authentication

* QSSenderId: Puid

##### Future work: MFA for user or client record deletion

User and client deletion are very destructive operations. We should probably require MFA for the associated operations.

### Client record management

#### Create new client record

Create a new client record with the given data.

```rust
struct CreateClientRecordParams {
  client_record_auth_key: SignaturePublicKey,
  queue_encryption_key: HpkePublicKey,
}
```

The QS creates the record indexed by a freshly sampled Pcid and returns the Pcid.

##### Authentication

* QSSenderId: Puid

#### Edit client record

Overwrite the data of the client record with the given PCID with the given data.

```rust
struct EditClientRecordParams {
  pcid: Pcid,
  client_record_auth_key: SignaturePublicKey,
  queue_encryption_key: HpkePublicKey,
  blocklist_entries: Vec<GroupId>,
}
```

##### Authentication

* QSSenderId: Pcid

#### Get client record

Get the data associated with the client record with the given PCID.

```rust
struct GetClientRecordParams {
  pcid: Pcid,
}
```

The QS returns the following data.

```rust
struct GetClientRecordResponse {
  client_record_auth_key: SignaturePublicKey,
  queue_encryption_key: HpkePublicKey,
  blocklist_entries: Vec<GroupId>,
}
```

##### Authentication

* QSSenderId: Pcid

#### Delete client record

Delete the client record with the given PCID. The last client in a user record can only be deleted by deleting the user record itself.

```rust
struct DeleteClientRecordParams {
  pcid: Pcid,
}
```

##### Authentication

* QSSenderId: Puid

#### Publish KeyPackages

Publish the given [KeyPackageTuple](glossary.md#keypackage-tuple) under the given PCID.

All of the KeyPackages contained in the tuples have to contain a [QueueConfigExtension](glossary.md#queueconfig-extension) and at least one of the KeyPackageTuple has to contain a KeyPackage marked as [KeyPackage of last resort](glossary.md#last-resort-extension).

```rust
struct PublishKeyPackagesParams {
  pcid: Pcid,
  key_package_tuples: Vec<KeyPackageTuple>,
}
```

The QS deletes all existing KeyPackages before publishing the new ones.


##### Authentication

* QSSenderId: Pcid

##### Future work: Allow more granular KeyPackage rotation

Throwing all KeyPackages away regardless of their remaining validity is a bit wasteful. A client should have more granular control over which KeyPackages it wants to remain on the QS.

##### Future work: More than one last-resort KeyPackage

Using the same KeyPackage of last resort in multiple groups can allow a federated DS to track the user across these groups. This could be mitigated somewhat by having multiple KeyPackages of last resort that the QS can cycle through when there are no other KeyPackages left.

#### Get client KeyPackage

Get the KeyPackage of the client with the given PCID. This allows clients of a user to fetch individual KeyPackagesTuples for other clients of the same user. These individual KeyPackageTuples are required to add new clients to existing groups.

```rust
struct GetClientKeyPackageParams {
  pcid: Pcid,
}
```

The QS returns one of the client's KeyPackageTuples and deletes the KeyPackageTuple afterwards (except if it contains a KeyPackage of last resort.)

##### Authentication

* QSSenderId: Puid

#### Get KeyPackage batch

Get a [KeyPackageBatch](glossary.md#user-keypackage-batch) of the user with the given PUID.

```rust
struct GetKeyPackageBatchParams {
  puid: Puid,
  friendship_token: FriendshipToken,
}
```

The QS checks the [FriendshipToken](glossary.md#friendship-token) and if correct returns a [KeyPackageTuple](glossary.md#keypackage-tuple) of each of the user's clients, along with a signed [KeyPackageBatch](glossary.md#user-keypackage-batch) that includes a current time stamp and the references of the returned KeyPackages.

```rust
struct GetKeyPackageBatchResponse {
  key_package_tuples: Vec<KeyPackageTuple>,
  key_package_batch: KeyPackageBatch,
}
```

##### Authentication

Instead of a QSAuthToken, the QS requires the client to provide a friendship token. If the token matches the one in the pseudonymous user record, the query is considered valid.

#### Dequeue messages

Dequeue messages from a queue, starting with the message with the given sequence number.

```rust
struct DequeueMessagesParams {
  pcid: Pcid,
  sequence_number_start: u64,
  max_message_number: u64,
}
```

The QS deletes messages older than the given sequence number and returns messages starting with the given sequence number. The maximum number of messages returned this way is the smallest of the following values.

- The number of messages remaining in the queue
- The value of the `max_message_number` field in the request
- The QS configured maximum number of returned messages

##### Authentication

* QSSenderId: Pcid

#### Negotiate websocket connection

Allows a client to create a websocket connection with the QS. If such a websocket connection exists then whenever the QS would send a push notification, it instead signals the client via the websocket connection.

```rust
struct GetWebsocketConnection {
  pcid: Pcid,
}
```

##### Authentication

* QSSenderId: Pcid

## Local homeserver endpoints

Endpoints that are accessible by other services of the local homeserver. There is no authentication on these endpoints, as we assume that the infrastructure running the services provides sufficient access control.

### Local enqueue message

Enqueue a message on the queue of a client corresponding to the given ClientQueueConfig.


```rust
struct LocalEnqueueMessageParams {
  client_queue_config: ClientQueueConfig,
  message: FanOutMessage,
}
```

The QS checks the `client_homeserver_domain` in the `client_queue_config`. If it is the homeserver's own domain, the QS decrypts the ciphertext and checks if the group ID given in the message is in the queue's blocklist. If it isn't, it enqueues the message.

If the domain is not the homerserver's own domain, the QS calls the [federated enqueue message](queuing_service.md#federated-enqueue-message) endpoint of the QS of the corresponding domain.

#### Future work: Persist and EAR encrypt federated messages
