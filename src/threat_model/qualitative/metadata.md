# Metadata

Protection of metadata is a key goal of the Phoenix homeserver protocol. Metadata typically includes information like the sender and recipient of a message, the time it was sent, and the size of the message, and the group conversation in which it was sent. Metadata aggregation can reveal patterns of communication and can be used to infer social relationships. In particular, correlation can be used to infer social graphs.

The metadata threat model considers two types of adversaries: snapshot adversaries and active observer adversaries.

## Snapshot adversarires

Snapshot adversaries have access to snapshots of the server's persisted state. They can see the metadata of all messages that have been sent and received up to the point of the snapshot and that haven't been deleted from the long-term storage. An example of a snapshot adversary is a government that has seized the server's hard drives, or a provider that is forced to comply with a government request to hand over data.

The main protection against the snapshot adversary is encryption at rest for most relevant metadata. In this instance, encryption at rest means that clients hold the key material and supply it to the server for individual queries. The server can decrypt the data and process the query before re-encrypting the data and deleting the key from memory.

As a consequence, snapshots become largely useless to the adversary since the senders of messages, the group in which messages were sent, etc. are encrypted. The encryption keys are either only held by clients, or are only held temporarily by the server in volatile memory and are therefore not part of the snapshot.

### Data available to snapshot adversaries

Despite the use of encryption at rest, the snapshot adversary can still see the following data:

- For each group on the DS
  - The rough size of the group
  - The timestamp of the last time the group was active (e.g. due to a key update)
- Reserved group IDs
- AS key material
  - Credentials of all clients (includes usernames)
  - AS credentials and key material
  - OPAQUE key material
  - Privacy Pass key material
- The number of messages in the connection queues of all clients (but not the contents of the messages)
- All user pseudonyms and their associated client pseudonyms
- All currently available KeyPackages of all clients
- The number of messages in all client queues and the (padded) length of each message (but not the contents)
- QS key material
  - QS signing keys
  - QS HPKE encryption keys

## Active observer adversaries

Active observer adversaries have full access to the server including its volatile memory in addition to the snapshots.

The main protection against the active observer is that users interact with the services using per-group pseudonyms instead of their real identities. This means that the adversary cannot link any individual action of a user with the user's identity as seen by other users. The adversary can still see the metadata of messages, but cannot link them to the real identities of the users.

### Data available to active observer adversaries

As per definition, active observer adversaries have access to all data on the server at the time of their activity.

## Traffic analysis

Traffic analysis, i.e. the analysis of data traffic patterns, is not considered part of the threat model. While the use of network metadata can allow an active observer (be it on the server or just the network) to infer social graphs and other sensitive information, the Phoenix homeserver protocol does not aim to protect against such attacks. Instead, common countermeasures such as onion routing or the use of mixnets can be used in conjunction with the Phoenix homeserver protocol to mitigate such attacks. However, to the extent possible, the protocol avoids behaviours obvious traffic patterns such as batched queries, messages at regular intervals, etc.

## Privacy Pass

The [Privacy Pass batched tokens issuance protocol](https://datatracker.ietf.org/doc/draft-ietf-privacypass-batched-tokens/) is used to rate-limit individual users without violating their metadata privacy.

The protocol flow is as follows:

- A registered client requests tokens from the AS. To amortize the cost of the issuance, the client can request several tokens at once.
- The AS verifies the client is allowed to request the requested number of tokens and issues the requested tokens. In the positive case, the AS issues the tokens. The Privacy Pass protocol ensures that the tokens are not linked to the client's identity when redeemed later.
- Whenever the client connects to an endpoint rate-limited using Privacy Pass, it sends a token along with the request.
- The endpoint verifies that the token has previously been issued by the AS and has not yet been spent. If both criteria are met, the endpoint marks the token as spent and continues with processing the request.

In the issuance and redemption phase of tokens the AS uses a pair of public/private keys. As long as the AS uses the same keypair for all clients, it cannot correlate the tokens with the clients' identities. However, a malicious AS could use distinct keypairs for each client and thus correlate the tokens with the clients' identities. See [Section 6.2 of RFC 9576](https://www.rfc-editor.org/rfc/rfc9576.html#section-6.2) for more information on this threat.

The Phoenix homeserver protocol currently does not mitigate this attack. In the future, AS public/private keys will either be gossiped between clients as part of their group state, or a key transparency-like mechanism will be used to allow clients to verifiably track Privacy Pass key material used by the AS.
For now, clients can however still download the public key anonymously at any given point in time, making it harder for the server to issue a specific key per client.
