# Authentication in messaging applications

All of the commonly used messaging applications aim to provide authentication as one of their primary security guarantees. The goal is that recipients of a given message can authenticate the sender based on (public) cryptographic key material which we call the sender’s *cryptographic identity* (for brevity sometimes just *identity*). The cryptographic identity is typically a signature public key or a public key that can be used in an authenticated key exchange. The holder of the corresponding private key material is the owner of the cryptographic identity.

The recipient can use the cryptographic identity of the sender to either authenticate individual messages (e.g. verifying the sender’s signature on an incoming message using the sender’s public signature key) or to establish an authenticated channel to the sender by way of an initial, authenticated key agreement (e.g. a Diffie-Hellman (DH) style key exchange involving the DH public key of both parties). In both cases, the recipient relies on the fact that the cryptographic identity it uses on for authentication is indeed owned by whom the recipient thinks the sender of the message is.

This leads to the question of how the sender and receiver can ensure (or at least increase their confidence) that the cryptographic identity each has of its peer is correct.

The problem is typically solved by the sender and receiver using a trusted channel to either exchange cryptographic identities or verify them later.

## Meddler-in-the-middle (MITM) attacks

In both cases, the threat is that an adversary, often called a meddler-in-the-middle [sits between sender and receiver](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) and controls the channel they use to exchange messages. When the sender and receiver exchange their respective cryptographic identities, the adversary replaces them with ones that it controls. After such an attack, the adversary can impersonate one victim towards the other, as long as sender and receiver use a channel for communication that the adversary controls.

# Existing authentication concepts

There are many methods for the the recipient to ensure (or gain confidence in the fact) that the cryptographic identity it uses to authenticate the sender of a message corresponds to the private key material held by the original sender.

Some of these methods rely on the properties of the channel used to **exchange** the identities and some on additional ways of **verifying** identities after they were exchanged through a channel with limited trust.

The approaches are not mutually exclusive. Typically messaging systems use at least two of the approaches detailed below.

## Trust on first use (TOFU)

As the name implies, [trust on first use](https://en.wikipedia.org/wiki/Trust_on_first_use) means that the receiver trusts the cryptographic identity of the sender upon initial reception, regardless of the transportation channel.

**Threat model:** The adversary either controls the channel, but is passive (can observe, but not tamper with traffic) at the time the exchange happens, or controls the general communication channel, but not the one used to exchange identities.

## Trusted third party (exchange)

A typical scenario in the messaging world is where the receiver trusts the cryptographic identity of the sender because it received the identity from a [trusted third party](https://en.wikipedia.org/wiki/Trusted_third_party) (TTP). This approach only works if there is a designated trusted third party that all participants of the messaging system are either provisioned with in some way or have an authenticated way of retrieving. Typically, TTPs are no regular participants in the messaging system.

- The receiver could obtain the sender’s identity from the provider of the messaging service, as long as the cryptographic identity of the messaging service itself was provisioned upon downloading and installing the app.
- The receiver could obtain the sender’s identity from the sender’s identity provider, as long as it has some way of discovering the address and cryptographic identity of said provider through an authenticated channel.

**Threat model:** The adversary actively controls the network, but not the trusted third party. The adversary also can’t interfere with the provisioning process.

## Out-of-band verification

To establish trust in the validity of the cryptographic identities of the sender, the recipient party can verify via an [out-of-band (OOB) channel](https://ssd.eff.org/en/glossary/out-band-verification).

- Sender and receiver could meet in the real world and verify their respective cryptographic identities by comparing public keys on their screens, or by scanning a QR code. Note, that this scenario requires both parties to trust their hard- and software (e.g. display drivers) to show the right public key or QR code.

**Threat model:** The adversary controls the network and in particular the channel (or the third party) used to exchange the identities, but does not control the channel used to verify them.

## Delegated verification

Instead of performing the verification itself, the receiver can also decide to trust another party to verify the sender’s identity for them. There are several variants of this scenario, although they are all based on the principle of delegated verification or delegated trust.

### Trusted third party (verification)

Similar to the use of TTPs as an authenticated channel to exchange identities, a TTP can also be used to verify a cryptographic identity. As with the use of a TTP for the exchange of identities, the cryptographic identity of the TTP has to be known to all users for this approach to work.

- Many people use their homepages or social media profiles to publish a fingerprint of their cryptographic identity in one or more messaging apps. As long as the receiver can establish an authenticated channel to the TTP hosting the fingerprint, it can verify the identity.
- In the context of the WebPKI, when connecting to a web server via HTTPS, the server sends its certificate and provides a certificate chain from the server certificate to a WebPKI root of trust. The client can now verify the chain up to the root of trust. The certificate authorities that sign server certificates in the root of trust thus act as TTPs for verification.

**Threat model:** The adversary controls the network and in particular the channel (or the third party) used to exchange the identities, but not the third party that facilitates the verification.

### Cross-signing

Cross-signing is similar to the use of a TTP for verification, except that the parties that cross-sign a cryptographic identity are regular participants of the messaging system. Cross-signing is often used in addition to other authentication mechanisms.

- Cross-signing is commonly used if a party has multiple clients, each with its own cryptographic identity. In this case, the cryptographic identity of a new client can be cross-signed by an existing one, allowing other parties that have already verified the identity of the cross-signer to also trust in the new client.

**Threat model:** The adversary controls the network, but does not control the party cross-signing the cryptographic identity in question. The adversary also cannot compromise parties that are trusted to cross-sign.

### Web of trust

The [web of trust](https://en.wikipedia.org/wiki/Web_of_trust) is a special case of cross-signing, where any party can cross-sign the cryptographic identity of any other party and publish its cross-signature in a publicly accessible directory. A party judging the trustworthiness of a cryptographic identity can base its decision on the quantity and quality of cross-signatures for a given identity, where the quality of the signature is equivalent to the amount of trust placed in the cross-signer.

While other verification approaches have a clear condition under which the receiver will trust the veracity of a sender’s cryptographic identity, this is not necessarily the case with the web of trust. Here, the receiver has to decide to which degree it trusts a certain cryptographic identity depending on who signed a sender’s identity and how much the receiver trust the signers.

- The web of trust is used in PGP, where parties can upload their signature over other cryptographic identities to a well-known set of public key servers. The receiver can download the sender’s public key and check who has cross-signed it. If the receiver concludes that enough people have signed whom it trusts (and the public keys of whom it has already verified), the receiver can use the sender’s public key for authentication.

**Threat model:** The verifier has several parties that it trusts to verify other cryptographic identities for them and the identity of which the verifier has indeed verified. The adversary cannot compromise enough trusted parties to pass the threshold required for the verifier to trust that a cryptographic identity is valid.

## Verifiable data structures

[Verifiable data structures](https://transparency.dev/verifiable-data-structures/) cannot prevent MITM attacks entirely, but make them harder for an adversary to perform without detection.

A TTP (or any other party that either publishes cryptographic identities or provides delegated verification) can record its actions (either publication or verification of a cryptographic identity) in a verifiable data structure. The data structure allows the TTP to prove to other parties that its records are consistent (i.e. that past records were not altered or deleted) and that individual entries are indeed part of the data structure (proof of inclusion of a given record). The TTP can then regularly publish a *view* of its current records in which an inclusion proof holds. Consistency is then guaranteed by providing proofs that a view is the successor of previous view.

Parties can now check that their own identity is correct in the view that the TTP presents to them.

Also two communicating parties can include the view they received from the TTP in their messages. If two views differ (and one is not a successor of the other), it is a sign that the TTP has tampered with its records and shown different records to the two parties.

- Certificate authorities (CAs) in the WebPKI can (and are sometimes required to) use Certificate Transparency to publish the server certificates they sign. Clients can demand from servers to prove that its certificate has been published in the Certificate Transparency log. If a (CA-signed) certificate is used to conduct a MITM attack, there is proof that the CA has either accidentally or intentionally helped facilitate the attack.

**Threat model:** The adversary can compromise the TTP and actively impersonate a party towards a victim by claiming that one of their own identities is the genuine identity of the victim. However, it will have to show different views to the impersonated party and the victim. If parties exchange their views of the TTP with one-another, the adversary has to decide which party to show which view to avoid detection. If two parties exchange differing views, the adversary is caught.

## Verification question

Similar to an OOB channel, the sender and receiver can use shared information to help gain trust in one another’s cryptographic identity. [An adaptation of the socialist millionaire problem](https://dl.acm.org/doi/abs/10.1145/1314333.1314340) allows one party to ask a question and provide the expected answer. The other party learns the question and provides its answer. The protocol then allows the parties to learn both if their replies and their shared cryptographic identities match without leaking their specific answers.

- The Off-the-Record (OTR) protocol allows for authentication via a verification question as described above to verify cryptographic identities. The initiating user is asked to provide a question and corresponding answer. The responder is then shown the question and also prompted for an answer. If the answers match, the identities are considered verified.

**Threat model:** The adversary can control the network completely, as long as it doesn’t know the answer to the question.

# Approaches used in messaging applications

Different messaging applications make use of one or more of the authentication approaches, sometimes with small variations to the general concepts described above.

### Multi-client or composed user identities

Messaging applications often allow the use of multiple clients (e.g. on different devices) by a single user. However, since users are typically interested in authenticating users rather than individual clients, applications have to deal with the challenge of presenting a single user identity despite messages being sent potentially from more than one client.

Multi-client authentication is not covered in the discussion of existing approaches above, because the individual messaging applications have found unique ways of dealing with this problem.

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

Due to the diversity of approaches to authentication in general and multi-device in particular, it is hard to compare the individual applications directly. However, we can categorize them broadly. Note that none of the applications below support transparency via a verifiable datastructure or verification via a verification question.

| Application | TTP | OOB Auth | Cross-signing |
| ----------- | --- | -------- | ------------- |
| Signal      | ✅   | ✅        | ❌[^1]         |
| WhatsApp    | ✅   | ✅        | ✅             |
| Keybase     | ✅   | ❌        | ✅             |
| Threema     | ✅   | ✅        | ❌[^2]         |
| PGP         | ❌   | ✅        | ✅[^3]         |
| Element     | ✅   | ✅        | ✅             |
| iMessage    | ✅   | ❌        | ❌             |


[^1]: Signal uses the same cryptographic identity for all of a user's clients.
[^2]: Threema only supports multi-client by routing all messages through a main client. Secondary clients thus don't have their own cryptographic identity.
[^3]: Only if individual clients have their own key material.
