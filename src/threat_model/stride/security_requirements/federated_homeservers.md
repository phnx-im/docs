# Federated Homeservers

Federated homeservers allow their respective users to communicate with one
another. Each group is owned by one homeserver, which fans out messages to that
group to other homeservers (of which there are members in that group) such that
they can store and forward those messages to their users. The owning homeserver
in turn allows users of other homeservers to perform group operations for groups
that they are a member of.

The only federation-relevant action is thus the fan-out action, as all other
actions are essentially public actions.

## Message fan-out

| STRIDE property | Requirement                                                                                           | Remark |
| --------------- | ----------------------------------------------------------------------------------------------------- | ------ |
| Authentication  | Any federated homeserver can deliver messages to another homeserver                                   |        |
|                 | Homeservers must not be able to fan out messages on behalf of other homeservers                       |        |
| Integrity       | No risk                                                                                               |        |
| Non-repudiation | The recipient must be able to identify the sending homeserver                                         |        |
| Confidentiality | Not a risk                                                                                            |        |
| Availability    | Homeserver should generally be able to process fanned-out messages                                    |        |
| Authorization   | The sender must be authorized to send messages to the recipient as per recipient policy               |        |
|                 | The sender must prove that it is authorized to deliver a message to the target client                 |        |
| Spam prevention | The homeserver must limit the number of message deliveries to its client, as they are message sending |        |