# Group interactions

Clients can interact with the homeserver to create and manage MLS groups using the following operations.

## Group creation

To create a new group, the client requests a [group ID from the DS](../delivery_service.md#request-group-id).

It then freshly generates the following key material.

- credential encryption key (see [credential encryption](../delivery_service/group_state_encryption.md#credential-encryption))
- group state encryption key (see [group state encryption](../delivery_service/group_state_encryption.md#group-state-encryption))
- a [leaf credential](../authentication_service/credentials.md#leaf-credentials)

Finally, user creates the initial MLS group locally and calls the [group creation endpoint](../delivery_service.md#create-group) of the DS.

## Inviting other users

A client can invite other users (or rather their clients) that they are [connected with](../authentication_service/connection_establishment.md) to join a group by first fetching an AddPackage for each client that is meant to be added to the target group.

The inviting client prepares the query to the DS by performing the following steps:

- encrypt the credentials of the invitee clients to the group's credential encryption key
- perform a commit in the relevant MLS group, adding the new clients using the clients' KeyPackages
- include the encrypted credentials in the AAD of the resulting MLSMessage
- encrypt the [welcome attribution infos](../glossary.md#welcome-attribution-info) under the invitee clients' respective encryption keys

Finally, the client queries the [relevant DS endpoint](../delivery_service.md#adding-new-users-to-the-group). If the query is successful, the client merges the commit.
