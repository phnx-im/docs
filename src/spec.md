# Specification (Draft)

This section contains a proof-of-concept specification of the Phoenix homeserver protocol. The goal is for it to fulfill the [functional requirements](./functional_requirements.md) and be the target for analysis via the [threat model](./threat_model.md).

The specification is split into subsections according to the modular structure of the homeserver. In particular, the specification covers the following modules.

## Authentication Service (AS)

The [authentication service](spec/authentication_service.md) deals with user management, as well as user and client identity. For each user, it keeps an AS user record that contains the user name, as well as identifiers and cryptographic key material of the user's client. The AS also provides the necessary funtionality for [discovery and connection establishment](spec/authentication_service/connection_establishment.md).

When communicating with the AS, users authenticate themselves using their real identity. This is as opposed to the rest of the homeserver, where users use pseudonyms for authentication.

## Queuing Service (QS)

The queuing service stores queued messages for each client and forwards messages to remote queuing services. It also stores MLS KeyPackages uploaded by clients that allow users to add connected users to groups. After registration with the AS, clients can create a pseudonymous group message queue and upload KeyPackages. The homeserver's DS can deliver messages to individual queues and ask the QS to store-and-forward messages to remote QS'.

The queues and KeyPackages of a user's clients are organized on the QS in a QS user record. This record is not linked directly to the user's AS user record and prevents the homeserver from gathering metadata of individual users. For more information see the homeserver's [threat model](threat_model.md).

## Delivery Service (DS)

The delivery service is fulfills the role of the service of the same name described in the [MLS architecture document](https://www.ietf.org/id/draft-ietf-mls-architecture-08.html#section-4.3). Clients can use it to create new groups and deliver MLS messages to these groups. Group member management is transparent via the MLS-native member management.

Communication between clients and the DS is authenticated based on per-group pseudonyms. Each distinct pseudonym used by a given user on the DS is linked to the user's pseudonymous record on the QS via the [ClientQueueConfig](spec/glossary.md#sealed-queue-config) of each of the user's clients. In particular, this means that a DS alone cannot correlate group memberships of a given user. While not very meaningful in the context of an individual homeserver, this separation of pseudonyms is more relevant in the federated setting.
