# Cryptographic primitives and schemes

To fulfill the functional and performance requirements, as well as those for security and privacy, the homeserver will need to employ a number of cryptographic primitives, e.g. signature, authenticated encryption, etc. This chapter discusses the primitives required, as well as the choice of schemes that are used for the initial specification. As customary in other protocol specifications the set of schemes used by a homeserver is called the homeserver's ciphersuite.

## Federation and cryptographic agility

Each homeserver operator will have its own requirements with regard to the ciphersuite that they want to use. This could be, for example, due to the requirement for the primitives to be certified in some way, or to meet a specific level of security for organizational policy reasons.

The underlying Messaging Layer Security (MLS) supports agility for a number of reasons and homeservers should, too.

For the initial homeserver specification, however, only one ciphersuite will be supported, even though the design will accommodate the requirement for homeservers to advertise their ciphersuite support.

## List of cryptographic primitives and schemes

For now, the homeserver is anticipated to require mostly basic symmetric and asymmetric primitives to fulfill its functional requirements. This is with the exception of an unlinkable token scheme to enable rate-limiting while allowing clients to access the homeservers services anonymously.

Along with the list of primitives, the list also includes a concrete scheme for each primitive that will be used in the initial specification.

In addition to the scheme's fitness to facilitate the listed requirements of the homeserver (functional, security, privacy, performance), an implementation of each scheme has to be available and fulfill the following criteria:

- Open source with an OSI-recognized license
- ideally implemented in Rust
- actively and well maintained

The RustCrypto suite of cryptographic implementations provides well-maintained Rust implementations of a large variety of cryptographic primitives and schemes. While they are generally not as highly optimized for performance as other available implementations, they provide a good baseline that is sufficient for a PoC homeserver implementation. In the future, we hope to upgrade any implementations we decide on for the PoC with similarly well performing ones that have formally verified properties such as functional correctness and secret-independent execution,

When it comes to the concrete choice of primitives and schemes, an easy choice for the initial specification is the suite of schemes in the “mandatory-to-implement” (MTI) ciphersuite of MLS, as it represents a compromise between security and good performance on a variety of devices. In particular, the primitives and schemes are

- Signatures: Ed25519
- Hash function: SHA256
- AEAD scheme: AES128-GCM
- KEM: DHKEMX25519

Additional recommendations for primitives that might be used in the homeserver specification are

- MAC: HMAC-SHA256
- KDF: HKDF-SHA256

While these are the schemes that should make up the first ciphersuite of the homeserver, the specification should remain modular, such that multiple schemes can be supported and different versions of the homeserver protocols can introduce or deprecate individual ciphersuites.

All schemes above have been well-tested in a variety of modern, cryptographic protocols (e.g. Signal, TLS). The only choice that might not be immediately obvious is AES128-GCM as AEAD scheme, since in many cases, Chacha20-Poly1905 is frequently used as a more efficient scheme on devices that don’t have hardware support for AES, most of those with ARM processors. However, with the most recent generations of ARM processors supporting AES instructions, AES128-GCM is a good choice for a first ciphersuite, given that support for multiple ciphersuites is planned for the future.

## Unlinkable tokens

Should unlinkable tokens be used to facilitate a metadata friendly rate-limiting mechanism, their performance is essentially to not ultimately represent a denial of service vulnerability themselves.

Since the Privacy Pass protocol is the only well-known unlinkable token scheme that has seen extensive real-world use and analysis, it is recommended as the protocol of choice for metadata-friendly rate-limiting. Besides its use in practice, it is also on the finishing stretch of the IETF standardization process. Privacy Pass defines a number of token types, each backed by another cryptographic scheme. Performance tests have revealed, however, that all schemes suggested in the draft specification are prohibitively expensive in terms of computational overhead in the various protocol operations. To mitigate this shortcoming, the authors of this document have proposed a new batched token type based on the [Ristretto255](https://datatracker.ietf.org/doc/draft-irtf-cfrg-ristretto255-decaf448/) group (also in the process of IETF standardization), which has significantly improved performance over the schemes used by existing token types. These improved results come at a slight security cost, which can be mitigated by a conservative key rotation policy.

The authors have already performed initial performance tests justifying the effort and published [a document](https://raphaelrobert.github.io/privacypass-batched-tokens/draft-robert-privacypass-batched-tokens.html) defining the new token type, as well detailing the security recommendations for consideration by the Privacy Pass working group at the IETF.