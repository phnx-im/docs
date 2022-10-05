# Friendship tokens and KeyPackage publication

To prevent a homeserver from learning which groups a user is a member of, the user's information on the QS and DS are pseudonymous. However, other users must be able to authenticate the user when one of the user's clients joins a group. This would typically happen through the Credential in a client's KeyPackage. However, since the KeyPackages are published on the QS and the public trees to which the KeyPackages are added are observable by the DS.

The goal of this mechanism is thus to protect the unlinkability between the user's pseudonymous identity (i.e. the pseudonymous user record on the QS and the user's user and client profiles on the local DS) and the user's actual identity against a malicious homeserver.

The mechanism outlined in this section does not protect against the use of traffic pattern analysis. However, it can be used in conjunction with an onion routing system or a mixnet.

## Friendship keys and credential encryption

When establishing a [connection with another user](../authentication_service/connection_establishment.md), the users exchange [friendship keys](../glossary.md#friendship-keys). This set of keys includes a friendship encryption key and a friendship token. Both keys are used in the context of KeyPackage publishing.

Possession of a friendship token authorizes a client to add the original owner of the token to a group. Once added to a group, all group members must be able to authenticate the newly added user and its clients.

However, the QS that stores the published KeyPackages that facilitate group member additions must not learn the user's identity. Thus, KeyPackages are always published as [KeyPackageTuples](../glossary.md#keypackage-tuple), where the KeyPackage only contains a pseudonymous [Leaf Credential](../authentication_service/credentials.md#leaf-credentials) and the [intermediate client credential](../authentication_service/credentials.md#intermediate-client-credentials) that links the pseudonymous credential with the client's [Client Credential](../authentication_service/credentials.md#client-credentials) is encrypted under the friendship encryption key.

After retrieving such a KeyPackageTuple (as part of a [KeyPackageBatch](../glossary.md#user-keypackage-batch)), the adding client first decrypts the encrypted intermediate client credential and verifies that the [credential chain](../glossary.md#client-credential-chain) is correct. If this is the case, the client re-encrypts the intermediate client credential under the target group's [credential encryption key](../delivery_service/group_state_encryption.md) before performing the actual addition.

Members of the group can now decrypt the intermediate client credential and authenticate the user.

Note that neither DS nor QS are in possession of either the friendship encryption key or the credential encryption key. Both QS and DS store the intermediate client credential exclusively as ciphertexts.

## Welcome attribution

The friendship encryption key is also used to allow the recipient of a Welcome message to quickly authenticate the sender while hiding the sender's identity from the DS and QS involved.

In particular, instead of a plain MLS Welcome message, new joiners receive a [WelcomeBundle](../glossary.md#welcomebundle), which (along with the Welcome and some other data) contains a [WelcomeAttributionInfo](../glossary.md#welcome-attribution-info). The WelcomeAttributionInfo in turn contains the senders [client ID](../glossary.md#client-id-cid) and is signed by the sender's [Client Credential](../authentication_service/credentials.md#client-credentials). The recipient of the WelcomeBundle can thus identify and authenticate the sender without retrieving the group's public tree first.

However, to avoid accumulation of metadata, the sender's identity must be hidden from both DS and QS. This is achieved by encrypting the WelcomeAttributionInfo in the WelcomeBundle using the recipient's friendship encryption key.

## Friendship token rotation

A user must rotate its friendship token when removing the connection with another user. The token is rotated by sampling a fresh, random token and replacing the old token on the QS. After replacing the token on the QS, the user must broadcast the new token to the rest of its own clients, as well as all users with which the user wishes to keep a connection.

### Future work: Friendship encryption key rotation

It should also be possible to rotate the friendship encryption key.

Rotating the friendship encryption key can lead to annoying race conditions (e.g. a Welcome sent shortly before the key was rotated). If we want to rotate it, we could use key ids (see [here](./delivery_service/group_state_encryption.md)) and a grace period before an old key is not accepted anymore.

### Future work: Replace the friendship token

It is conceivable that the frienship token can be replaced with a better mechanism that allows for easier rotation. It is conceivable that puncturable signatures or blocklistable anonymous credentials could be useful here.
