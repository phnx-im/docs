# Glossary

## User ID (UID)

ID of a user consisting of a UUID qualified with the domain of the user's AS.

```rust
struct UserId {
    as_domain: FQDN,
    user_id: Uuid,
}
```

## Client ID (CID)

ID of a user's client, qualified by the user id of the owner's user.

```rust
struct ClientId {
    user_id: UserId,
    client_uuid: UUID,
}
```

## QS User ID (QsUid)

The ID of a QS user record on the user's QS.

```rust
struct QsUid {
    user_id: UUID,
}
```

## QS client ID (QsCid)

The ID representing a client in the owning user's QS user record.

```rust
struct PseudonymousClientId {
    qs_uid: QsUid,
    client_id: UUID,
}
```

## Alias

A string that a user registers anonymously with the AS. Other users can use it to discover that user and [establish a connection with them](./authentication_service/connection_establishment.md).

## Alias auth key

A public signature key used by the user who registered the associated alias with the AS to authenticate itself, for example, when dequeuing messages or when uploading connection packages.

## User profile

Personal user data that represents the user towards other users in the context of groups. For now, the user profile only contains the user's [display name](./glossary.md#display-name). The user profile is stored on the AS, encrypted under the user's user profile encryption key.

## User profile encryption key

A symmetric key used to encrypt or decrypt a user's user profile. It is held by the user itself and provisioned ([encrypted](./delivery_service/group_state_encryption.md#user-profile-key-encryption)) to the DS as part of the DS' group state.

## Display name

A string that is shown to other users that a user is in a conversation with. The display name does not have to be unique.

## QueueConfig encryption key

A public HPKE key owned by a homeserver's QS. It is used by the homeserver's clients to encrypt their [sealed queue configs](glossary.md#sealed-queue-config). The QS holds the private key so that it can decrypt the sealed queue configs when receiving messages.

## Sealed queue config

The sealed queue config is a struct that contains the information required by a DS to deliver messages to a client. In particular, the sealed queue config contains the following data.

```rust
struct ClientQueueConfig {
    client_homeserver_domain: FQDN,
    sealed_config: SealedQueueConfig,
}

struct QueueConfig {
    option_push_token_key: Option<PushTokenEarKey>,
    pseudonymous_client_id: PseudonymousClientId,
}
```

The `SealedQueueConfig` is the client's `QueueConfig` encrypted using HPKE in the asymmetrically authenticated mode using the [`QueueConfigEncryptionKey`](#queueconfig-encryption-key) of the client's QS and signed using the client's own [QS client record key](glossary.md#qs-client-record-auth-key).

## Queue ID encryption key

An HPKE public key that is stored by the DS alongside encrypted group states. It is used to encrypt any sealed queue configs returned by a QS during message fanout. The corresponding private key is part of the encrypted group state.

## RolesExtension

An MLS group context extension that contains information on the roles of the individual members in a given group. A member's role can be either *admin* or *member*.

```rust
struct RolesExtension {
    admins: Vec<LeafIndex>,
    members: Vec<LeafIndex>,
}
```

## Welcome attribution info

Encrypted under the recipient's friendship encryption key. The TBS has to be signed by the sender's client credential.

```rust
struct WelcomeAttributionInfoPayload {
    sender_client_id: ClientId,
    group_credential_encryption_key: GroupCredentialEarKey,
}

struct WelcomeAttributionInfoTbs {
    payload: WelcomeAttributionInfoPayload,
    group_id: GroupId,
    welcome: Vec<u8>,
}

struct WelcomeAttributionInfo {
    payload: WelcomeAttributionInfoPayload,
    signature: Signature,
}
```

## WelcomeBundle

A bundle allowing a client to join a new group with a [Welcome](https://datatracker.ietf.org/doc/html/rfc9420#name-joining-via-welcome-message) message.

```rust
struct WelcomeBundle {
    welcome: Welcome,
    encrypted_welcome_attribution_info: Vec<u8>,
    encrypted_group_state_ear_key: Vec<u8>,
    encrypted_credential_encryption_key: Vec<u8>,
    group_id: GroupId,
}
```

The WelcomeAttributionInfo is encrypted under the joining client's friendship encryption key. The group state EAR key is encrypted by the DS under the recipient's init key (contained in the recipient's KeyPackage).

## AddPackage

A struct consisting of a KeyPackage, the associated [Client Credential](authentication_service/credentials.md#client-credentials) encrypted under the [Friendship Encryption Key](glossary.md#friendship-encryption-key), and a freshly generated [Signature Encryption Key](glossary.md#signature-encryption-key) encrypted under the [Friendship Encryption Key](glossary.md#friendship-encryption-key).

```rust
struct AddPackage {
    key_package: KeyPackage,
    encrypted_client_credential: Vec<u8>,
    encrypted_signature_encryption_key: Vec<u8>,
}
```

### Signature encryption key

The signature encryption key is used to encrypt the signature in LeafCredentials. These signatures are encrypted to prevent the DS from linking LeafCredentials across groups. Before an AddPackage is used in the context of a group, the inviting client decrypts the signature encryption key and re-encrypts it under the group's credential encryption key. See [here](./queuing_service/keypackage_publication.md#friendship-keys-and-credential-encryption) for more information.

## Friendship keys

A set of keys known to users that the owning user has a [connection](authentication_service/connection_establishment.md) with.

### Welcome Attribution Info encryption key

A symmetric key used to encrypt information in the [Welcome Attribution Info](#welcome-attribution-info).

### Friendship encryption key

A symmetric key used to encrypt the client credential and the [signature encryption key](glossary.html#signature-encryption-key) attached to AddPackages.

### Friendship token

A random byte string that is used by users to prove that they have a [connection](authentication_service/connection_establishment.md) with the owning user, which in turn allows them to fetch the user's key packages. Can be rotated by the owning user by updating the value on its QS and broadcasting it to all of the user's (remaining) connections.

## Queue encryption key

HPKE public key associated with a client's fan-out or direct queue and used to facilitate [at-rest encryption of messages in the queue](queuing_service/queue_encryption.md).

## QS client record auth key

Signature public key associated with a QS client record. Used by the QS to authenticate the owning client.

## QS user record auth key

Signature public key associated with a QS user record. Used by the QS to authenticate the user's clients.

## QS signing key

Signature key used by federated QS' to authenticate the owning QS.

## Fan out message

Message sent from the DS to its local QS, either for forwarding to another QS or for enqueuing into a local queue.

```rust
enum MessageContent {
    Commit(MlsMessage),
    Application(MlsMessage),
    WelcomeBundle(WelcomeBundle),
}

struct FanOutMessage {
    client_queue_configs: Vec<ClientQueueConfig>,
    message_content: MessageContent,
}
```

## Last resort extension

A KeyPackage extension that marks the given KeyPackage as a [KeyPackage of last resort](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#name-keypackage-reuse).
While ordinary KeyPackages are meant to be used only once, a Last Resort
KeyPackage is meant to be used multiple times when no further ordinary
KeyPackages are available.

## Push-token encryption key

A symmetric encryption key that a client uses to encrypt its push-token on its QS client record. It is part of the client's encrypted [QueueConfig](glossary.md#sealed-queue-config).

## QueueConfig extension

A KeyPackage extension that contains a client's [ClientQueueConfig](glossary.md#sealed-queue-config). This extension is required for all KeyPackages uploaded to a QS.
