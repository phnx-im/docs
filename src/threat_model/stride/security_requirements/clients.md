# Clients

This document details the STRIDE requirements for client-initiated actions.

Many client-initiated actions are operations in the context of an MLS group.
While the server can track the public group state and thus validate certain
subsets of MLS messages (such as the validity of client signatures), it cannot
validate all aspects of each MLS message. For example, the server cannot inspect
parts of the payload that are encrypted to private key material held only by
group members. As a consequence, the Integrity STRIDE property is limited to
publicly readable parts of each message.


## Get user clients

| STRIDE property | Requirement                                                                                                                                                | Remark |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Authentication  | Not a risk. Every client should be able to obtain the set of clients of another user                                                                       |        |
|                 | Access should be limited to prevent DoS attacks                                                                                                            |        |
| Integrity       | The user retrieving the client information must be able to verify it                                                                                       |        |
| Non-repudiation | Not a risk. Users must be able to discover other users anonymously                                                                                         |        |
| Confidentiality | Not a risk as long as basic [confidentiality and authentication requirements](./../security_requirements.md#basic-confidentiality-and-authentication) hold |        |
| Availability    | Users should be able to discover a reasonable number of users at a time. However, it must be hard to exhaust a user's KeyPackages                          |        |
| Authorization   | Not a risk. All users should be able to discover other users                                                                                               |        |
| Spam prevention | Not a risk. Discovering a user should not be message sending                                                                                               |        |

## Group creation

| STRIDE property | Requirement                                                                             | Remark                                    |
| --------------- | --------------------------------------------------------------------------------------- | ----------------------------------------- |
| Authentication  | Only clients of local users can create new groups                                       |                                           |
|                 | The group creator has to be able to authenticate itself as the only member of the group |                                           |
| Integrity       | The homeserver must perform checks to ensure the group state is valid                   | Only possible to a certain extent         |
| Non-repudiation | Not a risk as long as the authentication requirements hold.                             |                                           |
| Confidentiality | The homeserver must not learn the identity of the group creator                         | Authentication can be done pseudonymously |
| Availability    | Users should always be able to create new groups (limited by anti-spam measures)        |                                           |
| Authorization   | Not a risk. All users should be able to create groups                                   |                                           |
| Spam prevention | The homeserver must limit the number of groups created by a given user                  |                                           |


## Message delivery

| STRIDE property | Requirement                                                                             | Remark                                    |
| --------------- | --------------------------------------------------------------------------------------- | ----------------------------------------- |
| Authentication  | Only local or federated clients can send messages                                       |                                           |
|                 | Only members of a given group can send messages to that group                           |                                           |
|                 | Members must not be able to send messages on behalf of other members                    |                                           |
| Integrity       | The homeserver must perform checks to ensure the message is valid                       | Only possible to a certain extent         |
| Non-repudiation | Group members must be able to identify the sender of a message                          |                                           |
| Confidentiality | The homeserver must not learn the identity of the sender                                | Authentication can be done pseudonymously |
| Availability    | Users should always be able to send messages to groups that they are members of         |                                           |
| Authorization   | Not a risk. All group members should be able to send messages                           |                                           |
| Spam prevention | The homeserver must limit the number of message deliveries, as they are message sending |                                           |
