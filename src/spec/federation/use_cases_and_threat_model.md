# Federation use cases

A federated messaging architecture is different from a centralized one in a number of ways. Here, we list a number of use-cases where a federated architecture is advantageous.

The most obvious one is an organisation that wants more control over the operational aspects of secure messaging. An organisation that runs its own homeserver controls where the homeserver is hosted (e.g. in a jurisdiction or a geopolitical context of its choice), who can and can’t register or indeed use the service (e.g. only members of the organisation) or what other homeservers it wants to federate with (ruling out e.g. known sources of spam and abusive content).

In the context of consumer messaging, users have the option to choose which messaging provider in a given federation they sign up with and they have the option to switch the provider without (necessarily) losing access to the overall messaging network. This is especially important as large messaging providers tend to be driven by motivations other than the freedom of their users. Users also have the option of running their own homeserver, gaining the advantages listed above.

Perhaps most importantly, a federated architecture is more resilient in a number of ways. If a net-split happens (e.g. due to an internet shutdown by an authoritative government), local providers can continue to operate autonomously and homeserver instances on each side of the net-split can continue to federate. Similarly, a federated messaging system is harder to censor than a single messaging provider. On the one hand that is simply because there are more instances to block. On the other hand, new instances can be spun up if required and users can move to unblocked instances if they are affected by the block.

## Federation functionality

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
