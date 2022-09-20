# Queuing Service (QS)

TODO: Since this service is also publishing KeyPackages now, it should get a new name.

The main purpose of the queuing service is to store-and-forward messages delivered by the [delivery service](./delivery_service.md). In this chapter we discuss the various concepts used by the QS to provide its functionalitites, as well as its individual endpoints.

TODO: If we have a single unified pseudonym for queues, key package publication and more, we could use something like a maccaroon or a biscuit for authorization purposes, i.e. who can enqueue, delete a queue, read from a queue, upload key packages, etc.

## QS state

The QS keeps a database with queues, where each queue is indexed by a UUID, its queue ID. Each queue is associated with the following pieces of data.

* **Authentication keys:** Various parties are allowed to interact with a given queue in different ways. The QS thus holds a number of keys that act as authorization keys for different operations.
  * **Owner key:** Authorizes the holder to dequeue (i.e. fetch and delete) messages in the queue, as well as to change the queue configuration such as the authentication keys.
  * **Enqueue public key:** HPKE key owned by the client and used to encrypt the queue ID for use by a DS.
  * **Deletion key:** Authorizes the holder to delete the queue.
* **Queue encryption key material:** Key material to perform [queue encryption](./queuing_service/queue_encryption.md).
  * **Owner public key:** HPKE public key of the queue owner
  * **Encryption ratchet key:** Symmetric key used to derive queue encryption keys.
* **Queue type:** A flag indicating the type of queue, i.e. either direct queue or fan-out queue. See [queue types](./queuing_service/queue_types.md) for more details.
* **Queued messages:** A sequence of ciphertexts containing the messages in the queue. Each incoming message is [encrypted](./queuing_service/queue_encryption.md) and is assigned a sequence number.
* **Blocklist:** List of domains that the QS should not accept messages from.

Additionally, the QS keeps the following state:

* **Queue ID encryption keypair:** A public/private HPKE keypair that clients can encrypt their queue ID under before providing it to a local or federated DS.
* **QS-to-QS queues:** A database of queues indexed by the remote QS' domain. Each queue has the same queue encryption key material attached as the client queues.

## Fetch queue ID encryption key

Clients can fetch the QS' queue ID encryption key through this endpoint.

## Create queue

Clients can their own queue by providing a set of authentication keys, their owner public key, a queue type, as well as a queue ID. The QS then creates the queue with the provided information.

## Delete queue

Clients can delete a given queue by sending a request to the QS that includes the queue ID and that is authenticated with the queue's deletion key. The QS then deletes the queue along with all remaining messages.

## Enqueue message

A local or federated DS can enqueue a message by providing the QS with an encrypted queue ID, as well as a message to enqueue in the queue specified by the encrypted ID. The request has to be authenticated with the queue's enqueue authorization key and the message has to fit the queue's [queue type](./queuing_service/queue_types.md).

## Dequeue message

The client owning the queue can dequeue messages by providing the QS with a queue ID and a sequence number. The message has to be authenticated using the queue's dequeue authorization key.

## Forward to federated queuing service

TODO: How exactly to do mutual authentication? Probably just have the QS publish a signature key in a well-known place and sign requests with that key.

* Can be used only by local DS
* Local DS sends as input an MLS message to forward to members of a given group, each message with attached queuing information.
* When receiving a message, the QS checks if there already exists a QS-to-QS queue for the destination domain. If so, the QS just enqueues the message there and tries to enqueue messages from that queue in the remote QS. If the queue is empty afterwards, it deletes the queue.
* Communication between two QS MUST be mutually authenticated
* If there is not already a queue, it creates a queue and repeats the previous step.