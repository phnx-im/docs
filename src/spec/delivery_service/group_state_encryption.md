# Group state encryption

Generally, the DS encrypts group state at rest, such that only a group ID and a time stamp are stored in plaintext.

Parts of the group state are additionally encrypted such that they are only readable by group members.

## Encryption at rest (EAR)

* Clients send the key to the server for each operation
* The key is used to encrypt the whole group state
* The client creating the group samples the key freshly upon group creation
* Clients joining the group receive the key encrypted via a group info extension (i.e. either via a Welcome or via an External Init package)
* The key is fixed and is never rotated
* Future work: The key should be rotated (see below)

## Credential encryption

* Clients hold a Credentials Encryption Base Key (only known to clients)
* Used to encrypt and decrypt credential chains
* The client creating the group samples the key freshly upon group creation
* Clients joining the group receive the key encrypted via a group info extension (i.e. either via a Welcome or via an External Init package)
* The key is fixed and is never rotated
* Future work: The key should be rotated (see below)

## Future work: Key rotating encryption

This scheme allows the simultaneous use of two secrets for decryption, such that two keys are always valid at a time. The use of two valid keys at a time allows the rotation of keys in a fixed interval.

* The keys used in this scheme are not directly used for encryption. Instead, they act as a base secret from which sender and receiver derive a key ID and the encryption/decryption key. When encrypting, the cipertext is tagged with the key ID.
* A key also has a lifetime known to sender and receiver.
* The sender derives from the base secret and encrypts the payload.
* Before decryption, the recipient derives from the base secret and checks if the key ID matches the one the ciphertext is tagged with.
* Once the key has reached its lifetime, the sender samples a new base key and encrypts a copy of it under the old base key.
* From now on, the sender always uses the new base key for encryption and attaches the encrypted key to ciphertexts (in addition to the key ID)
* The recipient can now check based on the ID of the key it holds, as well as that on a given ciphertext (and that on a potential attached encrypted key) if it first needs to decrypt the wrapped key, or if it can simply decrypt the main ciphertext.

