# Functional Requirements

Before we introduce the architecture and the individual modules, we first define the functional requirements for the homeserver as a whole. We begin with the requirements dictated by the underlying Messaging Layer Security (MLS) protocol. While MLS doesn’t necessarily require a central component like a homeserver, it greatly increases the robustness of the protocol.

- Message distribution: Clients must be able to have the homeserver distribute messages to other members of a group.
- Authentication: Clients must be able to authenticate other clients. The homeserver must provide the baseline in terms of client authentication by vouching for the identity of individual clients.
- KeyPackage distribution: Clients must be able to upload their own KeyPackages to the homeserver and download those of other clients to facilitate asynchronous group additions.

Besides these basic functional requirements, a modern messaging infrastructure has to provide a few additional services.

- User and client management: Users must be able to register and add or remove clients afterwards.
- Queuing: Since message distribution must be possible asynchronously, the homeserver has to provide a mechanism for messages to be queued until their intended recipient comes online to collect them. To avoid metadata, the homeserver will likely maintain two different queue types, depending on how closely associated the queue needs to be with the user’s identity.
- Realtime notifications: When a message is queued for a given user, the homeserver should try to alert the user in real-time, such that they can retrieve their message. The ability to do so might depend on the user’s client(s).
- Group management: Message distribution will require that a certain amount of group-specific information is stored on the homeserver. Users should be able to manage that information. Ability to manage the group should be subject to a member’s role in the group (Admin/Member)
- User discovery: Users must be able to find other users and initiate a conversation with them. This is closely related to KeyPackage distribution, which enables the establishment of the shared key material between the two users’ clients.
- Asset distribution: Clients must be able to share files with one-another that are too large for individual MLS messages and the queuing system. The homeserver must enable clients to upload encrypted files for other clients to download.

Besides these basic example, a modern messenger should enable other sophisticated features such as real time voice and video chat. Since the goal of this project is an initial proof of concept, we omit these for now.

# Modularization and Architecture Overview

The requirements sketched above provide a good distribution of services across modules within the homeserver. Here, we provide a list of modules, as well as the functionality we expect them to provide to users and their clients.

## Modules

- Delivery service:
    - Initial creation of a group and management of the corresponding state (including addition and removal of members)
    - Message delivery to members of a given group
- Authentication service:
    - Registration of new users
    - Addition and removal of clients of a given user
    - User discovery
    - Authentication of users through their clients
- Queuing service:
    - Creation of queues for clients
    - Enqueuing of messages by the delivery service
    - Dequeuing of messages by the client owning the queue
- KeyPackage service:
    - Upload of KeyPackages
    - Retrieval of KeyPackages for a given client or user

## Architecture

The following shows a simplified interface between client and homeserver. Note, that for some of the security requirements (which we will detail in a later report), we might add one or more additional modules. For example, we will likely add a module that provides DDoS protection.

![Simplified interface for client ↔ backend communication.](images/homeserver_interface.png)

Simplified interface for client ↔ backend communication.

## Federated Architecture

We will later generalize the above architecture to work in a federated setting. The general principle here is that individual clients only ever communicate through their own homeserver, so if a client’s query is w.r.t. another client, user or group that exists on another backend, that query will be forwarded accordingly. The target client/user/group’s homeserver is implicit in the GroupId/UserId/QueueId in question.

![Sketch of a message delivery from one client across two homeserver. Alice contacts her Delivery Service, which in turn contacts the Queuing Service of Backend 2. Finally, Bob can retrieve the message from his homeserver by contacting the Queuing Service.](images/cross_homeserver_flow.png)

Sketch of a message delivery from one client across two homeserver. Alice contacts her Delivery Service, which in turn contacts the Queuing Service of Backend 2. Finally, Bob can retrieve the message from his homeserver by contacting the Queuing Service.
