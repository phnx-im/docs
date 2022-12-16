# Concepts and security properties

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
