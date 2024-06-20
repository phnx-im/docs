# Discovery and connection establishment

Discovery allows users to find other users based on their user name and establish a connection with them. The connection establishment process establishes a *connection group* that consists of the devices of both connected users, but otherwise functions like a regular group with the exception of the group creation process.

Once two users have a *connection*, they can add one-another to groups.

The creation process of a connection group is restricted by two considerations: Until the responding user makes its decision to accept the connection, the initiator is considered untrusted and should not be able to push arbitrary content to the responder. That is with the exception of the initiator's user name which the responder can use to identify the initiator.

The second consideration is that neither of the homeservers involved (with the initiator and reponder potentially belonging to two federated homeservers) should know who is connected to whom. This is not straight-forward, as the initiator has to be able to discover the responder by its real user name and use the discovered information to initiate the process.

## Connection group creation

To allow connection establishment in the first place, every client of a user publishes a special *connection package* with the AS which contains the client's [client credential](credentials.md#client-credentials) instead of the Leaf Credential that is used in KeyPackages.

```rust
struct ConnectionPackage {
    encryption_key: ConnectionEncryptionKey,
    lifetime: ExpirationData,
    client_credential: ClientCredential,
    // TBS: All information above signed by the ClientCredential.
    signature: Signature
}
```

When discovering a user, the initiator fetches the connection packages of all of the user's clients. The next step if for the initiator to create a group that contains all of the initiator's clients. The initiator then sends a `ConnectionEstablishmentPackage` to the responder's clients.

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

The `ConnectionEstablishmentPackage`s are then signed using the initiator's client credential and each encrypted under the `ConnectionEncryptionKey` of the connection package of the responder's individual clients.

Receiving the `ConnectionEstablishmentPackage`, a responder's client first makes sure that its queue on the QS is empty. This is to ensure that the responding user hasn't already accepted the connection on another client. If there is a WelcomeBundle with the same GroupId as in the `ConnectionEstablishmentPackage`, the client uses the former to join the group and discards the `ConnectionEstablishmentPackage`.

If the queue is empty, the receiving client must verify the signature on the package. It must also fetch the AS credential and AS intermediate credential and verify the signature chain from the initiator's client credential to the AS credential. The responder can then decide based on the sender's user name (contained in the client credential) if it wants to accept the connection.

The responder accepts the connection by fetching the external commit information required to join the group from the initiator's DS and joins the group via the *join connection group* endpoint of said DS. After joining, the client adds all other clients of the same user to the group.

If joining via the *join connection group* endpoint fails, the new client must check its queue again to ensure that no other client has accepted the connection in the mean time. If the queue is still empty, the client discards the ConnectionEstablishmentPackage and aborts the process.

### Future work: Additional initiator information

In some cases, the responder might want to get additional information to validate the responder, such as a display name or a profile picture. The responder should not get this information delivered immediately, but instead it should be downloaded upon active request by the responder.
