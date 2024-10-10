# Queue encryption

Messages in queues contain metadata such as epochs, group IDs and explicit changes to group states (although all based on pseudonymous identities). To protect this metadata against state-leakages, messages in queues are stored encrypted in such a way that only the client owning the queue can decrypt the messages.

Since state-leakage (i.e. state compromise) is part of the threat model, the encryption protocol provides forward secrecy and post-compromise security.

## Key ratcheting

Upon creation of a [QS client record](../queuing_service.html#create-new-qs-user-record), the creator provides a [queue encryption key](../glossary.md#queue-encryption-key) which the QS stores alongside the queue. The QS then samples a random string as the initial ratchet key, stores a copy and encrypts another copy under the queue encryption key. The resulting ciphertexts are enqueued like regular messages.

Whenever the QS enqueues a message, it first uses the ratchet key to derive two keys using distinct labels: The next ratchet key and a message encryption key. The message encryption key is used to encrypt the message before tagging it with the current sequence number (and advancing the currently stored sequence number) and enqueuing it. The next ratchet key is stored in place of the old ratchet key (which is deleted after use, along with the message encryption key). This process is repeated with every message to be enqueued.

The ratcheting of encryption keys (and the deletion of keys after use) provides forward secrecy.

The client fetching the queue for the first time first decrypts the ratchet key. For each message it fetches from the queue, the client follows the same derivation steps as the QS and decrypts the fetched messages.

## Key updates

To achieve post-compromise security, the ratchet keys must be periodically injected with fresh randomness. The frequency with which this happens can be configured via the [QS configuration options](../queuing_service.md#qs-configuration-options). The injected randomness must be encrypted and sent to the owner of the QS client record, so the frequency with which the keys are rotated represents a balance between security and performance.

Whenever the ratchet key should be refreshed, the QS derives a rotation key from the current ratchet key. The QS then derives a new ratchet key from the rotation key and some fresh randomness. The QS then encrypts the injected randomness under the queue encryption key. The encrypted randomness is then tagged with the current sequence number and enqueued like a regular message. The new ratchet key then replaces the old ratchet key.

When dequeuing such an encrypted key update, the client decrypts the update and follows the same derivation steps as the QS to obtain the fresh ratchet key.