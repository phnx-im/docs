# Work in progress: Client state

The documentation of the individual endpoints describes most of the behaviour expected by clients to interact with the homeserver. This chapter documents the state kept by each of a user's clients.

## Client state

* Client credential (plus private key)
* QS QueueConfig encryption key
* AS credential(s)
* AS intermediate credential(s)
* (Optional) push token and push token encryption key
* Own friendship token
* Own friendship encryption key
* Queue encryption key (plus private key)
* Pseudonymous client id
* Client record auth key
* QS User record auth key
* Own AddPackages (and private keys)
* Connection establishment KeyPackage (and private keys)
* For each group:
  * MLS group state
  * group state EAR key
  * credential encryption key
  * User auth key (plus private key)
  * Intermediate Client Credential (plus private key)
* For each connected user:
  * friendship token
  * friendship encryption key