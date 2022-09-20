# Specification (Draft)

This section contains a draft of the MLS homeserver specification. It is currently a rough sketch, but the goal is for it to fulfill the [functional requirements](./functional_requirements.md) and be the target for analysis via the [threat model](./threat_model.md).

The specification is split into subsections according to the modular structure of the homeserver. In particular, the specification covers the following modules.

## Authentication Service (AS)

The authentication service deals with user account registration, as well as user account resets and allows registered users to manage their public user profile and their clients (via their evolving identity (EID)). The AS also publishes the EID and user profile of each of its users.

## Anonymous Authentication Service (AAS)

The anonymous authentication module allows the operator of the homeserver to rate-limit homeserver access to users and federated homeservers. The AAS acts as a middleware gating access to the endpoints of all other modules with the exception of the user registration endpoint of the AS. Registered clients can obtain anonymous (Privacy Pass) tokens from the AAS, the quantity of which can be configured by the homeserver operator. These tokens can then presented to the AAS when accessing other homeserver endpoints.

## Delivery Service (DS)

The delivery service is fulfills the role of the service of the same name described in the [MLS architecture document](https://www.ietf.org/id/draft-ietf-mls-architecture-08.html#section-4.3). Clients can use it to create new groups and deliver MLS messages to these groups. Group member management is transparent via the MLS-native member management.

## Queuing Service (QS)

The queuing service stores queued messages for each client. After registration with the AS, clients can create a contact queue that other users can use to initiate initial contact, as well as a pseudonymous group message queue, which is used for all other group messages. The homeserver's DS (as well as its federated counterparts) can deliver messages to individual queues. At least for group messaging queues, the sending DS has to prove that it is authorized to do so.