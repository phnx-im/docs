# Comparison of existing applications

Different messaging applications make use of one or more of the authentication approaches, sometimes with small variations to the general concepts described above.

## Multi-client or composed user identities

Messaging applications often allow the use of multiple clients (e.g. on different devices) by a single user. However, users typically authenticate other users rather than individual clients. The applications thus have to compose individual client identities into user identities.

Compared to simple, single-client user identities, composed user identities pose additional challenges.

### Ghost clients

In many cases the messaging provider informs other users if a new client is added for a given user. A compromised or malicious provider can thus insert one of its own clients into the conversation without the target user noticing. Techniques like cross-signing can prevent such an attack, since any client additions have to be performed by one of the user's existing clients. Alternatively, [identity gossiping](authentication_systems.md#identity-gossiping), as well as OOB verification help detect such an attack.

### Revocation suppression

Users need to be able to remove lost or potentially compromised devices from their composed identity. Here, the problem is often that a malicious messaging provider can (silently) drop messages performing such a revocation operation. This can not be prevented entirely, since the messaging provider can always decide to drop messages. [Identity gossiping](./authentication_systems.md#identity-gossiping) can help mitigate this problem, as it prevents the messaging provider from dropping messages selectively. Instead, the provider is forced to drop all of a user's messages, thus significantly increasing the risk of detection.

Such an attack is also detected if two affected users verify one-another's identities OOB.

### Identity Gossiping

Some of the techniques discussed above (such as cross signing) help in ensuring that all participants of a given conversation agree on their respective identities. However, if the messaging protocol (cryptographically) includes the identity of a user in messages it sends, it can help mitigate the two attacks described above.

If the authentication systems includes a [verifyable data structure](authentication_systems.md#verifiable-data-structures) that records fingerprints of client identities, clients can also gossip their view of the data structure.

## Signal

The Signal app takes the TTP approach to authentication with [optional OOB verification by the user](https://support.signal.org/hc/en-us/articles/360007060632-What-is-a-safety-number-and-why-do-I-see-that-it-changed-). Users can discover other users via the Signal servers. Once a connection is established with another user, the server provides both users with the cryptographic identities of their peers. The same is true if a user joins a group with other users with whom it doesn’t necessarily have a previous connection.

Additionally, users can verify the identity of other users out-of-band, either by comparing a numerical code or by scanning a QR code. Notably, the code presented by the Signal app is not the cryptographic identity of a specific user, but instead a byte string specific to the connection between the users.

### Multi-client

Signal provisions all clients of a user with the same cryptographic identity, thus allowing users to perform OOB verification a single time to verify the cryptographic identity of all of the clients of a given user.

Since all clients have the same identity and thus can’t be told apart cryptographically, there is no way to revoke the individual cryptographic identity of a client. However, client removal is still possible with the assistance of the Signal server, which manages the message queues of individual clients.

## WhatsApp

WhatsApp uses the same general approach to authentication as Signal. Cryptographic identities are distributed via the WhatsApp servers, which act as TTPs. Users can additionally verify the safety numbers OOB, with the app providing the ability to scan another user’s safety number via a QR code. However, safety numbers are computed differently than by the Signal app, which is due to WhatsApp’s different approach to multi-client.

### Multi-client

Even though both applications share the underlying protocol, WhatsApp takes a different approach to multiple clients than Signal. Instead of sharing a cryptographic identity, each client has its own, distinct identity. Consequently, the [safety number](https://faq.whatsapp.com/791574747982248/?locale=en_US) for a pair of users is computed from the set of clients of both users. Thus, when comparing safety numbers OOB, users also verify their view of each other’s clients.

If a client is added, WhatsApp takes a [cross-signing approach](https://engineering.fb.com/2021/07/14/security/whatsapp-multi-device/) in addition to the TTP approach with optional OOB verification. When adding a new client, an existing client is required to sign the cryptographic identity of the new one. This does not happen automatically but instead requires the user to scan the identity of the new client using the existing one. The signed identity of the new client is then sent to the peers of the user. The user’s peers can thus trust the new client as much as it trusted the existing client. In particular, if user and peer previously verified safety codes OOB, they do not have to do so again after the addition of the new client.

Similarly, the user can revoke its own clients with an existing client authenticating the revocation.

## Keybase

Keybase keeps a [signature chain](https://book.keybase.io/docs/teams/sigchain) as the user’s cryptographic identity. That signature chain contains the identities of all of the users’ clients and can, optionally, include other “proofs of identity”, such as a link to a social media account. This link is verified by the Keybase servers rather than by other end users. The chain can only be extended by the user’s clients. The chain is never shortened and clients are removed by recording the removal in the chain rather than shortening or changing the chain.

Since the chain is distributed by the Keybase servers, the approach is a TTP approach with optional OOB verification and added links to external (non-cryptographic) identities.

### Multi-client

Each client has its own cryptographic identity. A new client is added to the chain by an existing client upon creation. Similarly, client removals are recorded on the chain by the client performing the removal.

## Threema

Threema follows a TTP approach with optional OOB verification (see page 6 on [Threema’s cryptography whitepaper](https://threema.ch/press-files/2_documentation/cryptography_whitepaper.pdf)). Threema is special in that it very prominently displays the degree of authentication for a given contact. If the cryptographic identity of a given contact was simply fetched from the Threema server, it shows a single red point. If there is a phone number or email address associated with the contact that is already in the user’s address book, it shows two orange points (this is essentially still relying entirely on the TTP). Finally, if the user has performed an OOB verification, it shows three green points.

### Multi-client

The multi-client approach taken by Threema is not reflected in the user’s cryptographic identity (see page 20 and following in the whitepaper linked above). There is a primary (mobile) client, with the option to open a secondary client in a browser. That client does not have its own cryptographic identity but instead relays messages through the primary client.

The secondary clients can be managed by the primary client.

## PGP

PGP follows the web of trust approach with optional OOB verification. As described above, users can use their cryptographic identities to sign those of other users. This usually happens after the user has verified OOB that a given identity indeed belongs to the user in question. A user who has no way to verify an identity OOB can then check the existing signatures on the identity and decide, depending on how much the user trusts the judgment of the signers if the it can trust the identity in question.

### Multi-client

PGP doesn’t have a notion of clients specifically and instead requires the user to manage its own key material. It is thus possible to either use different identities on different clients or just use the same on all clients. The user is also free to choose if and how to cross-sign the identities or to use a hierarchy of identities.

## Element

Element clients obtain the identity of their peers from their respective home servers, which act as trusted third parties. In addition, Element allows the verification of users OOB either in an interactive process, where users compare a sequence of Emoji in real-time or by comparing the full “session key” (which is presumably the client’s Curve25519 identity key). The former approach has to happen in real-time, which allows Element to only display a short sequence of emoji, while the latter can happen asynchronously.

### Multi-client

Element clients have distinct identity keys. When a new client is created, it [can be cross-signed](https://element.io/enterprise/device-verification) by an existing client. Thus, when performing the OOB verification process described above, all cross-signed clients are verified at once. Interestingly, after an OOB verification took place, new clients have to be verified even after they were cross-signed by an existing client. See also the Matrix specification for [cross-signing](https://spec.matrix.org/v1.3/client-server-api/#cross-signing).

Peers keep track of a user’s clients (*devices* in the Matrix specification) by periodically checking the user’s list of devices. If a user changes its list of devices, e.g. by removing a device, its peers will learn of this when they next synchronize with their respective homeservers.

## iMessage

iMessage [purely relies on the TTP approach](https://support.apple.com/guide/security/how-imessage-sends-and-receives-messages-sec70e68c949/web), where Apple distributes the cryptographic identities of individual parties to conversation partners. There is no way for parties to verify their peers’ public keys OOB.

### Multi-client

iMessage clients have their own cryptographic identity and Apple automatically distributes keys of new clients, as well as changes to a user’s client list to a user’s contacts.

# Comparison of messaging applications

Due to the diversity of approaches to authentication in general and multi-device in particular, it is hard to compare the individual applications directly. However, we can categorize them broadly. Note that none of the applications below support transparency via a verifiable datastructure or verification via a verification question. The composed identity colum determines if the application exposes a fingerprint of the [composed user identity](authentication_systems.md#multi-client-or-composed-user-identities) (as opposed to the identities of the individual clients).

| Application | TTP | OOB Auth | Cross-signing | Transparency | Composed Identity |
| ----------- | --- | -------- | ------------- | ------------ | ----------------- |
| Signal      | ✅   | ✅        | ❌[^1]         | ❌            | ❌[^1]             |
| WhatsApp    | ✅   | ✅        | ✅             | ❌            | ✅                 |
| Keybase     | ✅   | ❌        | ✅             | ✅[^4]        | ✅                 |
| Threema     | ✅   | ✅        | ❌[^2]         | ❌            | ❌                 |
| PGP         | ❌   | ✅        | ✅[^3]         | ❌            | ❌                 |
| Element     | ✅   | ✅        | ✅             | ❌            | ❌                 |
| iMessage    | ✅   | ❌        | ❌             | ❌            | ❌                 |


[^1]: Signal uses the same cryptographic identity for all of a user's clients.
[^2]: Threema only supports multi-client by routing all messages through a main client. Secondary clients thus don't have their own cryptographic identity.
[^3]: Only if individual clients have their own key material.
[^4]: Keybase's signature chains are no [verifiable datastructure](https://transparency.dev/verifiable-data-structures/), but they do allow users to verify the consistency of another user's identity.
