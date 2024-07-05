# Creation

User creation across the different services processes in multiple stages:

1. Initiate AS registration
2. Finish AS registration
3. QS registration
4. KeyPackage upload

## Initiate AS registration

To initialize AS registration, the user has to provide

- a password
- a (desired) [user name](../glossary.md#user-name-un), which includes the domain of the provider

The client can then fetch the [credentials of the providers AS](../authentication_service.md#get-as-credentials) and the [verifying key of the provider's QS](../queuing_service.md#fetch-qs-verifying-key).

Following that, the client

- validates the AS intermediate credentials and picks one
- generates a UUID for the client's [client id](../glossary.md#client-id-cid) and
- creates a [client credential CSR](../authentication_service/credentials.md#client-credential-signing-requests).

Finally, the client queries the AS to [initiate registration](../authentication_service.md#initialize-user-registration).

## Finish AS registration

Before finishing AS registration, the client freshly generates the following key material:

- Encryption keypairs and initial symmetric encryption keys for [AS and QS queue ratchets](../queuing_service/queue_encryption.md#queue-encryption).
- QS [user](../glossary.md#qs-user-record-auth-key) and [client record authentication keypairs](../glossary.md#qs-client-record-auth-key)
- Friendship key material
  - [Friendship token](../glossary.md#friendship-token)
  - [Friendship encryption key](../glossary.md#friendship-encryption-key)
  - [Welcome Attribution Info encryption key](../glossary.md#welcome-attribution-info-encryption-key)
  - [Push token encryption key](../glossary.md#push-token-encryption-key)
- [Connection encryption keypair](../authentication_service/connection_establishment.md#connection-group-creation)

The client then generates a number of [connection packages](../authentication_service/connection_establishment.md#connection-group-creation) and finishes AS registration using the [appropriate endpoint](../authentication_service.md#finish-user-registration).

## QS registration

With the key material generated in the previous step, the client already has all the information required to call the [qs user registration endpoint](../queuing_service.md#create-new-qs-user-record). The client then stores the returned QS [user](../glossary.md#qs-user-id-qsuid) and [client ID](../glossary.md#qs-client-id-qscid)s.

## KeyPackage upload

Finally, the client generates a number of KeyPackages, of which at least one marked for use in [last-resort](../glossary.md#last-resort-extension), which the client then [uploads to the QS](../queuing_service.md#publish-keypackages).
