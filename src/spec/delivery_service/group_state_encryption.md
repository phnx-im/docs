# Group state encryption

Generally, the DS encrypts group state at rest, such that only a group ID and a time stamp are stored in plaintext.

Parts of the group state are additionally encrypted such that they are only readable by group members.

## Encryption at rest (EAR)

With each group operation request, clients include the group's EAR key, which allows the DS to decrypt the group state, perform the group operation and re-encrypt the group state afterwards.

A group's EAR key is freshly sampled at the group's inception and is not rotated until the group is deleted.

* Clients send the key to the server for each operation
* The key is used to encrypt the whole group state
* The client creating the group samples the key freshly upon group creation
* Clients joining the group receive the key encrypted via a group info extension (i.e. either via a Welcome or via an External Init package)
* The key is fixed and is never rotated

## Credential encryption

* Clients hold a Credential Encryption Key (only known to clients)
* Used to encrypt and decrypt client credentials, as well as signature encryption keys
* The client creating the group samples the key freshly upon group creation
* Clients joining the group receive the key encrypted via a group info extension (i.e. either via a Welcome or via an External Init package)
* The key is fixed and is never rotated

## User profile key encryption

* Generated freshly by the group creator and distributed to group members alongside the credential encryption key
* Used to encrypt and decrypt the group members' individual [user profile encryption keys](../glossary.md#user-profile-encryption-key)
* The key is fixed and is never rotated
