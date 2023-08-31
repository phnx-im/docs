# Users

TODO: Maybe note somewhere generally how notifications, or more generally client-to-client messages must be authenticated? This seems to repeat here and in the client's section.

## Authentication service endpoints
### Initiate 2FA operation

| STRIDE property | Requirement                                                                 | Remark |
| --------------- | --------------------------------------------------------------------------- | ------ |
| Authentication  | Only local clients can initiate a 2FA operation                             |        |
| Integrity       | Not a risk. The entry in the DB are ephemeral and scoped to the client's ID |        |
| Non-repudiation | Not a risk. The client needs to be authenticated, but not otherwise traced  |        |
| Confidentiality | Not a risk. There is no confidential data in the query                      |        |
| Availability    | Clients should always be able to initiate 2FA operations                    |        |
| Authorization   | Not a risk. All clients should be allowed to initiate 2FA operations        |        |
| Spam prevention | Not a risk. The operation is not message-sending or otherwise enables spam  |        |

### Initiate user registration

| STRIDE property | Requirement                                                                                                                                                | Remark |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Authentication  | Not a risk. Anyone should be able to initialize user registration                                                                                          |        |
| Integrity       | The `client_csr` included in the request must be validated (e.g. username must be unique)                                                                  |        |
| Non-repudiation | Not a risk. The client does not need to be identified beyond the `client_csr`                                                                              |        |
| Confidentiality | Not a risk as long as basic [confidentiality and authentication requirements](./../security_requirements.md#basic-confidentiality-and-authentication) hold |        |
| Availability    | It is not critical that new users be able to register and the endpoint can be disabled to mitigate an ongoing attack                                       |        |
| Authorization   | Not a risk. Everyone should be able to initiate registration                                                                                               |        |
| Spam prevention | User registration is a big potential amplifier for spam and measures need to be taken to prevent sybil attacks                                             |        |


### Finish user registration

| STRIDE property | Requirement                                                                                                                                                | Remark |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Authentication  | Only clients that have initialized user registration before should be able to call this endpoint                                                           |        |
|                 | The OPAQUE setup must be completed successfully                                                                                                            |        |
| Integrity       | The KeyPackage in the query parameters must be a valid KeyPackage with a valid credential for the calling client                                           |        |
| Non-repudiation | Not a risk. There is no need to trace this operation                                                                                                       |        |
| Confidentiality | Not a risk as long as basic [confidentiality and authentication requirements](./../security_requirements.md#basic-confidentiality-and-authentication) hold |        |
| Availability    | It is not critical that new users be able to register and the endpoint can be disabled to mitigate an ongoing attack                                       |        |
| Authorization   | Not a risk. Only clients that have previously initiated registration should be able to complete it                                                         |        |
| Spam prevention | User registration is a big potential amplifier for spam and measures need to be taken to prevent sybil attacks                                             |        |


### User account deletion

| STRIDE property | Requirement                                                                                                                                                | Remark |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Authentication  | Only the user's own clients should be able to delete the user's account                                                                                    |        |
|                 | Account deletion should require multiple factors                                                                                                           |        |
| Integrity       | Not a risk. All user data is deleted                                                                                                                       |        |
| Non-repudiation | Not a risk. Account deletion does not need to be traced                                                                                                    |        |
| Confidentiality | Not a risk as long as basic [confidentiality and authentication requirements](./../security_requirements.md#basic-confidentiality-and-authentication) hold |        |
| Availability    | Users must always be able to delete their own account                                                                                                      |        |
| Authorization   | Users must only be able to delete their own account                                                                                                        |        |
| Spam prevention | Not a risk. Account deletion should not be message sending                                                                                                 |        |


### Initiate client addition

| STRIDE property | Requirement                                                                                                                                                | Remark |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Authentication  | Users should only be able to add clients for themselves                                                                                                    |        |
| Integrity       | The client credential must be well-formed and the client ID must contain the correct user name                                                             |        |
|                 | The client ID must be unique among all of the user's clients                                                                                               |        |
| Non-repudiation | Not a risk until the operation is finalized                                                                                                                |        |
| Confidentiality | Not a risk as long as basic [confidentiality and authentication requirements](./../security_requirements.md#basic-confidentiality-and-authentication) hold |        |
| Availability    | Users must always be able to add new clients                                                                                                               |        |
| Authorization   | Users must only be able to add clients under their own user name                                                                                           |        |
| Spam prevention | Not a risk. Initiating addition is not message sending and no clients are added yet                                                                        |        |

### Finish client addition

| STRIDE property | Requirement                                                                                                                                                | Remark |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Authentication  | Users should only be able to add clients for themselves                                                                                                    |        |
| Integrity       | There must be a record in the database indicating that the same user has previously initiated the client addition with matching data                       |        |
| Non-repudiation | Other clients of the user should be notified that a client was added                                                                                       |        |
| Confidentiality | Not a risk as long as basic [confidentiality and authentication requirements](./../security_requirements.md#basic-confidentiality-and-authentication) hold |        |
| Availability    | Users must always be able to add new clients                                                                                                               |        |
| Authorization   | Users must only be able to add clients under their own user name                                                                                           |        |
| Spam prevention | The query must be authenticated and notifications should only be sent to the user's other clients                                                          |        |
|                 | The number of clients for a user should be limited as clients can act as amplifiers for spam or DoS attacks                                                |        |

### Delete client

| STRIDE property | Requirement                                                                                                                                                | Remark |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Authentication  | Users should only be able to delete their own clients                                                                                                      |        |
| Integrity       | A client that is deleted must no longer be able to act as a valid client for the user                                                                      |        |
| Non-repudiation | Other clients of the user should be notified that a client was deleted                                                                                     |        |
| Confidentiality | Not a risk as long as basic [confidentiality and authentication requirements](./../security_requirements.md#basic-confidentiality-and-authentication) hold |        |
| Availability    | Users must always be able to delete their clients                                                                                                          |        |
| Authorization   | Users must only be able to their own clients                                                                                                               |        |
| Spam prevention | The query must be authenticated and notifications should only be sent to the user's other clients                                                          |        |

### Dequeue messages

| STRIDE property | Requirement                                                                                                                                                | Remark |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Authentication  | Clients should only be able to dequeue messages from their own queue                                                                                       |        |
| Integrity       | Outdated messages (sequence numbers lower than `sequence_number_start`) must be deleted from the queue                                                     |        |
| Non-repudiation | Not a risk. Message dequeuing need not be traced                                                                                                           |        |
| Confidentiality | Not a risk as long as basic [confidentiality and authentication requirements](./../security_requirements.md#basic-confidentiality-and-authentication) hold |        |
| Availability    | Clients must always be able to dequeue messages from their own queue                                                                                       |        |
| Authorization   | Clients must only be able to dequeue messages from their own queue                                                                                         |        |
| Spam prevention | Dequeuing messages is not a message sending operation                                                                                                      |        |

### Enqueue message

| STRIDE property | Requirement                                                                                                                               | Remark |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Authentication  | Not a risk. All clients should be able to enqueue messages into the queues of all other clients                                           |        |
| Integrity       | Not a risk.                                                                                                                               |        |
| Non-repudiation | Not a risk. Clients must be able to enqueue messages anonymously                                                                          |        |
| Confidentiality | Basic [confidentiality and authentication requirements](./../security_requirements.md#basic-confidentiality-and-authentication) must hold |        |
|                 | Enqueued messages must be encrypted according to the specification                                                                        |        |
| Availability    | The ability to enqueue messages for a given client might be deactivated or rate-limited if there is increased spam activity               |        |
| Authorization   | Not a risk. All clients may enqueue messages for all other clients                                                                        |        |
| Spam prevention | Enqueuing messages entails a high spam risk. Message enqueuing should be heavily rate-limited                                             |        |


