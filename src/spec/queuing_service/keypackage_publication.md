# Friendship tokens

* The user's first client creates a fresh friendship secret and subsequently sends it to all other clients
* The user's clients can then distribute the friendship secret to all clients of users that the owning user makes connections with
* Friendship token cycling can also be used to block individual clients.
* The client can cycle the friendship token by creating a new one and distribute it to all of its (unblocked) friends.
* Future work: Replace the brute-force cycling with something cleverer such as blocklistable anonymous credentials or puncturable encryption/signatures.

# KeyPackage publication

* Clients publish pseudonymous KeyPackages along with the corresponding intermediate client credential. Both are encrypted using the friendship token (or a derived key)
* The pseudonym they use is the same as for their pseudonymous queue.
* Other clients can fetch KeyPackages by providing the friendship token to the QS
* Clients can then use the KeyPackage + intermediate client credential for use in a pseudonymous group on the IDS
* KeyPackages need to contain an extension that contains queuing information (see [here](queue_id_encryption_and_authorization.md)).

# Discovery and connection establishment

* Users publish a "contact KeyPackage" for each of their clients, where the key package is signed by the client's client credential.
* Future work: Maybe use a specific structure instead of a KeyPackage or have an extension that modifies the TBS
* A user that wants to establish contact creates a group with all of their clients in it and prepares a group-bundle for an external init
* The user sends the group bundle to the direct queues of all of the target user's clients, each bundle encrypted under the client's contact key.
* The target user decides if they want to establish contact based on the sender's user name
* Future work: The sender could send additional information encrypted under the target user's HPKE key
* If they want to establish contact, they perform an external init using a fresh (pseudonymous) keypackage to join the group with the client they accepted the connection with
* The joining client then adds all other clients via the 'normal' pseudonymous group add mechanism and distributes the Welcome via the all-clients group.