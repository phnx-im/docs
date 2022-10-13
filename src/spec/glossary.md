# Glossary

## User Name (UN)

ID of a user consisting of a String qualified with the domain of the user's AS.

```rust
struct UserName {
    as_domain: FQDN,
    user_name: String,
}
```

## Client ID (CID)

ID of a user's client, qualified by the user name of the owner's user.

```rust
struct ClientId {
    user_id: UserName,
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
    option_push_token_key: Option<EarKey>,
    pseudonymous_client_id: PseudonymousClientId
}
```

The `SealedQueueConfig` is the client's `QueueConfig` encrypted using HPKE in the asymmetrically authenticated mode using the `QueueConfigEncryptionKey` of the client's QS and the client's own [QS QS client record key](glossary.md#qs-client-record-auth-key).

## RolesExtension

An MLS group context extension that contains information on the roles of the individual members in a given group. A member's role can be either *admin* or *member*.

```rust
struct RolesExtension {
    admins: Vec<LeafIndex>,
    members: Vec<LeafIndex>,
}
```

If a user has multiple clients in a given group, all clients of that user MUST have the same role.

### Future work: Manage roles via proposal

For now, chaning roles requires a GroupContextExtension proposal, which can be expensive if there is more than one extension. Instead it should probably be possible to change roles via a custom proposal.

## Welcome attribution info

Encrypted under the recipient's friendship encryption key. The TBS has to be signed by the sender's client credential.

```rust
struct WelcomeAttributionInfoPayload {
    sender_client_id: ClientId,
    group_credential_encryption_key: EarKey,
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

A bundle allowing a client to join a new group.

```rust
struct WelcomeBundle {
    welcome: Welcome,
    encrypted_welcome_attribution_info: Vec<u8>,
    encrypted_group_state_ear_key: Vec<u8>,
    group_id: GroupId,
}
```

The WelcomeAttributionInfo is encrypted under the joining client's friendship encryption key. The group state EAR key is by the DS under the recipient's init key (contained in the recipient's KeyPackage).

## User KeyPackage batch

When a client retrieves KeyPackages from a QS for a given user, the QS responds with the KeyPackages, the associated Intermediate Client credentials, as well as a UserKeyPackageBatch.

```rust
struct UserKeyPackageBatch {
  key_package_refs: Vec<KeyPackageRef>,
  timestamp: Timestamp,
  signature: Signature,
}
```

## AddPackage

A struct consisting of a KeyPackage and the associated [Intermediate Client Credential](authentication_service/credentials.md#intermediate-client-credentials), where the latter is encrypted under the user's current [friendship encryption key](glossary.md#friendship-encryption-key).

```rust
struct AddPackage {
    key_package: KeyPackage,
    icc_ciphertext: Vec<u8>,
}
```

## Friendship keys

A set of keys known to users that the owning user has a [connection](authentication_service/connection_establishment.md) with.

### Friendship encryption key

A symmetric key used to encrypt the credential information attached to KeyPackages, as well as the WelcomeAttributionInfo. This key is never rotated.

#### Future work: Rotate friendship encryption key

Rotating the friendship encryption key can lead to annoying race conditions (e.g. a Welcome sent short before the key was rotated). If we want to rotate it, we could use key ids (see [here](./delivery_service/group_state_encryption.md)) and a grace period before an old key is not accepted anymore.

### Friendship token

A random byte string that is used by users to prove that they have a [connection](authentication_service/connection_establishment.md) with the owning user, which in turn allows them to fetch the user's key packages. Can be rotated by the owning user by updating the value on its QS and broadcasting it to all of the user's (remaining) connections.

## Client credential chain

A tuple consisting of an intermediate client credential and a client credential. Used to authenticate individual clients in the context of an MLS group. Stored on the DS encrypted under the group's [credential encryption key](delivery_service/group_state_encryption.md).

## Queue encryption key

HPKE public key associated with a client's fan-out or direct queue and used to facilitate [at-rest encryption of messages in the queue](queuing_service/queue_encryption.md).

## QS client record auth key

Signature public key associated with a QS QS client record. Used by the QS to authenticate the owning client.

## QS user record auth key

Signature public key associated with a QS user record. Used by the QS to authenticate the user's clients.

## QS signing key

Signature key used by federated QS' to authenticate the owning QS, as well as by a local or federated DS to verify signatures on [user KeyPackage batches](./glossary.md#user-keypackage-batch).

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

## Push-token encryption key

A symmetric encryption key that a client uses to encrypt its push-token on its QS QS client record. It is part of the client's encrypted [QueueConfig](glossary.md#sealed-queue-config).

## QueueConfig extension

A KeyPackage extension that contains a client's [ClientQueueConfig](glossary.md#sealed-queue-config). This extension is required for all KeyPackages uploaded to a QS.

## ConnectionGroup extension

A GroupContext extension that contains no data, but the presence of which in a group indicates that the group is a [connection group](authentication_service/connection_establishment.md).