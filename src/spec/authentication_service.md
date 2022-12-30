# Authentication Service (AS)

In this chapter, we detail the different functionalitites of the authentication service. See the subchapter on [credentials](authentication_service/credentials.md) for more details on the credential types referenced in this chapter.

The AS detailed in this section represents only the base-line version. A more sophisticated version with additional features such as better cross-signing and transparency is work in progress.

For an overview over the security of the AS see [here](./authentication_service/security_guarantees.md).

## Configuration

The AS is configurable by use of the following configuration variables:

* **Anonymous token timeframe:** The amount of time which each client has to wait until it can obtain new anonymous authentication tokens.
* **Default token allowance:** The default amount of anonymous authentication tokens issued to each user in the anonymous token timeframe.
* **Identity provider:** Network address of the OpenID connect identity provider the authentication service delegates authentication to for account registration.
* **Maximal QS client record age:** Maximal age of an inactive client entry.
  * Default: 90d
* **Maximal number of requested messages:** Maximal number of messages that will be returned to a client requesting messages from a direct queue.

## AS state

The AS generally keeps the following state

* **User entries:** A database with one entry for each user account. Each entry is indexed by the user's [user name](glossary.md#user-name-un) and contains a number of sub-entries.
  * **OPAQUE user record:** The OPAQUE protocol artifact that allows the user to authenticate itself via its password in queries to the AS.
  * **Client entries:** Sub entries for each of the user's clients. Indexed by the clients' [client id](glossary.md#client-id-cid).
    * **Client credential:** The [credential](authentication_service/credentials.md#client-credentials) of the client.
    * **Token issuance records:** A record of how many tokens were issued to the client.
    * **Activity time:** Timestamp indicating the last time a client has fetched messages from the queue.
    * **Connection establishment KeyPackage:** A key package used to encrypt information in the [connection establishment process](authentication_service/connection_establishment.md).
    * **Direct queue:** A queue similar to the fan-out queues on the QS.
      * **Queue encryption key material:** Key material to perform [queue encryption](./queuing_service/queue_encryption.md).
        * **Queue encryption key:** HPKE public key of the queue owner
        * **Encryption ratchet key:** Symmetric key used to derive queue encryption keys.
      * **Current sequence number:** The current message sequence number.
      * **Queued messages:** A sequence of ciphertexts containing the messages in the queue. Each incoming message is [encrypted](./queuing_service/queue_encryption.md) and is assigned the current sequence number, after which the current sequence number is incremented.
* **Ephemeral OPAQUE DB:** An in-memory database that stores user-name or client-id indexed entries
  * **Client credential:** Credential of the client that is being added to an existing user entry, or the initial client of a new user entry.
  * **Optional OPAQUE state:** OPAQUE server state, only present in case of the OPAQUE online AKE flow, i.e. if a client performs 2FA, but not during an OPAQUE setup.
* **AS credential key material:** Credentials that can be retrieved by clients, as well as the private key material used by the AS to sign client [CSRs](authentication_service/credentials.md#client-credential-signing-requests).
* **AS OPAQUE key material:** OPRF seed, server public key and server private key as required for the server to perform OPAQUE registration a login flows with clients.

## Authentication

There are three modes of authentication for endpoints on the AS.

* **None:** For endpoints that are meant to be publicly accessible, e.g. user account registration
* **Client Credential:** A signature over a time stamp using the signature key in the calling client's ClientCredential.
* **Client Password:** An OPAQUE login flow.
* **Client 2FA:** The same as **Client**, but additionally performing an [OPAQUE](https://datatracker.ietf.org/doc/draft-irtf-cfrg-opaque/) login flow

## Initiate 2FA operation

The client has to query this endpoint before it can query an endpoint that requires **Client 2FA** authentication. Note that this endpoint is not meant to be used with endpoints that require **Client Password** authentication such as the client addition endpoint. Similarly, this endpoint is not meant to be used to set up 2FA (see [here](./authentication_service.md#initialize-user-registration) instead).

```rust
struct Initiate2FaAuthenticationParams {
  client_id: ClientId,
  opaque_ke1: OpaqueKe1,
}
```

The AS then performs the following steps:
* perform the [first (server side) step in the OPAQUE online AKE handshake](https://www.ietf.org/archive/id/draft-irtf-cfrg-opaque-09.html#name-online-authenticated-key-exc)
* store the resulting OPAQUE `server_state` in its ephemeral DB
* return the OPAQUE response to the client

```rust
struct Initiate2FaAuthenticationResponse {
  opaque_ke2: OpaqueKe2,
}
```

### Future work: Client and query binding

The query to initialize the 2FA operation should be tied to the query that actually requires 2FA. Also, the OPAQUE handshake should be bound to the client id and the 2FA endpoint.

### Future work: Prevent client enumeration attacks

As noted [in the OPAQUE RFC](https://www.ietf.org/archive/id/draft-irtf-cfrg-opaque-09.html#name-client-enumeration) the server should take measures against client enumeration attacks. We should implement them as described in the RFC.

## Authentication

* Client Credential

## Initialize user registration

The user registration functionality requires the user to perform an [OPAQUE registration flow](https://www.ietf.org/archive/id/draft-irtf-cfrg-opaque-09.html#name-offline-registration) with the homeserver. Note, that the user's chosen user name is part of the `client_csr`.

```rust
struct InitUserRegistrationParams {
  client_csr: ClientCsr,
  opaque_registration_request: OpaqueRegistrationRequest,
}
```

The AS then performs the following steps:
* check if a user entry with the name given in the `client_csr` already exists
* validate the `client_csr`
* perform the [first (server side) step in the OPAQUE registration handshake](https://www.ietf.org/archive/id/draft-irtf-cfrg-opaque-09.html#name-createregistrationrequest)
* sign the CSR
* store the freshly signed ClientCredential in the ephemeral DB
* return the ClientCredential to the client along with the OPAQUE response.

```rust
struct InitUserRegistrationResponse {
  client_credential: ClientCredential,
  opaque_registration_response: OpaqueRegistrationResponse,
}
```

After receiving the response, the client must call the [finalize user registration endpoint](./authentication_service.md#finish-user-registration).

### Authentication

* None

## Finish user registration

This endpoint allows the user to finish its registration.

```rust
struct FinishUserRegistrationParams {
  user_name: UserName,
  queue_encryption_key: HpkePublicKey,
  connection_key_package: KeyPackage,
  opaque_registration_record: OpaqueRegistrationRecord,
}
```

The AS performs the following actions:
* look up the initial client's ClientCredential in the ephemeral DB based on the `user_name`
* authenticate the request using the signature key in the ClientCredential
* check (again) if the user entry exists
* create the user entry with the information given in the request
* create the initial client entry
* delete the entry in the ephemeral OPAQUE DB

### Authentication

* Client 2FA (the AS has to successfully complete the OPAQUE handshake and the client needs to provide Client Credential authentication using the signature key of the initial client)

### Future work: Sybil attack protection

Being able to create arbitrarily many users can enable a number of DDoS attacks and effectively renders the anonymous token DDoS prevention strategy useless. Thus we need some form of restriction here, such as a CAPTCHA or other proof of personhood.

## Get user clients

Given a user name, get the [client credentials](authentication_service/credentials.md#client-credentials) of all of the user's clients.

```rust
struct UserClientsParams {
  user_name: UserName,
}
```

The AS returns the following information.

```rust
struct UserClientsResponse {
  client_credentials: Vec<KeyPackage>,
}
```

### Authentication

* None

## User account deletion

Delete the user account with the given user name.

```rust
struct DeleteUserParams {
  user_name: UserName,
  opaque_ke3: OpaqueKe3,
}
```

The AS performs the following actions:
* look up the OPAQUE `server_state` in the ephemeral DB based on the `client_id`
* authenticate the request using the signature key in the ClientCredential
* perform the `ServerFinish` step of the OPAQUE online AKE flow
* delete the user entry
* delete the entry in the ephemeral OPAQUE DB

### Authentication

* Client 2FA

## Initiate client addition

Add a new client entry to the user's user entry.

```rust
struct InitiateClientAdditionParams {
  client_csr: ClientCsr,
  opaque_ke1: OpaqueKe1,
}
```

The AS then performs the following steps:
* check if a client entry with the name given in the `client_csr` already exists for the user
* validate the `client_csr`
* perform the [first (server side) step in the OPAQUE online AKE handshake](https://www.ietf.org/archive/id/draft-irtf-cfrg-opaque-09.html#name-online-authenticated-key-exc)
* sign the CSR
* store the resulting OPAQUE `server_state` in its ephemeral DB (along with the freshly signed ClientCredential)
* return the ClientCredential to the client along with the OPAQUE response.

```rust
struct InitiateClientAdditionResponse {
  client_credential: ClientCredential,
  opaque_ke2: OpaqueKe2,
}
```

After calling this endpoint, the client must call the ["finish client addition endpoint"](./authentication_service.md#finish-client-addition) to finish the client addition.

### Authentication

* Client Password

## Finish client addition

```rust
struct FinishClientAdditionParams {
  client_id: ClientId,
  queue_encryption_key: HpkePublicKey,
  connection_key_package: KeyPackage,
  opaque_ke3: OpaqueKe3,
}
```

The AS performs the following actions:
* look up the new client's ClientCredential, as well as the OPAQUE `server_state` in the ephemeral DB based on the `client_id`
* authenticate the request using the signature key in the ClientCredential
* perform the `ServerFinish` step of the OPAQUE online AKE flow
* check (again) if the client entry exists in the DB
* create the new client entry
* delete the entry in the ephemeral OPAQUE DB

### Authentication

* Client 2FA (checking the signature via lookup in the ephemeral DB, see above)

## Delete client

Delete the client with the given client id. This endpoint can't be used to delete the last client entry in a given user entry.

```rust
struct DeleteClientParams {
  client_id: ClientId,
}
```

### Authentication

* Client

## Dequeue messages

Dequeue messages from a client's direct queue, starting with the message with the given sequence number.

```rust
struct DequeueMessagesParams {
  client_id: ClientId,
  sequence_number_start: u64,
  max_message_number: u64,
}
```

The AS deletes messages older than the given sequence number and returns messages starting with the given sequence number. The maximum number of messages returned this way is the smallest of the following values.

- The number of messages remaining in the queue
- The value of the `max_message_number` field in the request
- The AS configured maximum number of returned messages

### Authentication

* Client

## Enqueue message

Enqueue a message into a client's direct queue.

```rust
struct EnqueueMessageParams {
  client_id: ClientId,
  connection_establishment_ctxt: Vec<u8>,
}
```

### Authentication

* None

## Get AS credentials

Get the currently valid [AS credentials](authentication_service/credentials.md#as-credentials) and [AS intermediate credentials](authentication_service/credentials.md#as-intermediate-credentials).

```rust
struct AsCredentialsResponse {
  as_credentials: Vec<AsCredentials>,
  as_intermediate_credentials: Vec<AsIntermediateCredential>,
  revoked_certs: Vec<Fingerprint>,
}
```

## Future work: Evolving Identity

For now, the AS relies on [client credential chains](glossary.md#client-credential-chain), but in the future, client authentication should be achieved using [evolving identity](authentication_service/evolving_identities.md).