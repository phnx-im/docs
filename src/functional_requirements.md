# Functional Requirements

This section describes the functional requirements of a homeserver. Before we detail the individual requirements, we first introduce the different roles that interact with the system.

Note that while the homeserver should ultimately provide the functionality described, the entity in the given role might have to keep state and perform protocol actions to interact with the homeserver.

## Roles

A large part of the infrastructure consists of the homeserver, which is operated by the _operator_. The homeserver exposes most of its API endpoints to _clients_ of _users_ that are registered with the homeserver. The remaining endpoints of the homeserver's API are accessible to _the network_, as well as _federated homeservers_. Individuals that are registered with other homeservers are _federated users_ who run _federated clients_.

### Operator

The entity that operates a homeserver. It is assumed to have control over the domain they configure as the homeserver's _home domain_.

### The Network

The network includes all entities that have access to the port exposing the homeserver endpoints.

### Users

Users are individuals that have registered as users with the homeserver and that are associated with a unique and immutable _user id_ scoped by the homeserver's home domain. Each user also has a unique UUID called _user id_. Registered users can have one or more registered clients.

The user is ultimately the entity that other users authenticate before starting a conversation.

In the context of federation, users of a given homeserver will sometimes be called _local_ users as opposed to _federated_ users.

### Clients

A client is a piece of software associated with and run by a user. The client holds the key material used to authenticate (and thus the user) to other clients and their users. Each client is associated with a _client id_ that is scoped by its user's ID.

### Federated Homeservers

Other instances of the homeserver that are reachable via the network, where both the local and the federated homeserver have been configured to allow mutual federation. Each is assumed to be configured with its own home domain.

### Federated Users

Individuals that have registered with a federated homeserver in the same way as users have with the local one.

### Federated Clients

Federated clients are clients which are run by federated users. They are regular clients that are associated with a federated user instead of a local user.

## Functional Requirements for each Role

### Functional Requirements for Homeserver Operators

* Homeserver management: Operators MUST be able to configure the homeserver and manage its users locally.
* Homedomain setup: Operators MUST be able to set the home domain of the homeserver during setup.
* Federation configuration: Operators MUST be able to configure federation: Either by allowlisting other homeservers by their home domain, or by allowing open federation except for a blocklist of home domains for homeservers with which federation is not desired.

### Functional Requirements for the Network

* User registration: Entities in the public network MUST be able to register a new user.

### Functional Requirements for Users

The distinction between users and their clients is difficult because the user will perform most of their interactions with the homeserver through the client. The following is a list of operations performed through the client, which concern the user as their own entity and might thus also affect all of their clients.

* Display name change: Users MUST be able to change their display name. Members of groups the user is in MUST be notified of the new name.
* User discovery: Users MUST be able to discover other users.
* Connection establishment: Users MUST be able to initialize a connection with previously discovered users (via a two-user MLS group, implies retrieval of [KeyPackages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-11) of all of the other user's clients).
* Connection rejection: Users MUST be able to accept or reject a connection initialized by another (local or federated) user. The sender of the connection request SHOULD be notified of the acceptance or rejection of the request.
* Account deletion: Users MUST be able to delete their account. Members of groups the user is in MUST be notified of the deletion.

### Functional Requirements for Clients

MLS natively provides a number of group management mechanics such as membership management. The homeserver's task is thus to fulfill the role of an MLS [Delivery Service](https://www.ietf.org/id/draft-ietf-mls-architecture-08.html#section-4).

* Group creation: Clients MUST be able to initialize an MLS group.
* Group deletion: Clients MUST be able to delete an MLS group.
* Message delivery: Clients MUST be able to asynchronously send [MLS messages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-7) to all members of an MLS group that it is a member of (this implies the "filtering server" role specified by the ["delivery of messages"](https://www.ietf.org/id/draft-ietf-mls-architecture-08.html#section-4.3) requirement of the MLS architecture document). If a given group member has provided the homeserver with a notification for this group, the homeserver MUST attach the notification policy to the message when delivering it. If a group member is a federated client, the homeserver MUST forward the message to the federated homeserver for delivery.
* Message queuing: The homeserver MUST store messages sent to a client either by itself or forwarded by a federated homeserver while the client is offline.
* Message notifications: Clients MAY provide the homeserver with a means to notify them when a new message is queued, as well as a default notification policy. If the client has provided the homeserver with such a means and corresponding policy, the homeserver MUST notify the client according to either the policy attached to the message, or, if none was attached, according to the default policy.
* KeyPackage retrieval: Clients MUST be able to retrieve [KeyPackages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-11) for clients of users with a previously established connection (this implies the ["key storage"](https://www.ietf.org/id/draft-ietf-mls-architecture-08.html#name-key-storage) requirement). As long as there is more than one KeyPackage, the server MUST delete the KeyPackage after it was provided to a (local or federated) client.
* Welcome message delivery: Clients MUST be able to send [Welcome](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-13.4.3.1) messages to clients of users with a previously established connection. If the client has provided the homeserver with a means of notification (and default notification policy), the homeserver MUST notify the client according to the default policy.
* Message retrieval: Clients MUST be able to fetch messages queued by the homeserver.
* Client authentication: Clients MUST be able to verify the authenticity of MLS leaf [Credentials](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#name-credentials) of clients with which it shares a group (this implies at least partially fulfilling the [Authentication Service](https://www.ietf.org/id/draft-ietf-mls-architecture-08.html#name-authentication-service) role).
* KeyPackage publishing: Clients MUST be able to publish [KeyPackages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-11) (this implies the ["key retrieval"](https://www.ietf.org/id/draft-ietf-mls-architecture-08.html#name-key-retrieval) requirement). When a client publishes new KeyPackages, the homeserver MUST delete all remaining previously uploaded KeyPackages of that client.

### Functional Requirements for Federated Homeservers

* Federated message delivery: Federated homeservers MUST be able to send messages for delivery to one of the homeserver's clients.

### Functional Requirements for Federated Users

The functional requirements _user discovery_ and _connection establishment_ for local users also apply for federated users.


### Functional Requirements for Federated Clients

The functional requirements _message delivery_, _KeyPackage retrieval_, _welcome message delivery_, _client authentication_ and _notification configuration_ for local clients also apply for federated clients.
