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

## Pseudonymous User ID (PUID)

The ID of a pseudonymous user record on the user's QS.

```rust
struct PseudonymousUserId {
    user_id: UUID,
}
```

## Pseudonymous Client ID (PCID)

The ID representing a client in the owning user's pseudonymous user record on the client's QS.

```rust
struct PseudonymousClientId {
    puid: PseudonymousUserId,
    client_id: UUID,
}
```

## QueueConfig encryption key

A public HPKE key owned by a homeserver's QS. It is used by the homeserver's clients to encrypt their [sealed queue configs](glossary.md#sealed-queue-config). The QS holds the private key so that it can decrypt the sealed queue configs when receiving messages.

## Pseudonymous client record key

A public HPKE key owned by a client and stored in the client's pseudonymous record on the client's QS. Used to authenticate the client as the owner of the record.

## Sealed queue config

The sealed queue config is a struct that contains the information required by a DS to deliver messages to a client. In particular, the sealed queue config contains the following data.

```rust
struct ClientQueueConfig {
    client_homeserver_domain: FQDN,
    sealed_config: SealedQueueConfig,
}

struct QueueConfig {
    option_push_token_key: EarKey,
    pseudonymous_client_id: PseudonymousClientId
}
```

The `SealedQueueConfig` is the client's `QueueConfig` encrypted using HPKE in the asymmetrically authenticated mode using the `QueueConfigEncryptionKey` of the client's QS and the client's own [pseudonymous client record key](glossary.md#pseudonymous-client-record-key).

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

## User KeyPackage batch

When a client retrieves KeyPackage from a QS for a given user, the QS responds with a UserKeyPackageBatch.

```rust
struct UserKeyPackageBatch {
  key_package_refs: Vec<KeyPackageRef>,
  timestamp: Timestamp,
  signature: Signature,
}
```

## Friendship keys

A set of keys known to contacts of a user.

### Friendship base key

Secret shared by a user with all of its friends.

### Friendship encryption key

A key derived from the friendship base key with the following label

```rust
"mls infra friendship encryption key"
```

### Friendship token

A key derived from the friendship base key with the following label

```rust
"mls infra friendship friendship token"
```

## Client credential chain

A tuple consisting of an intermediate client credential and a client credential. Used to authenticate individual clients in the context of an MLS group. Stored on the DS encrypted under the group's [credential encryption key](delivery_service/group_state_encryption.md).