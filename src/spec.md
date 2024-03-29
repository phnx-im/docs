# Specification (Draft)

This section contains a draft of the MLS homeserver specification. It is currently in an early draft status, but the goal is for it to fulfill the [functional requirements](./functional_requirements.md) and be the target for analysis via the [threat model](./threat_model.md).

Note that federation features are still work in progress.

The specification is split into subsections according to the modular structure of the homeserver. In particular, the specification covers the following modules.

## Work in progress: Authentication Service (AS)

This section of the specification is still work in progress. As such it is currently incomplete and subject to significant changes.

The [authentication service](spec/authentication_service.md) deals with user and client management, as well as user and client identity. For each user, it keeps an AS user record that contains the user name, as well as identifiers and cryptographic key material of the user's devices. The AS also provides the necessary funtionality for [discovery and connection establishment](spec/authentication_service/connection_establishment.md).

When communicating with the AS, users and their clients authenticate themselves using their real identity. This is as opposed to the rest of the homeserver, where each user has a pseudonym that they use for authentication.

## Work in progress: Anonymous Authentication Service (AAS)

This section of the specification is still work in progress. As such it is currently incomplete and subject to significant changes.

The anonymous authentication module allows the operator of the homeserver to rate-limit homeserver access to users and federated homeservers. The AAS acts as a middleware gating access to the endpoints of all other modules with the exception of the user registration endpoint of the AS. Registered clients can obtain anonymous (Privacy Pass) tokens from the AAS, the quantity of which can be configured by the homeserver operator. These tokens can then presented to the AAS when accessing other homeserver endpoints.

## Queuing Service (QS)

The queuing service stores queued messages for each client and forwards messages to remote queuing services. It also stores MLS KeyPackages uploaded by clients that allow users to add connected users to groups. After registration with the AS, clients can create a pseudonymous group message queue and upload KeyPackages. The homeserver's DS can deliver messages to individual queues and ask the QS to store-and-forward messages to remote QS'.

The queues and KeyPackages of a user's clients are organized on the QS in a QS user record. This record is not linked directly to the user's AS user record and prevents the homeserver from gathering metadata of individual users. For more information see the homeserver's [threat model](threat_model.md).

## Delivery Service (DS)

The delivery service is fulfills the role of the service of the same name described in the [MLS architecture document](https://www.ietf.org/id/draft-ietf-mls-architecture-08.html#section-4.3). Clients can use it to create new groups and deliver MLS messages to these groups. Group member management is transparent via the MLS-native member management.

Communication between clients and the DS is authenticated based on per-group pseudonyms. Each distinct pseudonym used by a given user on the DS is linked to the user's pseudonymous record on the QS via the [ClientQueueConfig](spec/glossary.md#sealed-queue-config) of each of the user's clients. In particular, this means that a DS alone cannot correlate group memberships of a given user. While not very meaningful in the context of an individual homeserver, this separation of pseudonyms is more relevant in the federated setting.
