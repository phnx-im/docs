# Threat model

This section describes the different security properties the Phoenix homeserver protocol aims to provide and identifies and assesses threats to those properties.

The threat model is divided into two parts, a threat model based on a slightly modified version of the STRIDE methodology and a more qualitative description of the impacts of various compromise scenarios on the individual actors of the system.

## Security properties

The Phoenix homeserver system aims to provide confidentiality and authenticity for the communication between its users, privacy for user metadata and availability for the individual service components. The following briefly summarizes the security users can expect and explains on a high level why the individual security properties hold.

### Confidentiality and authenticity

Confidentiality and authenticity of user communication are the foremost priorities of the protocol. Specifically, users can expect that messages are readable only by their intended recipients and that recieved messages indeed originate from the apparent sender.

Both of these properties are already provided by the Messaging Layer Security (MLS) protocol underlying the Phoenix homeserver protocol. However, the authentication service (AS) is responsible for the issuance and distribution of authentication key material. Consequences of AS compromise on confidentiality and authenticity are detailed in the qualitative threat model.

### Metadata privacy

Confidentiality and authenticity cover the content of messages, but the metadata of messages can also reveal sensitive information. Metadata typically includes information like the sender and recipient of a message, the time it was sent, and the size of the message, and the group conversation in which it was sent. Metadata aggregation can reveal patterns of communication and can be used to infer social relationships. In particular, correlation can be used to infer social graphs. The main objective of the Phoenix homeserver protocol is to protect this metadata from adversaries by minimizing metadata and decorrelating the remaining data points as much as possible. See the [qualitative threat model](./threat_model/qualitative/metadata.md) for more details.

### Availability

Availability of services is the final security property of the Phoenix homeserver protocol. While clients are expected to go offline periodically, an adversary should not be able to render a server unavailable to its users.

Besides industry standard mitigation approaches like IP-based rate-limiting, the Phoenix homeserver protocol uses Privacy Pass to rate-limit individual users without violating their metadata privacy.
