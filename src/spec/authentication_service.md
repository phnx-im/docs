# Authentication Service (AS)

In this chapter, we detail the different functionalitites of the authentication service. See the subchapter on [evolving identities (EIDs)](./authentication_service/evolving_identities.md) for more details on the evolving identities mechanism for client management and authentication.

## Configuration

The AS is configurable by use of the following configuration variables:

* **Anonymous token timeframe:** The amount of time which each client has to wait until it can obtain new anonymous authentication tokens.
* **Default token allowance:** The default amount of anonymous authentication tokens issued to each user in the anonymous token timeframe.
* **Identity provider:** Network address of the OpenID connect identity provider the authentication service delegates authentication to for account registration.

TODO: Be specific about technical details for OIDC provider (OIDC provider ID and secret)

## AS state

The AS generally keeps the following state

* **Account database:** A database with one entry for each user account. Each entry is indexed by the user's user name and contains a number of sub-entries.
  * **Evolving Identity:** The evolving identity state of the user.
  * **Token issuance records:** A record of how many tokens were issued to each of the clients of the user.

## User account registration

The user registration functionality delegates authentication to an OpenID connect-capable identity provider. In particular, it requests the scopes *openid* (for authentication).

* **Future work:** In the organizational messaging case, where there are fewer privacy concerns, the AS could request more information from the IdP and, for example, use an existing email address as user name, an existing nickname as displayname and an existing image as user profile picture.

After the user authenticates to the IdP, the AS stores the user's IdP identity (`sub` in the openid claim) along with the user's user name.

After registration of the data above, the user registers its first client. The client creates a credential submits it for signature by the AS (see [here](./authentication_service.md#credential-signature-requests-csrs)). The AS validates the credential, signs it and returns it to the client. The client then sends the initial EID state to the AS, which it again validates and stores.

Finally, the client requests anonymous tokens so that it can create its queues at the queuing service and create its all-clients group.

TODO: We probably want a bearer token to authenticate towards the AS for anonymous auth token retrieval, profile changes and other directly authenticated endpoints. I'm not sure, but I think we get this from the IdP.
TODO: Add links to definition of credentials and information on how they are verified, as well as to the token issuance section.

## User account deletion

This functionality allows users to log in with an existing client and delete their account. The AS can only delete the user data it has stored itself. The client performing the deletion MUST in addition delete any queues owned by clients of the user.

## User account reset/recovery

TODO: Document account recovery, either via an exportable recovery key ("paper backup"), or just via login.

## Evolving identities submissions

The EID submission functionality allows a client to submit an MLS message, which the AS validates and uses to update the EID state of the user. See [here](./authentication_service/evolving_identities.md) for more details.

## Look up EID

The EID look up functionality is provided to other clients, both local and federated. Given a user name, the AS will check if there exists an EID matching that user name. If that is the case, it will return the EID. If not, it will respond that the given user doesn't exist.

