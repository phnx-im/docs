# Authentication Service Threat Model

The authentication service (AS) plays a central role in ensuring the end-to-end authentication and confidentiality of client-to-client communication in the Phoenix homeserver protocol. Its main purpose is the issuance of client credentials to new clients. These credentials are in turn used to authenticate the leaf credentials that clients use in their MLS groups.

## Baseline: Security under an honest AS

An honest AS will issue only one client credential per user name. As a consequence, users can trust that MLS messages indeed comes from the apparent sender if the signature chain on the message is valid for the given user name. In particular,

- the signature of the message must be valid under the sender's leaf credential
- the signature on the leaf credential must be valid under the sender's client credential
- the signature on the client credential must be valid under the AS intermediate credential
- the signature on the AS intermediate credential must be valid under the AS credential
- the client credential contains the user name expected in the given context

### Security under a compromised AS

In case an adversary has compromised an AS (or the AS operator is malicious), security guarantees are limited. In this scenario, the adversary can create and sign new credentials for arbitrary users. The new credential then allows the adversary to sign new leaf credentials, which the adversary can then use to impersonate the user in MLS groups.

However, there are still some security guarantees that the authentication system (in conjunction with MLS) provides in that scenario. More concretely, the adversary cannot impersonate the target user towards any other user that was in an MLS group with the target user before the AS was compromised.

This is because clients are not expected to change their client credential and use the same client credential to sign leaf credentials in all of their groups.

As a consequence, the adversary can only impersonate users towards other users that they are not already in contact with. Additionally, the adversary risks detection of the compromise if at a later point, a user that was already in a group with the target discovers the new credential. This can happen, for example, if such a user joins a group with the maliciously issued credential.
