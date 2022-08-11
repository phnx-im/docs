# Threat model

This document contains a threat model for an MLS homeserver. For now, this threat model is based solely on its functional requirements. As we build the specification of the homeserver, we will update this document to consider any additional assets.

## Functional requirements

The homeserver is expected to fulfill the functional requirements described [here](./functional_requirements.md) and to interface with parties fulfilling the roles outlined in that document.

## MLS as underlying protocol

The homeserver facilitates communication between clients via the Messaging Layer Security (MLS) protocol, which already provides a number of [security guarantees](https://www.ietf.org/id/draft-ietf-mls-architecture-08.html#name-intended-security-guarantee) which we do not describe here.

### Application assets

The homeserver will have to perform all actions listed for the individual roles in the [functional requirements section](./functional_requirements.md).

To fulfill its requirements, the homeserver will likely keep the following state (information assets). Each piece of state is annotated by the actions involved in its lifecycle.

1. Group state (for message delivery, including associated metadata such as group membership lists)
    * Create: Group creation
    * Read, Update, Delete: Message delivery (with inline MLS group management)
1. [KeyPackages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#name-key-packages) for retrieval by clients
    * Create/Update/Delete: KeyPackage publishing (publication of new KeyPackages implies deletion of old ones)
    * Read/Delete: KeyPackage retrieval (reading implies deletion, except for [KeyPackages of last resort](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#name-keypackage-reuse))
1. Authentication key material of users and their clients
    * Create/Update/Delete: Client management
    * Delete: Account reset
    * Read: Client authentication
1. Users' user names
    * Create: User registration
    * Update: User name change
    * Read: User discovery
    * Delete: Account deletion

### Security Assumptions

#### Operators

The operator has a large amount of control over the homeserver and the homeserver's users have to trust it to a certain degree. In particular, users trust homeserver operators in the following ways.

* Operators are trusted not to subject users to spam
* Operators are trusted not to deny the homeserver's service to individual users

### Analysis using STRIDE

In this section, we analyze each operation described in section [Application assets](./threat_model.md#application-assets) according to the STRIDE properties as described [here](https://www.securesoftware.nl/resources/FrameworkSecureSoftware_v1.pdf).

#### Spam reduction

Spam (outside of its potential for a denial-of-service attack on the homeserver) is a general risk for a messaging service. As a consequence, we consider spam potential a security risk for every action that leads to messages being sent to users.

#### Basic confidentiality and authentication

As a general rule, all actions performed via the network (which includes all actions with the possible exception those of the operator) MUST use be performed through a unilaterally authenticated TLS connection using the home domain of the homeserver.

TODO: Details on requirements regarding the TLS connection.

#### Metadata minimalism

Another general rule is that metadata that is not required to provide functionality at a later point in time has to be deleted immediately. Similarly, all metadata that is not required to be stored in the clear must be stored encrypted-at-rest.

#### Homeserver operator actions

##### Homeserver management

| STRIDE property | Requirement                                                             | Remark                                             |
| --------------- | ----------------------------------------------------------------------- | -------------------------------------------------- |
| Authentication  | Only administrators can perform management actions                      |                                                    |
| Integrity       | The homeserver must perform checks to ensure the configuration is valid |                                                    |
| Non-repudiation | Not a risk. There is only one homeserver operator                       | Different operators will be added later            |
| Confidentiality | Not a risk. Homeserver management actions are not confidential          |                                                    |
| Availability    | Operators should always be able to perform management actions           |                                                    |
| Authorization   | Not a risk. There is only one homeserver operator role                  | Different operator roles can be added later        |
| Spam prevention | Not a risk, since homeserver management actions are not message sending | TODO: We might have to trust operators not to spam |

##### Homeserver setup

| STRIDE property | Requirement                                                             | Remark |
| --------------- | ----------------------------------------------------------------------- | ------ |
| Authentication  | Not a risk. The entity setting up the server can only be the operator   |        |
| Integrity       | The homeserver must check if the homeserver domain is a FQDN at startup |        |
| Non-repudiation | Not a risk. At the time of setup, identities don't exist yet            |        |
| Confidentiality | Not a risk. The home domain is public                                   |        |
| Availability    | This is a setup action. The homeserver is not running at that time      |        |
| Authorization   | Not a risk. Setup is done by a single entity.                           |        |
| Spam prevention | Not a risk. Users don't exist at the time of homeserver setup           |        |

##### Federation configuration

| STRIDE property | Requirement                                                                | Remark                                      |
| --------------- | -------------------------------------------------------------------------- | ------------------------------------------- |
| Authentication  | Only administrators can configure federation                               |                                             |
| Integrity       |                                                                            |                                             |
| Non-repudiation | Not a risk. There is only one homeserver operator                          | Different operators will be added later     |
| Confidentiality | Not a risk. Federation configuration actions are not confidential          |                                             |
| Availability    | Operators should always be able to manage homeserver federation            |                                             |
| Authorization   | Not a risk. There is only one homeserver operator role                     | Different operator roles can be added later |
| Spam prevention | Not a risk, since federation configuration actions are not message sending |                                             |

#### (Public) Network actions

##### User registration

| STRIDE property | Requirement                                                                                             | Remark                 |
| --------------- | ------------------------------------------------------------------------------------------------------- | ---------------------- |
| Authentication  | Not a risk. Registration is publicly accessible                                                         |                        |
| Integrity       | The homeserver must perform checks to ensure the user name entered is valid                             | TODO: Specify validity |
| Non-repudiation | The user or its client must prove possession of authentication key material                             |                        |
| Confidentiality | The homeserver must not store metadata other than the user's user name                                  |                        |
| Availability    | User registration can be limited when the homeserver has limited resources                              |                        |
| Authorization   | Not a risk. No authorization required for registration                                                  |                        |
| Spam prevention | To mitigate spam attacks downstream of user registration, the homeserver can restrict user registration |                        |

#### User actions

##### Client management

| STRIDE property | Requirement                                                                           | Remark                                        |
| --------------- | ------------------------------------------------------------------------------------- | --------------------------------------------- |
| Authentication  | Users can only manage their own clients                                               |                                               |
| Integrity       | Not a risk. Users are responsible for the integrity of their own clients              | TODO: We might want to lock this down further |
| Non-repudiation | Other clients of the user must be able to determine which client performed the action |                                               |
| Confidentiality | Not a risk. A user's client management is considered public                           |                                               |
| Availability    | Users should always be able to manage their own clients                               |                                               |
| Authorization   | Not a risk. The user can manage clients through any of their clients                  | Client management policy can be added later   |
| Spam prevention | Client management actions should be limited, as they can be message sending           |                                               |

##### Account reset

| STRIDE property | Requirement                                                                         | Remark                                |
| --------------- | ----------------------------------------------------------------------------------- | ------------------------------------- |
| Authentication  | Users can only reset their own accounts                                             |                                       |
| Integrity       | Not a risk. A reset replaces all user data except the user name                     |                                       |
| Non-repudiation | The users contacts must be notified of the user's account reset                     |                                       |
| Confidentiality | Not a risk. No confidential user data is transmitted during an account reset        |                                       |
| Availability    | Users should be able to reset their own account once every month                    | TODO: Is this the time frame we want? |
| Authorization   | Not a risk. The user is the only entity that can perform this action                |                                       |
| Spam prevention | The homeserver must limit the number of account resets, as they are message sending |                                       |

##### User name change

| STRIDE property | Requirement                                                                            | Remark                                |
| --------------- | -------------------------------------------------------------------------------------- | ------------------------------------- |
| Authentication  | Users can only change their own user name                                              |                                       |
| Integrity       | The homeserver must ensure the new user name is valid                                  | (see user registration)               |
| Non-repudiation | The users contacts must be notified of the user's user name change                     |                                       |
| Confidentiality | The new user name must be communicated to contacts end-to-end encrypted                |                                       |
| Availability    | Users should be able to change their user name once every month                        | TODO: Is this the time frame we want? |
| Authorization   | Not a risk. The user is the only entity that can perform this action                   |                                       |
| Spam prevention | The homeserver must limit the number of user name changes, as they are message sending |                                       |

##### User discovery

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

##### Connection establishment

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

##### Connection rejection

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

##### Connection management

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

##### Account deletion

| STRIDE property | Requirement                                                                 | Remark                                                  |
| --------------- | --------------------------------------------------------------------------- | ------------------------------------------------------- |
| Authentication  | All registered users should be able to delete their won account             |                                                         |
|                 | Each user should only be able to delete their own account                   |                                                         |
| Integrity       | Not a risk, as there should be no data left after deletion                  |                                                         |
| Non-repudiation | Not a risk. Account deletion should not be logged                           | TODO: Should other users learn of a contact's deletion? |
| Confidentiality | Not a risk as long as transport encryption and metadata mimimalism hold     |                                                         |
|                 | In particular, all metadata regarding the deleted account should be deleted |                                                         |
| Availability    | Users should always be able to delete their own account                     |                                                         |
| Authorization   | Not a risk. All users should be able to delete (their own) account          |                                                         |
| Spam prevention | Account deletion is message sending and thus has to be limited              | TODO: Is this really message sending?                   |

#### Client actions

##### Group creation

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


#### Message delivery

| STRIDE property | Requirement                                                                             | Remark                                       |
| --------------- | --------------------------------------------------------------------------------------- | -------------------------------------------- |
| Authentication  | Only local or federated clients can send messages                                       |                                              |
|                 | Only members of a given group can send messages to that group                           |                                              |
|                 | Members must not be able to send messages on behalf of other members                    |                                              |
| Integrity       | The homeserver must perform checks to ensure the message is valid                       | Only possible to a certain extent            |
| Non-repudiation | Group members must be able to identify the sender of a message                          |                                              |
| Confidentiality | The homeserver must not learn the identity of the sender                                | Authentication can be done pseudonymously    |
| Availability    | Users should always be able to send messages to groups that they are members of         |                                              |
| Authorization   | Not a risk. All group members should be able to send messages                           | Group policy enforcement will be added later |
| Spam prevention | The homeserver must limit the number of message deliveries, as they are message sending |                                              |

* TODO: Go over each action and add notes on resource consumption under "availability"
* TODO: We might want to go over everything from both an HS and a client perspective.
* TODO: Use the non-repudiation field to specify level of anonymity? Or the confidentiality field?
* TODO: Come up with basic questions that are good to ask for our own use case. (Rather than the ones from the book.)


# Old content below

The subject of this threat analysis is a running homeserver application that serves its client and, potentially, other, federated homeservers. This document currently does not consider threats to the code base of the homeserver the deployment process, or any other supply chain attacks.

## Requirements and objectives

The goal of this project is to provide an implementation of a homeserver. The homeserver should facilitate the communication between clients encrypted with the Messaging Layer Security (MLS) protocol, both for clients of the homeserver itself, as well as clients of other homeservers that the homeserver federates with.

<aside>
💡 Throughout this document, we will use the terms user and client interchangeably, as users interact with the homeserver through their client. We will distinguish explicitly where necessary.

</aside>

### Privacy requirements

The design of the homeserver has a significant influence on how much metadata it accrues about its users and their interactions.

The homeserver MUST minimize the amount of metadata it requires in general and should only keep it where it is necessary to perform its functional requirements. Where metadata is kept, the homeserver MUST take measures to ensure that it is deleted from memory as soon as possible and encrypted at rest if it needs to be stored.

For functionality, where the homeserver does not explicitly need to know a client’s identity, it MUST enable anonymous access. By anonymous access, we mean that the homeserver MUST NOT be able to link two individual client requests by the content of the request alone.

Where it is required to explicitly authenticate a given client, the homeserver MUST allow the client to authenticate via an ephemeral pseudonym. By pseudonym, we mean a pseudorandomly generated identifier that the client can use to repeatedly authenticate itself towards the homeserver and that is not linked to its actual client identity.

### Security requirements

The homeserver needs to implement security measures to protect three main assets: availability of its services, maintaining the privacy requirements detailed above and the veracity of the public authentication key material of its users.

#### Authentication key material

While the underlying MLS protocol ensures that messages are end-to-end encrypted, the homeserver is still trusted to facilitate the discovery of other users and the provisioning of users with the public authentication key material of other users they interact with. While in theory, users could exchange public keys manually, or verify the veracity of server-distributed key material out-of-band, this rarely happens in practice. The authentication service part of the homeserver MUST use a verifiable log to allow users to compare their view of the authentication key material distributed by the homeserver.

#### Availability

The homeserver MUST to ensure that no actor besides the homeserver’s operator can compromise the availability of the homeserver’s services to other, legitimate actors. More concretely, the homeserver MUST take measures to ensure that no actor other than the homeserver’s operator can consume a disproportionate amount of the homeserver’s resources, regardless of the impact on other actors.

#### Metadata protection

The homeserver MUST ensure that the metadata it holds to fulfill its functional requirements are not leaked to other actors that are not explicitly authorized to learn the metadata. For example, clients that are not part of a given group must not learn its membership list.

### Objectives

Given the functional requirements, as well as the security and privacy requirements above, we can now list a number of objectives that are targets to potential attacks on the homeserver.

- The metadata of its clients
- The veracity of the public authentication key material of the homeserver
- The availability of the homeserver’s individual services to its own clients
    - Sending messages and other group operations
    - Retrieving messages from queues
    - Managing the own user’s clients
    - Obtaining public key material of other clients
    - Uploading and downloading KeyPackages
- The availability of its services to clients of federated homeservers
    - Sending messages and other group operations
    - Obtaining public key material of other clients or the homeserver itself
    - Downloading KeyPackages
- The availability of its services to federated homeservers
    - Fanning out messages to queues of the homeserver’s clients
- The availability of its services to everyone with network access
    - Registering a new client
    - Obtaining the homeserver’s own public key material

## Scope

The scope of this threat model is the homeserver implementation in general and that of its individual services in particular, its availability, the metadata it stores in memory, as well as the metadata it stores at rest.

For now, we assume that the homeserver is deployed as a monolith, with its individual services communicating within a trust boundary. Other deployment types could require more granular threat models.

To keep it simple, we also don’t consider the supply chain part of the scope and define the threat model only for a running instance of the homeserver.

## Actors

The actors that are relevant for this threat model are as follows, each relative to a given homeserver:

- The individual homeserver services
- Operators of the homeserver
    - Those with access to data at rest
    - Those with access to data in memory
- The operator of the network (for now, we don’t distinguish between subsets of the network, e.g. between the homeserver and its clients, or the homeserver and other homeservers)
- The clients of the homeserver itself
    - Specifically the members of a given group on the homeserver
        - Admins of that group
        - Regular Members of that group
- Other homeservers that federate with the given homeserver and their operators
- Clients of homeservers that federate with the given homeserver
- Non-clients, i.e. anyone else communicating with the homeserver

### Trust boundaries

Trust boundaries between the individual actors exist in several flavors, each characterized by the nature of the authentication that actors inside the respective trust boundary have to provide to one another.

- Homeserver operators and their own users
- Homeserver operators and operators of federated homeservers
- Homeserver operators and users of federated homeservers

## Threats

We now consider individual threats from the various actors in case they are malicious or compromised. For now, we consider threats that emerge from individual actors with their corresponding capabilities or set of capabilities.

This is a collection of a variety of threats that we will update as we go and as we explore as we design the homeserver specification.

### Denial of service attacks

There are several vectors that can be used to attack the availability of one or more of the homeserver’s services.

#### Attacks by adversaries with network access to the homeserver

We assume here that the homeserver requires authentication for all endpoints that are not required to bootstrap entry into one of the trust boundaries. These are, in particular, user registration, and obtaining the homeserver’s public key material (for federation).

- **DoS1:** Opening a large amount of connections to the homeserver: This potentially includes the setup of a TLS connection, as well as any potential attempt of the homeserver to authenticate the sender (explicitly, pseudonymously, or anonymously).
- **DoS2:** Registering a large number of users: The registration of a user (and the initial client) takes up computational resources (as the homeserver has to sign credentials for the new client) and storage (storing key material of the new user and client).

#### Attacks by registered clients/users

Attacks by registered clients/users can be attacks by existing, malicious clients/users, or as a follow-up of a **DoS2** attack. Note that a successful **DoS2** attack will multiply the impact of each of these attacks.

- **DoS3:** Creating a large amount of (potentially very large) groups or increasing the size of existing groups, thus consuming storage of the homeserver proportional to the number and the size of the group, as well as (potentially) bandwidth for downloads of the group’s public state by new clients. This attack can also be used to flood other users/clients with group invitations.
- **DoS4:** Sending a large amount of (potentially very large) messages to (potentially very large, see **DoS3**) groups or to a large number of groups, thus consuming storage (for message queuing), computational resources (for message verification and validation) and bandwidth. This attack can also be used to flood other users/clients with messages.
- **DoS5:** Uploading a large number of KeyPackages, thus consuming storage of the homeserver.
- **DoS6:** Uploading KeyPackages or creating groups with large extensions, thus consuming storage of the homeserver.
- **DoS7:** Downloading all KeyPackages of one or more users, thus consuming bandwidth and degrading the security of the owners by forcing other clients to use the KeyPackage of last resort (see [Section 16.4 of the MLS specification](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-16.4)).

#### Attacks by clients/users of federated homeservers

Here, the attacks **DoS4** and **DoS7** apply in the same way.

#### Attacks by (non-admin) members of a specific group

We assume here that non-admins can still manage their own clients within the group and send regular group messages.

- **DoS8:** Sending an invalid update to the group that the homeserver can recognize as invalid.
- **DoS9:** Sending an invalid update to the group that the homeserver can’t recognize as invalid.
- **DoS10:** Sending an invalid update to the group that only a part of the group can recognize as invalid.

We consider invalid group updates DoS attacks since they will make the homeserver change the state of the group, forcing other clients to either somehow role the state back (if they recognize the operation as invalid) or re-join the group if the invalid update caused their local group state to diverge from the rest of the group. Since all of these require additional interaction by clients and generally complicate the logic around state-keeping by the homeserver, the potential for invalid updates should be minimized.

The MLS protocol leaves the possibility for an updating group member to encrypt invalid path secrets to individual subtrees, which can only be detected by members in leaves of the affected subtree.

#### Attacks by an admin of a specific group

Conversely, admins can perform the same attacks as regular members. We don’t consider abuse of their elevated privileges an attack. This is equivalent to an admin wanting to sabotage a group by simply removing all members, thus effectively closing the group.

#### Attacks by the operator of the homeserver

The operator of the homeserver or an adversary that has active control over it can fully control the homeserver’s interactions with individual clients/users. Depending on their knowledge of the client’s/user’s identity relative to its pseudonyms, the attack can be more or less targeted.

- **DoS11:** Denying a specific client access to explicitly authenticated services (e.g. client additions/removals).
- **DoS12:** Removing all of a specific user’s clients.
- **DoS13:** Denying a specific user the issuance of unlinkable tokens, thus disallowing them access to all endpoints after they have spent their remaining tokens.
- **DoS14:** Denying a client with specific pseudonym access to pseudonymously authenticated services (e.g. sending messages in a group, retrieving messages).
- **DoS15:** Denying a client access to messages. This can affect either all messages, messages from a particular group, or individual messages chosen by the adversary.

#### Attacks by the operator of a federated homeserver

Operators of federated homeservers have two main capabilities: Fanning out messages to the target homeserver’s queues and being able to register arbitrarily many clients that in turn can interact with the target homeserver.

- **DoS16:** Sending a large number of messages to queues of the homeserver’s clients.
- **DoS17:** Same as **DoS4** and **DoS7**, except that they are performed by a large number of clients.

### Metadata exfiltration attacks

The following are attacks of actors with various capabilities with the goal of obtaining metadata about the clients of the homeserver.

#### Passive network access

- **ME1:** Observing traffic to identify individual clients and their contacts, either through message inspection or message flow pattern analysis.

#### Active network access

- **ME2:** Tamper with traffic to make client/user queries that reveal their identity, either through message inspection or through message flow pattern analysis.

#### Database leak or warrant for at-rest data

This threat covers cases where an adversary obtains a snapshot of all the homeserver’s at-rest state. This could be through a database leak or through a warrant served to the homeserver’s operator.

- **ME3:** Obtaining a copy of the homeserver’s database and all metadata in it. This could either reveal metadata immediately (lists of group members, queues with messages that have group ids attached), or by analyzing patterns in the data, e.g. by correlating time-stamped data points such as messages in queues or updates to group states.

#### Persistent, passive compromise of the homeserver

A persistent, passive compromise includes the adversary’s capability to read memory and observe incoming messages. For now, we don’t differentiate between full homeserver compromise and compromise of the individual services.

- **ME4:** Observing the homeserver’s memory and all metadata in it.
- **ME5:** Observing incoming messages and changes to specific groups in an effort to link individual users/clients to one or more anonymous or pseudonymous interactions.

#### Malicious homeserver operator or active, persistent compromise

An adversary with active, persistent access has the same capabilities as an adversary with passive persistent access, along with the ability to make arbitrary changes to the homeserver’s memory and at-rest state including any key material.

- **ME6:** Impersonating the homeserver to one of its clients in an effort to link an otherwise anonymous or pseudonymous interaction with the client’s actual identity.

### Meddler-in-the-middle attacks

While encrypted MLS messages are generally authenticated by the sender’s signature, the homeserver plays a role in distributing the public signature keys of clients and their KeyPackages.

#### Active network access

- **MitM1:** Impersonate the homeserver towards a client and provide that client with bogus signature keys that allow the adversary to impersonate other clients towards the target client.

#### Malicious homeserver operator or active, persistent compromise

- **MitM2:** Provide a client with bogus key material that allows the adversary to impersonate other clients towards the target client, potentially doing the same in the other direction as well for a full meddler-in-the-middle attack.
- **MitM3:** Create one or more fake clients for a given user and try to have the client’s contacts add them to the groups that the user’s other clients are in.