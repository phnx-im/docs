# Performance goals

The goal of the homeserver(s) is to facilitate messaging for consumers in general and in particular consumers with resource constrained devices such as smartphones running. Despite the advances in battery capacity, memory and computational performance in modern smartphones, consumers should still be able to communicate with the infrastructure without incurring disproportionate resource consumption. Additionally applications should be able to use the infrastructure without requiring exceptions to the restrictions that platforms running on modern smartphones (Android, iOS) impose on applications such as limited computation time when running in the background.

Similarly, homeserver operators should be able to run their servers without a significant overhead in costs due to computation, memory or persistent storage.

Quantifying performance goals is hard, since the hardware differs significantly across vendors. However, since cryptography is going to make up the largest consumer of computation time, as well as memory and persistent storage, we are going to use asymmetric cryptographic operations (public key encryption, signatures) as unit of measurement for computational overhead and number of keys that need to be stored as unit of measurement for memory and persistent storage overhead. This has the added advantage that we can later estimate the additional overhead in case of adoption of a Post-Quantum secure scheme with significantly larger key sizes.

## Client Queries

- Network overhead: Clients should be able to perform the most commonly used queries (i.e. send a message to a group or retrieve messages from their queue) with a single round-trip. In particular, it should be possible to request own messages in batch, although the service might limit the size each batch of messages retrieved at a time, leading to additional round trips.
- Computational overhead: The estimated computational cost for a given query heavily depends on the type of query. Let n be the number of member in a given group.
    - Simple (application) message to a group: One asymmetric operation for both sender and receiver
    - Message updating encryption key material: Less than 3log(n) asymmetric operations for the sender and less than log(n)
    - Message updating authentication key material of adding/removing a group member: Same as updating encryption key material plus <10 asymmetric operations per joining client
    - Joining a new group: between 5n and 10n asymmetric operations

Performance constraints are especially important if the respective operations are potentially performed in the background, i.e. when a user is not currently interacting with the client.

## Homeserver Processing

- Network overhead: Homeservers should typically respond to one message with one response, although fan-out of a message can result in additional messages sent over the network in a federated setting. Note, that disproportionate amplification of network traffic (i.e. one client request causing a number of server requests) will be mitigated by rate limiting measures such as traditional IP-based rate limiting and schemes such as Privacy Pass.
- Computational overhead: With the exception of the group joining cost, the computational cost of the server will be roughly equivalent to that of a client processing a given message.
- IO overhead: A query from a client should result in at most one read and one write operation from/to the homeserver’s database, although we expect caching to reduce the number of reads significantly.

## Additional Overhead due to Rate-Limiting

More sophisticated rate limiting measures, such as the Privacy Pass protocol might create additional overhead, such as an occasional additional round-trip and asymmetric operation to retrieve an access token from the homeserver. We estimate the required frequency of this kind of operation to be at most once per day.
