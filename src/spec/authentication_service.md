# Authentication Service (AS)

In this chapter, we detail the different functionalitites of the authentication service. See the subchapter on [credentials](authentication_service/credentials.md) for more details on the credential types referenced in this chapter.

## Configuration

The AS is configurable by use of the following configuration variables:

* **Anonymous token timeframe:** The amount of time which each client has to wait until it can obtain new anonymous authentication tokens.
* **Default token allowance:** The default amount of anonymous authentication tokens issued to each user in the anonymous token timeframe.
* **Identity provider:** Network address of the OpenID connect identity provider the authentication service delegates authentication to for account registration.
* **Maximal client record age:** Maximal age of an inactive client entry.
  * Default: 90d
* **Maximal number of requested messages:** Maximal number of messages that will be returned to a client requesting messages from a direct queue.

## AS state

The AS generally keeps the following state

* **User entries:** A database with one entry for each user account. Each entry is indexed by the user's [user name](glossary.md#user-name-un) and contains a number of sub-entries.
  * **User IdP identity:** The user's identity at the user's IdP. Required for authentication with the IdP.
  * **Client entries:** Sub entries for each of the user's clients. Indexed by the clients' [client id](glossary.md#client-id-cid).
    * **Client credential:** The [credential](authentication_service/credentials.md#client-credentials) of the client.
    * **Token issuance records:** A record of how many tokens were issued to the client.
    * **Activity time:** Timestamp indicating the last time a client has fetched messages from the queue.
    * **Direct queue:** A queue similar to the fan-out queues on the QS.
      * **Queue encryption key material:** Key material to perform [queue encryption](./queuing_service/queue_encryption.md).
        * **Queue encryption key:** HPKE public key of the queue owner
        * **Encryption ratchet key:** Symmetric key used to derive queue encryption keys.
      * **Current sequence number:** The current message sequence number.
      * **Queued messages:** A sequence of ciphertexts containing the messages in the queue. Each incoming message is [encrypted](./queuing_service/queue_encryption.md) and is assigned the current sequence number, after which the current sequence number is incremented.

## Authentication

The AS uses the user's IdP to authenticate the user. Some endpoints require simple authentication, while for others, the AS requires MFA authentication via the IdP.

TODO: Be more specific here. Which token, what scope, etc.
TODO: We probably want a bearer token to authenticate towards the AS for anonymous auth token retrieval, profile changes and other directly authenticated endpoints. I'm not sure, but I think we get this from the IdP.
TODO: Structure the following endpoints.

## User account registration

The user registration functionality delegates authentication to an OpenID connect-capable identity provider. Note, that the user's chosen user name is part of the `client_csr`.

```rust
struct CreateUserParams {
  idp_identity: Vec<u8>,
  client_csr: ClientCsr,
  queue_encryption_key: HpkePublicKey,
}
```

The AS uses the given `idp_identity` to authenticate the user with the IdP and then creates the user entry and initial client entry. The AS then signs the CSR and returns it to the client.

```rust
struct CreateUserResponse {
  client_credential: ClientCredential,
}
```

### Future work: Sybil attack protection

Being able to create arbitrarily many users can enable a number of DDoS attacks and effectively renders the anonymous token DDoS prevention strategy useless. Thus we need some form of restriction here, such as a CAPTCHA or other proof of personhood.

## Get user clients

Given a user name, get the [client credentials](authentication_service/credentials.md#client-credentials) of all of the user's clients.

```rust
struct GetClientCredentialsParams {
  user_name: UserName,
}
```

The AS returns the following information.

```rust
struct GetClientCredentialsResponse {
  client_credentials: Vec<ClientCredential>,
}
```

## User account deletion

Delete the user account with the given user name.

```rust
struct DeleteUserParams {
  user_name: UserName,
}
```

### Authentication

Requires MFA authentication as the user's `idp_identity` with the IdP.

## Add new client

Add a new client entry to the user's user entry.

```rust
struct CreateClientParams {
  idp_identity: Vec<u8>,
  client_csr: ClientCsr,
  queue_encryption_key: HpkePublicKey,
}
```

The AS validates the CSR, creates the client entry, signs the CSR and returns it to the client.

```rust
struct CreateClientResponse {
  client_credential: ClientCredential,
}
```

### Authentication

Requires MFA authentication as the user's `idp_identity` with the IdP.

## Delete client

Delete the client with the given client id. This endpoint can't be used to delete the last client entry in a given user entry.

```rust
struct DeleteClientParams {
  client_id: ClientId,
}
```

### Authentication

Requires MFA authentication as the user's `idp_identity` with the IdP.

## Get client entry

Get the information contained in a client entry.

```rust
struct GetClientEntryParams {
  client_id: ClientId,
}
```

The AS responds with the following struct.

```rust
struct GetClientEntryResponse {
  client_credential: ClientCredential,
  queue_encryption_key: HpkePublicKey,
}
```

### Authentication

Requires authentication as the user's `ipd_identity` with the IdP.

## Dequeue messages

Dequeue messages from a client's direct queue, starting with the message with the given sequence number.

```rust
struct DequeueMessagesParams {
  client_id,
  sequence_number_start: u64,
  max_message_number: u64,
}
```

The AS deletes messages older than the given sequence number and returns messages starting with the given sequence number. The maximum number of messages returned this way is the smallest of the following values.

- The number of messages remaining in the queue
- The value of the `max_message_number` field in the request
- The AS configured maximum number of returned messages

## Enqueue message

TODO

## Get AS credentials

Get the currently valid [AS credentials](authentication_service/credentials.md#as-credentials) and [AS intermediate credentials](authentication_service/credentials.md#as-intermediate-credentials).

```rust
struct GetCredentialsResponse {
  as_credentials: Vec<AsCredentials>,
  as_intermediate_credentials: Vec<AsIntermediateCredential>,
}
```

### Future work: Revocation

For now, revocation happens by the AS stopping to publish a given cert. In the future, we might want a more sophisticated revocation story closer to what's happening in the WebPKI.

## Future work: Evolving Identity

For now, the AS relies on [client credential chains](glossary.md#client-credential-chain), but in the future, client authentication should be achieved using [evolving identity](authentication_service/evolving_identities.md).