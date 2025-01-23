# Friendship tokens and KeyPackage publication

To prevent a homeserver from learning which groups a user is a member of, the user's information on the QS and DS are pseudonymous. However, other users must be able to authenticate the user when the user's client joins a group. This would typically happen through the Credential in a client's KeyPackage. However, since the KeyPackages are published on the QS and the public trees to which the KeyPackages are added are observable by the DS, the client credential can't be included in the KeyPackage.

The goal of this mechanism is thus to protect the unlinkability between the user's pseudonymous identity (i.e. the QS user record and the user's user and client profiles on the local DS) and the user's actual identity against a malicious homeserver.

The mechanism outlined in this section does not protect against the use of traffic pattern analysis. However, it can be used in conjunction with an onion routing system or a mixnet.

## Friendship keys and identity links

When establishing a [connection with another user](../authentication_service/connection_establishment.md), the users exchange [friendship keys](../glossary.md#friendship-keys). This set of keys includes a friendship encryption key and a friendship token. Both keys are used in the context of KeyPackage publishing.

Possession of a friendship token authorizes a client to add the original owner of the token to a group. Once added to a group, all group members must be able to authenticate the newly added user via its client.

However, the QS that stores the published KeyPackages to facilitate group member additions must not learn the user's identity. A KeyPackage thus contains a [pseudonymous credential](../authentication_service/credentials.md#pseudonymous-credentials), which contains an identity link encrypted under a key derived from the user's friendship key. The identity link in turn contains a signature by the uploading client's client credential.

After retrieving a KeyPackage, the adding client decrypts the identity link and verifies the signature over the [PseudonymousCredential](../authentication_service/credentials.md#pseudonymous-credentials). If verification is successful, the client encrypts the identity link key under the target group's [identity link wrapper key](../delivery_service/group_state_encryption.md) before performing the actual addition.

As all members of a group are in possession of the identity link wrapper key, they can decrypt both the identity link key and subsequently the identity link and thus authenticate the added client.

Note that neither DS nor QS are in possession of either the friendship encryption key or the identity link wrapper key.

## Welcome attribution

The friendship encryption key is also used to allow the recipient of a Welcome message to quickly authenticate the sender while hiding the sender's identity from the DS and QS involved.

In particular, instead of a plain MLS Welcome message, new joiners receive a [WelcomeBundle](../glossary.md#welcomebundle), which (along with the Welcome and some other data) contains a [WelcomeAttributionInfo](../glossary.md#welcome-attribution-info). The WelcomeAttributionInfo in turn contains the senders [client ID](../glossary.md#client-id-cid) and is signed by the sender's [Client Credential](../authentication_service/credentials.md#client-credentials). The recipient of the WelcomeBundle can thus identify and authenticate the sender without retrieving the group's public tree first.

However, to avoid leakage of metadata, the sender's identity must be hidden from both DS and QS. This is achieved by encrypting the WelcomeAttributionInfo in the WelcomeBundle using the recipient's friendship encryption key.

## Friendship token rotation

A user must rotate its friendship token when removing the connection with another user. The token is rotated by sampling a fresh, random token and replacing the old token on the QS. After replacing the token on the QS, the user must broadcast the new token to all users with which the user wishes to keep a connection.
