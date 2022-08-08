# Functional Requirements

This section describes the functional requirements of a homeserver. Before we detail the individual requirements, we first introduce the different roles that interact with the system.

Note that while the homeserver should ultimately provide the functionality described, the entity in the given role might have to keep state and perform protocol actions to interact with the homeserver.

## Roles

A large part of the infrastructure consists of the homeserver, which is operated by the _operator_. The homeserver exposes most of its API endpoints to _clients_ of _users_ that are registered with the homeserver. The remaining endpoints of the homeserver's API are accessible to _the network_, as well as _federated homeservers_. Individuals that are registered with other homeservers are _federated users_ who run _federated clients_.

### Operator

The entity that operates a homeserver. It is assumed to have control over the domain they configure as the homeserver's _home domain_.

### The Network

The network includes all entities that have access to the port the homeserver exposes its endpoints on.

### Users

Users are individuals that have registered as users with the homeserver and that are associated with a unique and immutable _user id_ scoped by the homeserver's home domain. Each user also has a unique _user name_. Registered users can have zero or more registered clients.

The user is ultimately the entity that other users authenticate before starting a conversation.

In the context of federation, users of a given homeserver will sometimes be called _local_ users as opposed to _federated_ users.

### Clients

A client is a piece of software associated with and run by a user that. The client holds the key material that authenticate itself (and thus the user) to other clients and their users. Each client is associated with a _client id_ that is scoped by its user's user id.

### Federated Homeservers

Other instances of the homeserver that are reachable via the network, where both the given and the federated homeserver have been configured to allow mutual federation. Each is assumed to be configured with its own home domain.

### Federated Users

Individuals that have registered with a federated homeserver in the same way as users have with the given one.

### Federated Clients

Clients run by federated users. Just like a client, but associated with a federated user instead of a (local) user.

## Functional Requirements for each Role

### Functional Requirements for Homeserver Operators

* MUST be able to configure the homeserver and manage its users locally
* MUST be able to set the home domain of the homeserver during setup
* MUST be able to configure federation: Either by allowlisting other homeservers by their home domain, or by allowing open federation except for a blocklist of home domains for homeservers with which federation is not desired

### Functional Requirements for the Network

* MUST be able to register a new user

### Functional Requirements for Users

The distinction between users their clients is difficult, because the user will perform most of their interactions with the homeserver through the client. Here, we list the operations that, while performed through the client, concern the user as their own entity and might thus also affect all of their clients.

* MUST be able to manage clients
* SHOULD be able to reset the account if the last client is lost
* MUST be able to change their user name
* MUST be able to discover other users
* MUST be able to initialize a connection with other users (via a two-user MLS group, implies retrieval of other user's [KeyPackages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-11))
* MUST be able to accept or reject a connection initialized by another (local or federated) user
* SHOULD be able to block other (local or federated) users s.t. they don't receive messages from that user anymore

### Functional Requirements for Clients

MLS natively provides a number of group management mechanics such as membership management. The homeserver's task is thus mostly to deliver individual MLS messages to members of a given group, although the homeserver will likely have to track group membership to fulfill this requirement. Also, it has to provide a number of functionalities, such as the publishing of [KeyPackages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-11) and the delivery of [Welcome](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-13.4.3.1) messages to facilitate the group management functionality provided by MLS.

* MUST be able to initialize an MLS group
* MUST be able to asynchronously send [MLS messages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-7) to all members of an MLS group that it is a member of
* MUST be able to retrieve [KeyPackages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-11) for clients of users with a previously established connection
* MUST be able to send [Welcome](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-13.4.3.1) messages to clients of users with a previously established connection
* MUST be able to fetch own messages
* MUST be able to verify authenticity of MLS leaf [Credentials](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#name-credentials) of clients it shares a group with
* MUST be able to publish [KeyPackages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-11)
* SHOULD be able to configure notification settings of groups that it is a member of

### Functional Requirements for Federated Homeservers

* MUST be able to send messages for delivery to one of the homeserver's clients

### Functional Requirements for Federated Users

* MUST be able to discover users of this homeserver
* MUST be able to initialize a connection with other users of this homeserver (via a two-user MLS group)

### Functional Requirements for Federated Clients

* MUST be able to asynchronously send [MLS messages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#section-7) to all members of an MLS group that it is a member of (note that membership management in MLS happens client side, although the homeserver will have to track membership to fulfill this requirement)
* MUST be able to retrieve KeyPackages for clients of users with a previously established connection
* MUST be able to send Welcome messages to clients of users with a previously established connection
* MUST be able to verify authenticity of MLS leaf [Credentials](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#name-credentials) of clients
* SHOULD be able to configure notification settings of groups that it is a member of