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
