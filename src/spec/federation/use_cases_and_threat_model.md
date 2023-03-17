# Federation use cases

A federated messaging architecture is different from a centralized one in a number of ways. Here, we list a number of use-cases where a federated architecture is advantageous.

The most obvious one is an organisation that wants more control over the operational aspects of secure messaging. An organisation that runs its own homeserver controls where the homeserver is hosted (e.g. in a jurisdiction or a geopolitical context of its choice), who can and can’t register or indeed use the service (e.g. only members of the organisation) or what other homeservers it wants to federate with (ruling out e.g. known sources of spam and abusive content).

In the context of consumer messaging, users have the option to choose which messaging provider in a given federation they sign up with and they have the option to switch the provider without (necessarily) losing access to the overall messaging network. This is especially important as large messaging providers tend to be driven by motivations other than the freedom of their users. Users also have the option of running their own homeserver, gaining the advantages listed above.

Perhaps most importantly, a federated architecture is more resilient in a number of ways. If a net-split happens (e.g. due to an internet shutdown by an authoritative government), local providers can continue to operate autonomously and homeserver instances on each side of the net-split can continue to federate. Similarly, a federated messaging system is harder to censor than a single messaging provider. On the one hand that is simply because there are more instances to block. On the other hand, new instances can be spun up if required and users can move to unblocked instances if they are affected by the block.

# Federation functionality

The federation part of the Phoenix homeserver protocol can be split into four parts: Homeserver discovery, authentication key distribution, user discovery and connection establishment, and message distribution.

### Homeserver domain and discovery

Each homeserver has a homeserver domain associated with it. The domains determines how the individual components (AS, DS, QS) of the homeserver are discovered by other homeservers and it is also the domain that appears in the user IDs of the homeservers users in a federated context.

For the PoC-Phase, discovery is simple, as each homeserver is expected to be reachable at a fixed subdomain under the its homeserver domain: If the homeserver domain is `example.com`, the homeserver’s AS is reachable under `as.example.com`, the DS under `ds.example.com`, etc. For authorization reasons and to allow more flexible deployments, this will later be changed to involve a lookup at the `.well-known` of the homeserver domain.

### Authentication key distribution

At their respective domains, the each part of the homeserver provides an endpoint for federated homeservers (and clients) to retrieve their key material. The QS and DS provide endpoints to retrieve their respective verification keys and the AS provides an endpoint to retrieve the AS root and intermediate credentials, as well as associated revocation information. This key material can then be used to authenticate incoming messages and KeyPackageBatches (QS), remove proposals in the context of MLS groups (DS) or client credentials (AS). All queries to key discovery endpoints must be sent through TLS-secured channels to ensure that homeserver authentication is tied to the local web root of trust of the caller.

### User discovery and connection management

Homeservers must allow federated homeservers to discover local users to facilitate connection establishment across homeservers. This involves endpoints on the AS to retrieve connection establishment KeyPackages and one to deliver `ConnectionEstablishmentPackages` for a given user.

### Message distribution

Both QS and DS provide endpoints required for federated message distribution. The DS makes all of its endpoints publicly available safe for the `CreateGroup` and the `RequestGroupId` endpoints, as we only allow local clients to create groups on a given DS. The QS on the other hand only provides the `FederatedEnqueueMessage` endpoint for federated homeservers.

# Federation threat model

In this section we define the threat model for homeserver federation on the delivery protocol layer. This means in particular that we don’t consider threats such as fraud and abuse that require mitigations on higher layers.

For the purpose of this threat model, we consider a homeserver that allows registration of anonymous users, which is in contrast to, say, an organisation that only allows user registration by its members. We also don’t consider a restricted federation, where federation is only enabled between trusted homeservers. Note that this lack of trust in both users and federated homeservers only strengthens our threat model.

The lack of trust in local users means that we don’t consider fraud and abuse by remote users as part of our federated threat model, since the countermeasures and mitigation techniques are largely the same as for fraud and abuse between local users. The exception being that a high level of reported incidents originating from users of a specific remote backend may lead to a complete blockage of that backend, which our federation protocol does enable.

## Actors

In our threat model we distinguish between the following actors:

- Users: Individuals that have registered with the local homeserver and that operate one or more clients. Local users are not trusted and can behave either honestly or represent an attacker.
- Operator: The operator of the local homeserver, which is trusted in the context of this threat model.
- Remote Users: Users of a remote homeserver that the local homeserver federates with. Remote users may act as attackers even if their local homeserver operator is honest.
- Remote Operators: Operators of remote homeservers that the local homeserver federates with. As noted above, remote homeserver operators may be honest, but are not trusted in the context of this threat model. A remote homeserver operator may either be malicious and register a large number of users itself, or negligent such that a lack of user registration leads to another actor registering a large number of malicious users. Either way, this threat model assumes that there is no restriction on the registration of malicious users on a remote homeserver.

## Assets

This threat model considers the following assets of (local) users and the (local) homeserver operator:

- User assets
    - Attention: Any unwanted message that gathers the user’s attention (e.g. a push notification or just a message in the user’s inbox) can be an attack. Since basic homeserver functionality requires that it facilitates connection establishments between (otherwise unconnected) users, attacks on this asset exist on a spectrum. The homeserver won’t be able to prevent individual (unwanted) connections from reaching the user, but it may be able to limit the amount of such attacks.
    - Client resources: Since we consider the local resources of user clients to be limited, for example, because the client has limited computational power, or because it runs on battery, any action that causes an undue amount of resource consumption on a user’s device is considered an attack. As with attention, this threat exists on a curve and complete mitigation will not be possible.
    - Availability: Essentially an extreme instance of an attack on a client’s resources, we consider actions affecting the availability of a client as an attack.
- Operator
    - Availability: Actions by other actors that cause the disruption of the services provided by the local backend are considered attacks, both with regard to services provided to local users and services that enable federation.
    - Hosting resources/costs: This threat models considers an attack any action that leads to an undue consumption of resources (compute, bandwidth, storage) of the local homeserver. Even if availability isn’t affected, depending on the severity of the attack, excessive resource consumption can be expensive for the local operator.
    - Indirect attacks: If an actor manages to use the local homeserver to attack other targets (that don’t have to be part of the messaging infrastructure), this is considered an attack. For example, a remote user or homeserver operator could try to use the homeserver’s federation service in a DoS reflection attack. Or a local user could try to mount an attack on a remote homeserver through its own local homeserver.

## Threats

Threats in the context of the Phoenix homeserver federation protocol fall into the categories Denial-of-Service (DoS) and Spam. Both categories also arise in the non-federated architecture, but become more nuanced in the federated case due to the increased number of potential parties involved. Other attacks such as fraud and abuse are of course also relevant in the federated context. However, since they are largely defined through the content of messages (as opposed to the transmission of a message), their prevention, detection and reaction has to take place on another layer of the messaging stack (although blocklisting a federated homeserver might be part of a mitigation technique).

### DoS Attacks

DoS attacks in the federated context arise from the possibility of one actor flooding another actor with a large amount of messages. For example, a remote user could send a large amount of malformed messages to another user. The messages might not be enough to sufficiently stress the attacker’s or the victim’s homeservers, but since clients are typically have much fewer resources (e.g. in terms of compute), the victim’s client might be overwhelmed by the large amount of incoming messages. Even if victim’s client recognizes that the messages are malformed and won’t show them to the user, the validation process might tax the client’s resources to the point where it affects the client’s battery life, or even it’s availability to the user.

Another example for a DoS attack is one where a malicious user causes a remote homeserver to take anti-spam or anti-DoS mitigation measures. For example, because the user has managed to send a large amount of messages to the remote homeserver via the attacker’s local homeserver, thus forcing the remote homeserver to block the user’s homeserver from relaying further messages. The attacker thus prevents all other local user from interacting with users on the remote homeserver.

### Spam

While individual messages might be considered “Spam”, in the context of the federation protocol, spam is considered a number of messages that might not be enough to be considered a DoS attack, but that (in one way or another) captures the user’s attention. For example, a small number of messages that does not significantly affect the client’s resources, but that ends up on the user’s screen, distracting them from non-spam messages. Or a small number of messages that might not be well-formed enough to pass through the validation checks of the victim’s client, but that make the victim’s homeserver send a push notification that leads to the device actually notifying the user, thus unnecessarily catching the victim’s attention.

# Mitigation techniques

The following presents on overview over the techniques used by the Phoenix homeserver protocol to mitigate DoS and Spam attacks, beginning with the mitigation measures against DoS and Spam attacks in the context of interaction with local clients, followed by those for the federated context.

## Privacy Pass tokens

[Privacy pass](https://datatracker.ietf.org/doc/draft-ietf-privacypass-protocol/) (PP) is a protocol currently under standardization by the Privacy Pass working group at the IETF. Generally speaking, it allows one entity (the token issuer) to issue tokens upon request. Another entity can then require the redemption of such a token, e.g. when it serves a request. The Privacy Pass protocol is special in that the redeemed token can be validated, but not otherwise linked to the entity originally requesting the token.

### Using PP tokens for DoS and Spam mitigation

The Phoenix homeserver protocol uses PP for DoS and Spam mitigation while allowing clients to remain anonymous (at least to a certain degree) when interacting with endpoints that bear DoS or Spam potential. More concretely, clients can authenticate to the AS and request a number of tokens. The AS then keeps track of which client (or user) was issued how many tokens and can enforce a limit on the tokens issued per entity and/or in general. Any Homeserver endpoint that bears DoS or Spam potential can then require the calling client to present a token for each individual request. The number of tokens a client has thus limits the amount of requests it can make.

While token redemption has a certain computational overhead, it is comparable with the termination of a TLS connection and has the advantage of being scalable independently from other homeserver operations.

### Hoarding Attacks

The token scheme described above has the inherent risk of a hoarding attack, where one or more users frequently request tokens up to their allowed maximum and at some point spend them all at once to maximize the number of queries in a small amount of time. This risk can be mitigated at least partially by rotating the key material used in the token issuance and validation process. After keys have been rotated, all previously issued tokens become invalid and the attacker(s) have to start hoarding anew. Another mitigation measure for hoarding attack is traditional, IP-based rate-limiting, thus forcing the hoarding attacker to split its attack among sufficiently many IP addresses (or address spaces) to evade the rate-limiting measures.

### Mitigating Sybil-Attacks

Token based spam and DoS attack mitigation is largely reliant on the fact that it is infeasible for the attacker to register a large number of users, which in turn would allow it to request (and then use) a large number of tokens. A variety of measures can be taken to limit user registration such as Proof-of-Work (in the assumption that an attacker has limited computational resources) or Proof-of-Personhood, such as Captchas, or (to some degree) [hardware attestation](https://blog.cloudflare.com/introducing-zero-knowledge-proofs-for-private-web-attestation-with-cross-multi-vendor-hardware/).

## Token-based mitigation in the federated case

The Phoenix homeserver protocol uses token-based rate-limiting to prevent spam and DoS attacks in the federated case in the same way as the local case, where clients can request tokens from a federated homeserver, which they can then redeem when interacting with the remote DS. The only difference is in the way that remote clients authenticated with the local AS when they request tokens and in the way that the local AS rate-limits token issuance to individual clients.

### Authentication of remote clients

The protocol itself is the same for remote and for local clients in that both authenticate using their client credentials. The AS then verifies the signature on their client credentials. For local clients, the AS can use its own intermediate credentials to validate the signature. For remote clients, the local AS might lack the key material to authenticate the client credential. In that case, the local AS first fetches the key material (root credential, intermediate credentials, as well as revocation information) from the corresponding endpoint of the remote AS and then verifies the signature on the remote client’s credential.

### Rate-limiting of remote clients

Both for local and remote clients, the local AS keeps a record of how many tokens an individual client has been issued. For local clients, that is enough, since the local AS can control the rate with which new clients register and prevent Sybil attacks as described above. However, with the AS has no control over registration of remote clients and in case of a negligent or even malicious remote AS, a large amount of remote clients could each request tokens up to their limit to then flood the local DS with requests to either perform a DoS or a spam attack. The local AS thus has to rate-limit not just per-client, but also per remote homeserver (domain). An attacker could then only register a limited amount of clients per domain that they control. Naturally, this assumes that any adversary only controls a limited amount of domains. To mitigate the scenario where an adversary controls a large number of domains, the local AS could keep the limit for new domains low and only increase it slowly to at least prevent a sudden influx of remote clients.

## (Group-)Pseudonym-based rate-limiting

In addition to client-based and domain-based token rate-limiting the DS that redeems the tokens can additionally impose per-group rate-limiting measures based on individual group pseudonyms, thus limiting the amount of impact an attacker can have by sending messages to a single group.