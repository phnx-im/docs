# Queuing Service (QS)

The main purpose of the queuing service is to store-and-forward messages delivered by the [delivery service](./delivery_service.md). In this chapter we discuss the various concepts used by the QS to provide its functionalitites, as well as its individual endpoints.

## QS configuration options

* Maximal InterQsAuthToken age: Maximal age of an [InterQsAuthToken](./queuing_service.md#inter-qs-authentication) presented to the QS for inter-QS authentication.
  * Default: 1h

## QS state

The QS keeps the following state.

* **Pseudonymous user records:** Indexed by a [PUID](glossary.md#pseudonymous-user-id-puid), each record contains a number of sub fields.
  * **QS user auth key:** Public signature key used by a user to authenticate itself as the owner of this record.
  * **Friendship token:** Clients have to provide the friendship token to obtain a bundle of KeyPackages for a given user.
  * **Client specific records:** For each of the user's clients, the QS keeps the following state. Indexed by a [PCID](glossary.md#pseudonymous-client-id-pcid).
    * **KeyPackages:** Encrypted KeyPackages, each with an encrypted [client credential chain](./glossary.md#client-credential-chain) attached. At least one KeyPackage has to be marked as KeyPackage of last resort.
    * **Owning client key:** Authenticates the owner of this client record and authorizes them to dequeue (i.e. fetch and delete) messages in the queue, as well as to change the queue configuration such as the authentication keys, or to add entries to the block list. Also authorizes the client to upload KeyPackages.
    * **Optional push token:** The client's (optional) push token, stored encrypted at rest under the client's push token encryption key. Used by the QS to create a push notification for the client if a message is enqueued that includes the required encryption key.
    * **Fan-out queue:** A fan-out queue with a number of further record fields attached.
      * **Queue encryption key material:** Key material to perform [queue encryption](./queuing_service/queue_encryption.md).
        * **Owner HPKE key:** HPKE public key of the queue owner
        * **Encryption ratchet key:** Symmetric key used to derive queue encryption keys.
      * **Current sequence number:** The current message sequence number.
      * **Queued messages:** A sequence of ciphertexts containing the messages in the queue. Each incoming message is [encrypted](./queuing_service/queue_encryption.md) and is assigned the current sequence number, after which the current sequence number is incremented..
      * **Blocklist salt:** Salt that is used when hashing values for the *Group blocklist*. Generated randomly by the QS upon queue creation.
      * **Blocklist:** List of salted group ID hashes that the QS should not accept messages from for this queue.
* **Queue ID encryption keypair:** A public/private HPKE keypair that clients can encrypt their queue ID under before providing it to a local or federated DS.
* **QS signing key:** A public/private signature keypair that the QS uses to sign KeyPackage bundles before returning them upon request. Also used to sign messages when forwarding them from the local DS to a remote QS.
* **QS-to-QS queues:** A database of queues indexed by the remote QS' domain. Each queue has the same queue encryption key material attached as the client queues.

TODO: Add timestamps to client records, as well as a configurable expiration time.

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

* Puid: QS User auth key
* Pcid: Owning client key

## General endpoints

Endpoints meant to access public data.

### Fetch queue ID encryption key

Clients can fetch the QS' queue ID encryption key through this endpoint.

### Fetch QS signing key

A DS or QS can fetch the QS' signing key through this endpoint.

## User record management endpoints

Endpoints for management of pseudonymous and non-pseudonymous user records. Note that pseudonymous user record is deleted with its last client record.

### Create new pseudonymous user record

Given a QS user auth key and a friendship token, create a new pseudonymous user record. The client also has to provide data to create the first client record:

* Owning client key
* Owner HPKE key

The QS then creates the user record and client record and returns both Puid and Pcid.

TODO: Do we want to have the client prove that it owns both public keys?

### Modify pseudonymous user record

Given a pseudonymous user record ID and authenticated by the QS user auth key, a client can modify the following pieces of data associated with the record:

* QS user auth key
* Friendship token

#### Authentication

* QSSenderId: Puid

### Get pseudonymous user record

Given a pseudonymous user record ID and authenticated by the QS user auth key, a client can get the following pieces of data associated with the record:

* QS user auth key
* Friendship token
* Client record data:
  * UUID
  * Owning client key
  * Owner HPKE key

#### Authentication

* QSSenderId: Puid

### Delete pseudonymous user record

Given a pseudonymous user record ID and authenticated by the QS user auth key, a client can delete the pseudonymous user record with the given ID including all client records.

#### Authentication

* QSSenderId: Puid

#### Future work: MFA for user or client record deletion

## Client record management

### Create new client record

Given a pseudonymous user record ID and authenticated by the QS user auth key, a client can create a new client record by providing the following information

* Owning client key
* Owner HPKE key

The QS returns the client ID.

#### Authentication

* QSSenderId: Puid

### Modify client record

Given a user UUID and client UUID and authenticated by the client records's owning client key, the client can modify the queue configuration. In particular, the client can change:

* Owning client key
* Owner HPKE key
* Blocklist entries

#### Authentication

* QSSenderId: Pcid

### Get client record

Given a PUID and PCID and authenticated by the queue's owning client key, the client can retrieve the queue's current configuration and in particular the following pieces of data:

* Owning client key
* Owner HPKE key
* Blocklist entries

#### Authentication

* QSSenderId: Pcid

### Delete client record

Given a PUID and PCID, the client can delete the client record with the given PCID. The last client in a user record can only be deleted by deleting the user record itself.

#### Authentication

* QSSenderId: Puid

### Publish KeyPackages

Authenticated by the owning client key, the client can upload a number of KeyPackages of which at least one has to be marked as KeyPackage of last resort. Each KeyPackage has to be accompanied by the matching encrypted, intermediate client credential. The QS deletes all existing KeyPackages before publishing the new ones.

#### Authentication

* QSSenderId: Pcid

#### Future work: Allow more granular KeyPackage rotation

#### Future work: More than one last-resort KeyPackage

### Get client KeyPackage

Given a user UUID and client UUID and authenticated by the QS user auth key, the client can retrieve a KeyPackage of the given client. The QS deletes the retrieved KeyPackage afterwards, except if it's the KeyPackage of last resort.

#### Authentication

* QSSenderId: Puid

### Get pseudonymous user KeyPackage bundle

Given a pseudonymous user ID and the user's friendship token, the QS checks the friendship token. If it matches the one on record, it prepares a bundle containing KeyPackages, as well as Intermediate Client Credentials of all of the user's clients. The QS then signs the bundle along with a time stamp and the clients' encrypted pseudonymous client IDs and returns it.

#### Authentication

Instead of a QSAuthToken, the QS requires the client to provide a friendship token. If the token matches the one in the pseudonymous user record, the query is considered valid.

## Message delivery endpoints

### Enqueue message from remote

A remote QS can enqueue a message by providing the QS with an encrypted pseudonymous client ID, as well as a message to enqueue in the queue specified by the encrypted ID. It decrypts the ciphertext and checks if the group ID given in the message is in the queue's blocklist. If it isn't, it enqueues the message.

#### Inter-QS Authentication

For each query the sending QS has to provide an InterQsAuthToken signed with its QS signing key.

```rust
struct InterQsAuthToken {
  timetamp: Timestamp,
  // TBS: timestamp
  signature: Signature,
}
```

### Enqueue message from local

A local DS can enqueue a message by providing the QS with an encrypted pseudonymous client ID, as well as a message to enqueue in the queue specified by the encrypted ID. It decrypts the ciphertext and checks if the group ID given in the message is in the queue's blocklist. If it isn't, it enqueues the message.

#### Authentication

No explicit authentication here, since we assume that local services are connected using secure channels such as mutually authenticated TLS connections.

### Dequeue message

Given a pseudonymous user record and client UUID and authenticated with the owning client key, the client can provide the QS with a sequence number and a number of messages. The QS deletes messages older than the given sequence number and returns (at most) the requested number of messages from the queue.

#### Authentication

* QSSenderId: Pcid

### Forward to federated queuing service

* Can be used only by local DS
* Local DS sends as input an MLS message to forward to members of a given group, each message with attached queuing information.
* When receiving a message, the QS checks if there already exists a QS-to-QS queue for the destination domain. If so, the QS just enqueues the message there and tries to enqueue messages from that queue in the remote QS. If the queue is empty afterwards, it deletes the queue.
* The QS establishes a TLS connection to the destination QS and signs the forwarded message(s) with its signature key.
  * Channel binding should be achieved by including a TLS exporter key in the signed message.
* If there is not already a queue, it creates a queue and repeats the previous step.

#### Authentication

No explicit authentication here, since we assume that local services are connected using secure channels such as mutually authenticated TLS connections.


### Negotiate websocket connection

Allows a client to create a websocket connection with the QS. If such a websocket connection exists then whenever the QS would send a push notification, it instead signals the client via the websocket connection.

#### Authentication

* QSSenderId: Pcid
