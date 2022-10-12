# Users

TODO: Maybe note somewhere generally how notifications, or more generally client-to-client messages must be authenticated? This seems to repeat here and in the client's section.

## Client management

| STRIDE property      | Requirement                                                                                                    | Remark |
| -------------------- | -------------------------------------------------------------------------------------------------------------- | ------ |
| Authentication (C2S) | Users can only manage their own clients                                                                        |        |
| Authentication (C2C) | The user's contacts (local and federated) must be able to authenticate the user's actions end-to-end           |        |
| Authentication (S2S) | A federated homeserver forwarding a notification must be able to authenticate the users's homeserver as origin |        |
| Integrity            | Not a risk. Users are responsible for the integrity of their own clients                                       |        |
| Non-repudiation      | Other clients of the user must be able to determine which client performed the action                          |        |
|                      | The user's contacts must be notified of the action                                                             |        |
| Confidentiality      | Not a risk. A user's client management is considered public                                                    |        |
| Availability         | Users should always be able to manage their own clients                                                        |        |
| Authorization        | Not a risk. The user can manage clients through any of their clients                                           |        |
| Spam prevention      | Client management actions should be limited, as they are message sending                                       |        |

## Account reset

| STRIDE property      | Requirement                                                                                                    | Remark                                |
| -------------------- | -------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| Authentication       | Users can only reset their own accounts                                                                        |                                       |
| Authentication (C2C) | The user's contacts (local and federated) must be able to authenticate the user's actions end-to-end           |                                       |
| Authentication (S2S) | A federated homeserver forwarding a notification must be able to authenticate the users's homeserver as origin |                                       |
| Integrity            | Not a risk. A reset replaces all user data except the user name                                                |                                       |
| Non-repudiation      | The users contacts must be notified of the user's account reset                                                |                                       |
| Confidentiality      | Not a risk. No confidential user data is transmitted during an account reset                                   |                                       |
| Availability         | Users should be able to reset their own account once every month                                               | TODO: Is this the time frame we want? |
| Authorization        | Not a risk. The user is the only entity that can perform this action                                           |                                       |
| Spam prevention      | The homeserver must limit the number of account resets, as they are message sending                            |                                       |

## User name change

| STRIDE property | Requirement                                                                            | Remark                                |
| --------------- | -------------------------------------------------------------------------------------- | ------------------------------------- |
| Authentication  | Users can only change their own user name                                              |                                       |
| Integrity       | The homeserver must ensure the new user name is valid                                  | (see user registration)               |
| Non-repudiation | The users contacts must be notified of the user's user name change                     |                                       |
| Confidentiality | The new user name must be communicated to contacts end-to-end encrypted                |                                       |
| Availability    | Users should be able to change their user name once every month                        | TODO: Is this the time frame we want? |
| Authorization   | Not a risk. The user is the only entity that can perform this action                   |                                       |
| Spam prevention | The homeserver must limit the number of user name changes, as they are message sending |                                       |

## User discovery

| STRIDE property | Requirement                                                                               | Remark |
| --------------- | ----------------------------------------------------------------------------------------- | ------ |
| Authentication  | All registered users should be able to discover other users by their full user name       |        |
|                 | Users must be able to authenticate any user information in an end-to-end way              |        |
| Integrity       | Not a risk. End-to-end authentication implies integrity of the data                       |        |
| Non-repudiation | Not a risk. Users must be able to discover other users anonymously                        |        |
| Confidentiality | The user must be able to perform discovery anonymously                                    |        |
|                 | Users should only be able to discover a limited number of users every day                 |        |
| Availability    | Users should always be able to discover users with the given confidentiality restrictions |        |
| Authorization   | Not a risk. All users should be able to discover other users                              |        |
| Spam prevention | Not a risk. Discovering a user should not be message sending                              |        |

## Connection establishment

| STRIDE property | Requirement                                                                            | Remark |
| --------------- | -------------------------------------------------------------------------------------- | ------ |
| Authentication  | All registered users should be able to establish connections to other users            |        |
|                 | The establishing user must be able to authenticate the target user                     |        |
| Integrity       | Not a risk. Authentication implies integrity of the connection request                 |        |
| Non-repudiation | Not a risk. Users must be able to establish connections anonymously                    |        |
| Confidentiality | The user must be able to establish connections anonymously w.r.t. the homeserver       |        |
| Availability    | Users should always be able to establish connections (limited by anti-spam measures)   |        |
| Authorization   | Not a risk. All users should be able to establish connections                          |        |
| Spam prevention | High spam risk. Connection establishments are message sending must be limited per user |        |

## Connection rejection

| STRIDE property | Requirement                                                                          | Remark                                |
| --------------- | ------------------------------------------------------------------------------------ | ------------------------------------- |
| Authentication  | All registered users should be able to accept/reject incoming connections requests   |                                       |
|                 | The accepting/rejecting user must be able to authenticate the requesting user        |                                       |
| Integrity       | Not a risk. Authentication implies integrity of the connection request               |                                       |
| Non-repudiation | Not a risk as long as connection requests can be authenticated by the receiving user |                                       |
| Confidentiality | Not a risk as long as transport encryption and metadata minimalism are upheld        |                                       |
| Availability    | Users should always be able to accept or reject connections                          |                                       |
| Authorization   | Not a risk. All users should be able to accept/reject connections                    |                                       |
| Spam prevention | Connection accepts/rejects are message sending and thus have to be limited           | TODO: Is this really message sending? |

## Connection management

| STRIDE property | Requirement                                                                   | Remark                                |
| --------------- | ----------------------------------------------------------------------------- | ------------------------------------- |
| Authentication  | All registered users should be able to manage their existing connections      |                                       |
|                 | Each user should only be able to manage their own connections                 |                                       |
| Integrity       | Not a risk. Clients are responsible for their own list of connections         |                                       |
| Non-repudiation | Not a risk. Connection management does not need to be logged                  |                                       |
| Confidentiality | The homeserver should not learn which connections a user has                  |                                       |
|                 | The homeserver should not learn which users a user has blocked                |                                       |
| Availability    | Users should always be able to manage their own connections                   |                                       |
| Authorization   | Not a risk. All users should be able to accept/reject (their own) connections |                                       |
| Spam prevention | Connection management is message sending and thus has to be limited           | TODO: Is this really message sending? |

## Account deletion

| STRIDE property | Requirement                                                                 | Remark                                |
| --------------- | --------------------------------------------------------------------------- | ------------------------------------- |
| Authentication  | All registered users should be able to delete their won account             |                                       |
|                 | Each user should only be able to delete their own account                   |                                       |
| Integrity       | Not a risk, as there should be no data left after deletion                  |                                       |
| Non-repudiation | Not a risk. Account deletion should not be logged                           |                                       |
| Confidentiality | Not a risk as long as transport encryption and metadata mimimalism hold     |                                       |
|                 | In particular, all metadata regarding the deleted account should be deleted |                                       |
| Availability    | Users should always be able to delete their own account                     |                                       |
| Authorization   | Not a risk. All users should be able to delete (their own) account          |                                       |
| Spam prevention | Account deletion is message sending and thus has to be limited              | TODO: Is this really message sending? |

