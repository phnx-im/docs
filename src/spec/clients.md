# Clients

The only purpose of the AS, QS and DS is to serve users through their clients.
Using the homeserver services, users can establish connections with one-another,
create groups and invite people to them, join other groups and send messages.

This chapter details the main operations that the client supports and how the
client interacts with its homeserver, or other, federated instances.

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