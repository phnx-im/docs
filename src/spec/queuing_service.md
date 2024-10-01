# Queuing Service (QS)

The main purpose of the queuing service is to store-and-forward messages delivered by the [delivery service](./delivery_service.md), as well as the publication of KeyPackages. In this chapter we discuss the various concepts used by the QS to provide its functionalities, as well as its individual endpoints.

## Overview

The store-and-forwards functionality of the QS is designed in such a way that the QS cannot link the state it maintains for each of the homeserver's users with the user's actual identity as maintained by the AS.

To avoid a connection with the user's actual identity, each user has a QS user record with a sub-record the user's client. The records are created by the user's client after it has registered with the AS.

After creation, the user's client can publish KeyPackages and fetch messages from its queue, as well as rotate the key material used for authentication, or for at-rest encryption of queued messages.

The QS user record also contains the user's [friendship token](glossary.md#friendship-token), which the QS uses to authenticate requests the user's KeyPackages.

The client-specific records contain the client's fan-out queue (for messages received from a local or a federated DS), as well as the KeyPackages published by the client. See [here](queuing_service/keypackage_publication.md) for more information on KeyPackage publication and how unlinkability between a user's real identity and its pseudonym is maintained.

The fan-out queue in turn contains an optional push-token (encrypted at-rest under the owning client's [push-token encryption key](glossary.md#push)), as well as key material used for the [queue's at-rest encryption](queuing_service/queue_encryption.md).

## QS state

The QS keeps the following state.

* **QS user records:** Indexed by a [QsUid](glossary.md#qs-user-id-qsuid), each record contains a number of sub fields.
  * **QS user record auth key:** [Public signature key](glossary.md#qs-user-record-auth-key) used by a user to authenticate itself as the owner of this record.
  * **Friendship token:** Clients have to provide the friendship token to obtain a bundle of KeyPackages for a given user.
  * **Client specific records:** For the users' clients, the QS keeps the following state. Indexed by a [QsCid](glossary.md#qs-client-id-qscid).
    * **Activity time:** Timestamp indicating the last time a client has fetched messages from the queue.
    * **KeyPackages:** Encrypted KeyPackages, each with an encrypted [client credential chain](./glossary.md#client-credential-chain) attached. At least one KeyPackage has to be marked as KeyPackage of last resort.
    * **Public QS client record key:** [Public signature key](glossary.md#qs-client-record-auth-key). Authenticates the owner of this QS client record and authorizes them to dequeue (i.e. fetch and delete) messages in the queue, as well as to change the queue configuration such as the authentication keys, or to add entries to the block list. Also authorizes the client to upload KeyPackages.
    * **Optional push token:** The client's (optional) push token, stored encrypted at rest under the client's push token encryption key. Used by the QS to create a push notification for the client if a message is enqueued that includes the required encryption key.
    * **Fan-out queue:** A fan-out queue with a number of further record fields attached.
      * **Queue encryption key material:** Key material to perform [queue encryption](./queuing_service/queue_encryption.md).
        * **Queue encryption key:** [HPKE public key](glossary.md#queue-encryption-key) of the queue owner.
        * **Encryption ratchet key:** Symmetric key used to derive queue encryption keys.
      * **Current sequence number:** The current message sequence number.
      * **Queued messages:** A sequence of ciphertexts containing the messages in the queue. Each incoming message is [encrypted](./queuing_service/queue_encryption.md) and is assigned the current sequence number, after which the current sequence number is incremented..
      * **Blocklist salt:** Salt that is used when hashing values for the *Group blocklist*. Generated randomly by the QS upon queue creation.
      * **Blocklist:** List of salted group ID hashes that the QS should not accept messages from for this queue.
* **Queue ID authentication key:** A symmetric key that the QS uses to MAC a queue ID. Clients have to include that MAC when they encrypt their queue ID under the Queue ID encryption key. The QS uses this key to validate queue IDs after encryption and before enqueuing a message.
* **Queue ID encryption keypair:** [A public/private HPKE keypair](glossary.md#queueconfig-encryption-key) that clients can encrypt their queue ID and queue id MAC under before providing it to a local or federated DS.
* **QS signing key:** [A public/private signature keypair](./glossary.md#qs-signing-key) that the QS uses to sign KeyPackage bundles before returning them upon request. Also used to sign messages when forwarding them from the local DS to a remote QS.
* **QS-to-QS queues:** A database of queues indexed by the remote QS' domain. Each queue has the same queue encryption key material attached as the client queues.

## QS configuration options

* Maximal InterQsAuthToken age: Maximal age of an [InterQsAuthToken](./queuing_service.md#inter-qs-authentication) presented to the QS for inter-QS authentication.
  * Default: 1h
* Maximal QS client record age: Maximal age of an inactive QS client record.
  * Default: 90d
* Maximal number of requested messages: Maximal number of messages that will be returned to a client requesting messages from a queue.
  * Default: 500
* Ratchet key rotation interval: Interval in which queue encryption ratchet keys are rotated. See [here](queuing_service/queue_encryption.md) for more details on queue encryption.
* Maximal number of QS client records per user
  * Default: 10


## Authentication

Messages from the client to the QS are authenticated by the client by providing an QSAuthToken, where the QsSenderId in the token depends on the endpoint the client is querying.

```rust
enum QsSenderId {
  QsUid(QsUid),
  QsCid(QsCid),
}

struct QsAuthToken {
  sender_id: QsSenderId,
  timestamp: Timestamp,
  // TBS: sender_id and timestamp
  signature: Signature,
}
```

The verification key used to create the token depends on the sender_id:

* QsUid: [QS user record auth key](./glossary.md#qs-user-record-auth-key)
* QsCid: [QS client record auth key](./glossary.md#qs-client-record-auth-key)

All client endpoints additionally require the client to submit a privacy pass token valid under the privacy pass public key of the AS of the same domain as the QS. The only two exceptions are the endpoints to dequeue messages and to negotiate a websocket connection.

## DS-to-QS communication

### Enqueuing messages

A local DS can enqueue the message sending just a [`FanOutMessage`](./glossary.md#fan-out-message).

The receiving QS first checks the FQDNs in the `client_queue_configs` and
[forwards the messages for remote
QS'](./queuing_service.md#federated-qs-to-qs-communication).

For all local `client_queue_configs`, the QS decrypts the ciphertext, verifies the MACs using its symmetric
authentication key and checks if the QS client record exists. If it doesn't, it
responds to the sending QS with the following message.

```rust
struct QueueDeleted {
  client_queue_config: ClientQueueConfig,
  group_id: GroupId,
}
```

If the QS client record exists, the QS checks if the group ID in the
message is in the blocklist of the associated queue. If it isn't, the QS
enqueues the message.

## Publicly avaliable endpoints

### Fetch QS verifying key

A DS or QS can fetch the [QS' verifying key](./glossary.md#qs-signing-key)
through this endpoint. This endpoint has to be reachable via a TLS connection to
allow the sender to authenticate the verifying key using its local web root of
trust.

### Federated QS-to-QS communication

This endpoint allows a remote QS to enqueue a [fan-out
message](glossary.md#fan-out-message). This endpoint has to be reachable via a
TLS connection only to allow the remote QS to authenticate this QS based on its
local web root of trust.

```rust
struct QsToQsMessage {
  source: Fqdn,
  destination: Fqdn,
  payload: FanOutMessage,
  signature: Signature,
}
```

When receiving a `QsToQsMessage`, the QS verifies that it is the intended
destination and tries to fetch the verifying key of the sending QS (if it is not
cached locally) based on the `source` field. The recipient QS then uses the
verifying key to verify the signature. If the signature is valid, the QS
processes the message like a message from its local QS.

#### Rate-limiting

The QS can rate-limit messages from a federated QS in multiple ways. The most
obvious way is to rate-limit messages simply based on the sender's domain.
Secondly, the receiving QS can rate-limit based on the client queue config in
the FanOutMessage, which is unique to the group in the context of which the
message was fanned out. Finally, the QS can rate-limit the message based on the
decrypted queue config, i.e. on a per-queue/client basis.

## Client endpoints

Endpoints accessible to clients of the homeserver via HTTP requests.

### Fetch queue config encryption key

* Endpoint: `ENDPOINT_QS_QC_ENCRYPTION_KEY`

Clients can fetch the QS' [queue config encryption key](./glossary.md#queueconfig-encryption-key) through this endpoint.

### User record management endpoints

Endpoints for management of QS user records. Note that a QS user record is deleted with its last QS client record.

#### Create new QS user record

Create a new QS user record, as well as a first QS client record.

```rust
struct CreateUserRecordParams {
  user_record_auth_key: SignaturePublicKey,
  friendship_token: FriendshipToken,
  client_record_auth_key: SignaturePublicKey,
  queue_encryption_key: HpkePublicKey,
}
```

The QS creates the QS user record and QS client record, indexed by a freshly sampled QsUid and QsCid. The QS returns the following:

```rust
struct CreateUserRecordResponse {
  qs_uid: QsUid,
  qs_cid: QsCid,
}
```

#### Update QS user record

Update a given QS user record, overwriting the existing values with the one given in the message.

```rust
struct UpdateUserRecordParams {
  qs_uid: QsUid,
  user_record_auth_key: SignaturePublicKey,
  friendship_token: FriendshipToken,
}
```

##### Authentication

* QsSenderId: QsUid

#### Get own QS user record

Get the data associated with a given QS user record that you own.

```rust
struct UserRecordParams {
  qs_uid: QsUid,
}
```

The QS returns the following.

```rust
struct UserRecordResponse {
  user_record_auth_key: SignaturePublicKey,
  friendship_token: FriendshipToken,
  client_records: Vec<ClientRecordResponse>,
}
```

##### Authentication

* QsSenderId: QsUid

#### Delete QS user record

Delete the given QS user record including all associated QS client records.

```rust
struct DeleteUserRecordParams {
  qs_uid: QsUid,
}
```

##### Authentication

* QsSenderId: QsUid

#### Publish KeyPackages

Publish the given [AddPackage](glossary.md#addpackage) under the given QsCid.

All of the KeyPackages contained in the AddPackages have to contain a [QueueConfigExtension](glossary.md#queueconfig-extension) and at least one of the AddPackage has to contain a KeyPackage marked as [KeyPackage of last resort](glossary.md#last-resort-extension).

```rust
struct PublishKeyPackagesParams {
  qs_cid: QsCid,
  add_packages: Vec<AddPackage>,
}
```

The QS deletes all existing KeyPackages before publishing the new ones.

##### Authentication

* QsSenderId: QsCid

#### Get AddPackage

Get a KeyPackage of the user with the given friendship token.

```rust
struct AddPackageParams {
  friendship_token: FriendshipToken,
}
```

The QS checks if there is a QS user record with the given [FriendshipToken](glossary.md#friendship-token) and returns an [AddPackage](glossary.md#addpackage) of the matching user's client.

```rust
struct AddPackageResponse {
  add_packages: Vec<AddPackage>,
}
```

##### Authentication

Instead of a QSAuthToken, the QS requires the client to provide a friendship token. If the token matches the one in the QS user record, the query is considered valid.

#### Dequeue messages

Dequeue messages from a queue, starting with the message with the given sequence number.

```rust
struct DequeueMessagesParams {
  qs_cid: QsCid,
  sequence_number_start: u64,
  max_message_number: u64,
}
```

The QS deletes messages older than the given sequence number and returns messages starting with the given sequence number. The maximum number of messages returned this way is the smallest of the following values.

* The number of messages remaining in the queue
* The value of the `max_message_number` field in the request
* The QS configured maximum number of returned messages

##### Authentication

* QsSenderId: QsCid

#### Negotiate websocket connection

Allows a client to create a websocket connection with the QS. If such a websocket connection exists then whenever the QS would send a push notification, it instead signals the client via the websocket connection.

```rust
struct WsParams {
  qs_cid: QsCid,
}
```

##### Authentication

* QsSenderId: QsCid

## Local homeserver endpoints

Endpoints that are accessible by other services of the local homeserver. There is no authentication on these endpoints, as we assume that the infrastructure running the services provides sufficient access control.

### Local enqueue message

Receive a [fan-out-message](glossary.md#fan-out-message) to store-and-forward to local clients with the specified ClientQueueConfigs.

The QS checks the `client_homeserver_domain` in the `client_queue_config`. If it is the homeserver's own domain, the QS decrypts the ciphertext and checks if the group ID given in the message is in the queue's blocklist. If it isn't, it enqueues the message.

If the domain is not the homerserver's own domain, the QS calls the [federated enqueue message](queuing_service.md#federated-enqueue-message) endpoint of the QS of the corresponding domain.

If the QS learns that a message couldn't be delivered due to a missing queue, either because a local lookup has failed, or due to a response from a federated QS, it reports the `client_queue_config` and the group ID back to the DS via a [QueueDeleted](queuing_service.md#federated-enqueue-message) message.
