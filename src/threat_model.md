# Threat model

This section describes the different security properties the Phoenix homeserver protocol aims to provide and identifies and assesses threats to those properties.

The threat model is divided into two parts, a threat model based on a slightly modified version of the STRIDE methodology and a more qualitative description of the impacts of various compromise scenarios on the individual actors of the system.

## Security properties

The Phoenix homeserver system aims to provide confidentiality and authenticity for the communication between its users, privacy for user metadata and availability for the individual service components. The following briefly summarizes the security users can expect and explains on a high level why the individual security properties hold.

### Confidentiality and authenticity

Confidentiality and authenticity of user communication are the foremost priorities of the protocol. Specifically, users can expect that messages are readable only by their intended recipients and that recieved messages indeed originate from the apparent sender.

Both of these properties are already provided by the Messaging Layer Security (MLS) protocol underlying the Phoenix homeserver protocol. However, the authentication service (AS) is responsible for the issuance and distribution of authentication key material. Consequences of AS compromise on confidentiality and authenticity are detailed in the qualitative threat model.

### Metadata privacy

Secondary to confidentiality and authenticity is user metadata privacy. This means that the individual services (AS, queuing service (QS), delivery service (DS)) cannot link any individual action of a user with the user's identity (as seen by other users).

The metadata threat model considers two types of adversaries: a snapshot adversary, which has access to snapshots of the server's persisted state, and a more powerful active observer adversary, which has full access to the server including its volatile memory.

The main protection against the snapshot adversary is encryption at rest for almost all relevant metadata. Against the observer, users interact with the services using per-group pseudonyms instead of their real identities.

This threat model does not consider analysis of traffic patterns or other use of network metadata. While the use of network metadata can yield powerful attacks, common countermeasures such as onion routing or the use of mixnets are orthogonal to the Phoenix homeserver protocol and can be used in conjunction with it to mitigate such attacks.

### Availability

Availability of services is the final security property of the Phoenix homeserver protocol. While clients are expected to go offline periodically, an adversary should not be able to render a server unavailable to its users.

Besides industry standard mitigation approaches like IP-based rate-limiting, the Phoenix homeserver protocol uses Privacy Pass to rate-limit individual users without violating their metadata privacy.
