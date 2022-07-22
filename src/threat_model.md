# Threat model

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