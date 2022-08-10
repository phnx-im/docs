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

Users are individuals that have registered as users with the homeserver and that are associated with a unique and immutable _user id_ scoped by the homeserver's home domain. Each user also has a unique _user name_. Registered users can have one or more registered clients.

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

Operators

1. Homeserver management: MUST be able to configure the homeserver and manage its users locally
1. Homedomain setup: MUST be able to set the home domain of the homeserver during setup
1. Federation configuration: MUST be able to configure federation: Either by allowlisting other homeservers by their home domain, or by allowing open federation except for a blocklist of home domains for homeservers with which federation is not desired

### Functional Requirements for the Network

Entities able to access the homeserver via the network

1. User registration: MUST be able to register a new user

### Functional Requirements for Users

The distinction between users and their clients is difficult because the user will perform most of their interactions with the homeserver through the client. The following is a list of operations performed through the client, which concern the user as their own entity and might thus also affect all of their clients.

Users

1. Client management: MUST be able to manage clients (this includes updates to client key material)
1. Account reset: SHOULD be able to reset the account if the last client is lost
1. User name change: MUST be able to change their user name
1. User discovery: MUST be able to discover other users
1. Connection establishment: MUST be able to initialize a connection with other users (via a two-user MLS group, implies retrieval of [KeyPackages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-11) of all of the other user's clients)
1. Connection rejection: MUST be able to accept or reject a connection initialized by another (local or federated) user
1. Connection management: SHOULD be able to block other (local or federated) users s.t. they don't receive messages from that user anymore
1. Account deletion: MUST be able to delete their account

### Functional Requirements for Clients

MLS natively provides a number of group management mechanics such as membership management. The homeserver's task is thus to fulfill the role of an MLS [Delivery Service](https://www.ietf.org/id/draft-ietf-mls-architecture-08.html#section-4).

Clients

1. Group creation: MUST be able to initialize an MLS group
1. Message delivery: MUST be able to asynchronously send [MLS messages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-7) to all members of an MLS group that it is a member of (this implies the "filtering server" role specified by the ["delivery of messages"](https://www.ietf.org/id/draft-ietf-mls-architecture-08.html#section-4.3) requirement of the MLS architecture document)
1. KeyPackage retrieval: MUST be able to retrieve [KeyPackages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-11) for clients of users with a previously established connection (this implies the ["key storage"](https://www.ietf.org/id/draft-ietf-mls-architecture-08.html#name-key-storage) requirement)
1. Welcome message delivery: MUST be able to send [Welcome](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-13.4.3.1) messages to clients of users with a previously established connection
1. Message retrieval: MUST be able to fetch their own messages
1. Client authentication: MUST be able to verify the authenticity of MLS leaf [Credentials](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#name-credentials) of clients with which it shares a group (this implies at least partially fulfilling the [Authentication Service](https://www.ietf.org/id/draft-ietf-mls-architecture-08.html#name-authentication-service) role)
1. KeyPackage publishing: jMUST be able to publish [KeyPackages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-11) (this implies the ["key retrieval"](https://www.ietf.org/id/draft-ietf-mls-architecture-08.html#name-key-retrieval) requirement)
1. Notification configuration: SHOULD be able to configure notification settings of groups of which it is a member

### Functional Requirements for Federated Homeservers

Federated homeservers

1. Federated message delivery: MUST be able to send messages for delivery to one of the homeserver's clients

### Functional Requirements for Federated Users

The functional requirements 4. and 5. for local users also apply for federated users.


### Functional Requirements for Federated Clients

The functional requirements 2., 3., 4., 6. and 8. for local clients also apply for federated clients.