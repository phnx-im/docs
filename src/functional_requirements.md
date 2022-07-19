# Functional Requirements

Before we introduce the architecture and the individual modules, we first define the functional requirements for the homeserver as a whole. We begin with the requirements dictated by the underlying Messaging Layer Security (MLS) protocol. While MLS doesn’t necessarily require a central component like a homeserver, it greatly increases the robustness of the protocol. All references to MLS within this document refer to draft 16 of the MLS specification.

- Message distribution: Clients MUST be able to have the homeserver distribute [MLS messages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-7) to other members of a group.
- Authentication: Clients MUST be able to authenticate other clients. The homeserver MUST provide the baseline in terms of client authentication by providing a cryptographic link between the user's primary identifier (e.g. a username) of a given user and the key material they use to sign their MLS messages.
- KeyPackage distribution: Clients MUST be able to upload their own [KeyPackages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-11) to the homeserver and download those of other clients to facilitate asynchronous group additions.

Besides these basic functional requirements, a modern messaging infrastructure has to provide a few additional services.

- User and client management: Users MUST be able to register and add or remove clients afterwards, as well as delete their own user account, along with any of their associated data.
- Queuing: Since message distribution should be possible asynchronously, the homeserver MUST provide a mechanism for messages to be queued until their intended recipient comes online to collect them.
- Realtime notifications: When a message is queued for a given user, the homeserver MUST try to alert the user in real-time, such that they can retrieve their message. The ability to do so might depend on the user’s client(s).
- Group management: Message distribution will require that a certain amount of group-specific information is stored on the homeserver. Users MUST be able to manage that information. Ability to manage the group SHOULD be subject to a member’s role in the group (Admin/Member).
- User discovery: Users MUST be able to find other users by their full, primary identifier and obtain KeyPackages for their client(s).

Finally, homeservers should be able to federate with one-another. The scope of the federation, i.e. with which set of homeserver a given homeserver federates is up to the individual homeserver operators.

- Federated user discovery and KeyPackage distribution: Users on a given homeserver MUST be able to find users on other, federated homeservers and obtain KeyPackages for their client(s).
- Federated message distribution: Users must be able to send MLS messages to users on a federated homeserver.

Besides these basic example, a modern messenger should enable other sophisticated features such as real time voice and video chat. Since the goal of this project is an initial proof of concept, we omit these for now.
