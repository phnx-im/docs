# Authentication Service (AS)

In this chapter, we detail the different functionalitites of the authentication service. See the subchapter on [credentials](authentication_service/credentials.md) for more details on the credential types referenced in this chapter.

For an overview over the security of the AS see [here](./authentication_service/security_guarantees.md).

## Configuration

The AS is configurable by use of the following configuration variables:

* **Anonymous token timeframe:** The amount of time which each client has to wait until it can obtain new anonymous authentication tokens.
* **Default token allowance:** The default amount of anonymous authentication tokens issued to each user in the anonymous token timeframe.
* **Maximal QS client record age:** Maximal age of an inactive client entry.
  * Default: 90d
* **Maximal number of requested messages:** Maximal number of messages that will be returned to a client requesting messages from a direct queue.

## AS state

The AS generally keeps the following state

* **User entries:** A database with one entry for each user account. Each entry is indexed by the user's [user id](glossary.md#user-id-uid) and contains a number of sub-entries.
  * **OPAQUE user record:** The OPAQUE protocol artifact that allows the user to authenticate itself via its password in queries to the AS.
  * **Username:** The Username chosen by the user upon creation. It can be used by other users to discover the user.
  * **Client entries:** Sub entries for the users' clients. Indexed by the clients' [client id](glossary.md#client-id-cid).
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
* **Client Credential:** A signature over a time stamp using the signature key in the calling client's ClientCredential. The request additionally contains the calling client's ClientCredential.
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

## Authentication

* Client Credential

## Initialize user registration

The user registration functionality requires the user to perform an [OPAQUE registration flow](https://www.ietf.org/archive/id/draft-irtf-cfrg-opaque-09.html#name-offline-registration) with the homeserver. Note, that the user's chosen user id is part of the `client_csr`.

```rust
struct InitUserRegistrationParams {
  client_csr: ClientCsr,
  opaque_registration_request: OpaqueRegistrationRequest,
}
```

The AS then performs the following steps:

* check if a user entry with the name given in the `client_csr` already exists
* validate the [`client_csr`](./authentication_service/credentials.md#client-credential-signing-requests)
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

After receiving the response, the client must call the [finish user registration endpoint](./authentication_service.md#finish-user-registration).

### Authentication

* None

## Finish user registration

This endpoint allows the user to finish its registration.

```rust
struct FinishUserRegistrationParams {
  user_id: UserId,
  queue_encryption_key: HpkePublicKey,
  initial_queue_ratchet_key: RatchetKey,
  connection_packages: Vec<ConnectionPackage>,
  opaque_registration_record: OpaqueRegistrationRecord,
  username: Username,
}
```

The AS performs the following actions:

* look up the initial client's ClientCredential in the ephemeral DB based on the `user_id`
* authenticate the request using the signature key in the ClientCredential
* check (again) if the user id already exists
* check if the user's chosen username is already taken
* create the user entry with the information given in the request
* create the initial client entry
* delete the entry in the ephemeral OPAQUE DB

### Authentication

* Client 2FA (the AS has to successfully complete the OPAQUE handshake and the client needs to provide Client Credential authentication using the signature key of the initial client)

## Get user connection package

Given a username, get a [connection package](authentication_service/connection_establishment.md#connection-group-creation) for the user's client.

```rust
struct UserClientsParams {
  username: Username,
}
```

The AS returns the following information.

```rust
struct UserClientsResponse {
  connection_package: ConnectionPackage,
}
```

### Authentication

* None

## User account deletion

Delete the user account with the given user id.

```rust
struct DeleteUserParams {
  user_id: UserId,
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

* The number of messages remaining in the queue
* The value of the `max_message_number` field in the request
* The AS configured maximum number of returned messages

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

### Authentication

* None

## Token Issuance

This endpoints allows both local clients and clients of federated homeservers to retrieve Privacy Pass tokens which they can then redeem to interact with the local DS.

```rust
struct RequestToken {
  client_id: ClientId,
  token_request: TokenRequest,
}
```

`TokenRequest` represents a Privacy Pass token request.

### Rate-limiting

Token issuance is the main way of rate-limiting queries to the endpoints of the DS of a homeserver, which means that the AS should rate-limit token issuance based on a per-client basis, essentially giving each client a certain allowance of tokens.

If the calling client is a federated client, rate-limiting should also occur on a per-homeserver (or per homeserver domain) basis to protect agains malicious or negligent homeservers that allow an attacker to register a large number of clients.

### Authentication

For this endpoint, the AS also accepts authentication by federated clients. If the `client_id` in the client credential indicates that the client belongs to a federated homeserver, the AS looks up the authentication key material of that client's AS using the [corresponding endpoint](./authentication_service.md#get-as-credentials), or looks the key material up in its local cache. The AS then uses that key material to verify the query.

* Client Credential
