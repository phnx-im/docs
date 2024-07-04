# Credentials

The MLS protocol underlying the Phoenix homeserver protocol uses different types of credentials to identify different actors. It also uses credential chaining to make credential management easier.

## Credential Fingerprints

Credentials can be referenced via their credential fingerprints. A credential fingerprint is simply the hash of the credential, where the hash is determined by the CredentialCiphersuite.


```rust
struct CredentialCiphersuite {
	hash: Hash,
	signature_scheme: SignatureScheme,
}
```

## Credential Expiration

All credentials expire after a certain amount of time. The validity period of a credential is indicated by its ExpirationData. Credentials may only sign credentials where the validity period of the signed credential is within the validity period of the signing credential.


```rust
struct ExpirationData {
		not_before: u64,
		not_after: u64,
}
```

## Credential Signing and Self-Signatures

All credentials must include a signature over themselves to prove possession of the signature key. The credential structs thus have the following structure.

```rust
struct CredentialPayload {
		// ... credential data
}

struct CredentialSelfSignedPayload {
		payload: CredentialPayload,
		// SignWithLabel(., "CredentialPayload", CredentialPayload)
		self_signature: Signature,
}

struct Credential {
		signed_payload: CredentialSelfSignedPayload,
		// SignWithLabel(., "CredentialSelfSignedPayload", CredentialSelfSignedPayload)
		signature: Signature,
}
```

Note that the types of the individual credential types below must have their own distinct names so that the resulting signatures are specific to a credential type and cannot be used for a confusion attack.

In the following sections, we only describe the CredentialPayload struct for each credential.

## AS credentials

- Used to sign intermediate AS credentials.
- The AS keeps those offline or in an HSM.
- They should be valid for a long time, e.g. 10 years.
- Rotation MUST be possible despite the long validity period.
- The controller of domain should still be able to “rotate” by simply publishing a new cert via the *get AS credentials* endpoint.
- There is no higher-level credential, so this is self-signed only.

```rust
struct AsCredential {
		as_domain: Fqdn,
		expiration_data: ExpirationData,
		credential_ciphersuite: CredentialCiphersuite,
		public_key: SignaturePublicKey,
}
```

## AS intermediate credentials

- Used to sign client credentials.
- Should be published in the same way as the AS credential.
- Signed by an AS credential

```rust
struct IntermediateAsCredential {
		expiration_data: ExpirationData,
		credential_ciphersuite: CredentialCiphersuite,
		public_key: SignaturePublicKey, // PK used to sign client credentials
		signer_fingerprint: CredentialFingerprint, // fingerprint of the signing AsCredential
}
```


## Client Credentials

- Used by clients to sign IntermediateClientCredentials, as well as some other messages such as the [WelcomeAttributionInfo](../glossary.md#welcome-attribution-info) and the connection packages.
- Should be stored by clients in their device’s secure enclave if possible.

```rust
struct ClientCredential {
		client_id: ClientId,
		expiration_data: ExpirationData,
		credential_ciphersuite: CredentialCiphersuite,
		public_key: PublicSignatureKey,
		signer_fingerprint: CredentialFingerprint,
}
```

### Client credential signing requests

When a client requests that its credential be signed by an AS, it sends the CredentialSelfSignedPayload as defined [here](./credentials.md#credential-signing-and-self-signatures).

## Leaf credentials

- per-group credential
- to identify a client, the identifying client should follow the chain to the client cert.
- To be used in leaves in an MLS group.

```rust
struct LeafCredentialPayload {
		expiration_data: ExpirationData,
		credential_ciphersuite: CredentialCiphersuite,
		public_key: PublicSignatureKey,
		signer_fingerprint: CredentialFingerprint,
}
```
