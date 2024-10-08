# Discovery and connection establishment

Discovery allows users to find other users based on their [alias](../glossary.md#alias) or their [user id](../glossary.md#user-id-uid) and establish a connection with them. The connection establishment process establishes a *connection group* that consists of the clients of both connected users, but otherwise functions like a regular group with the exception of the group creation process.

Once two users have a *connection*, they can add one-another to groups.

The creation process of a connection group is restricted by two considerations: Until the responding user makes its decision to accept the connection, the initiator is considered untrusted and should not be able to push arbitrary content to the responder.

The second consideration is that neither of the homeservers involved (with the initiator and responder potentially belonging to two federated homeservers) should know who is connected to whom. This is not straight-forward, as the initiator has to be able to discover the responder by its alias and use the discovered information to initiate the process.

## Connection group creation

To allow connection establishment in the first place, the user's client publishes *connection packages* with the AS. Connection packages can be published either under the user's [user id](../authentication_service.html#upload-user-id-connection-packages) or any of the users' [Aliases](../authentication_service.html#upload-alias-connection-package-payloads). When published under an alias, a **connection package payload** is used rather than a full ConnectionPackage. As the aliases should not be associated with the user id of the creating user, or any other registered alias, the payloads lack the user's client's client credential and signature. Users also shouldn't use the same connection encryption key across multiple aliases and their user id.

```rust
struct ConnectionPackagePayload {
    encryption_key: ConnectionEncryptionKey,
    lifetime: ExpirationData,
}

struct ConnectionPackage {
    payload: ConnectionPackagePayload,
    client_credential: ClientCredential,
    // TBS: All information above signed by the ClientCredential.
    signature: Signature,
}
```

When discovering a user (either by [User ID](../authentication_service.html#get-user-id-connection-package) or [Alias](../authentication_service.html#get-alias-connection-package)), the initiator fetches a connection package for the user's client. The initiator then [creates a new group](../delivery_service.html#create-group) and sends a `ConnectionEstablishmentPackage` to the responder's client through either the [Alias Queue](../authentication_service.html#enqueue-alias-messages) or the [Direct User Queue](../authentication_service.html#enqueue-message).

```rust
struct ConnectionEstablishmentPackage {
    sender_client_credential: ClientCredential,
    connection_group_id: GroupId,
    connection_group_ear_key: GroupStateEarKey,
    connection_group_credential_key: CredentialEarKey,
    // TBS: All information above signed by the ClientCredential.
    signature: Signature,
}
```

The `ConnectionEstablishmentPackage` is signed using the initiator's client credential and encrypted under the `ConnectionEncryptionKey` included in the connection package.

The receiving client must verify the signature on the package and fetch the AS credential and AS intermediate credential to verify the signature chain from the initiator's client credential to the AS credential. The responder can then decide based on the sender's user name (contained in the client credential) if it wants to accept the connection.

The responder accepts the connection by fetching the [external commit information](../delivery_service.md#get-external-commit-information) required to join the group from the initiator's DS and joins the group via the [join connection group](../delivery_service.md#join-connection-group) endpoint of said DS.
