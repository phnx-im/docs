# Threat model

## Overview

The main concern of the Phoenix homeserver protocol is to minimize client metadata visible to the server. Two kinds of adversary are generally considered in this threat model: The snapshot adversary and the active observer adversary.

While both adversaries can be generally assumed to control the network, this threat model does not consider analysis of traffic patterns or other use of network metadata. While the use of network metadata can yield powerful attacks, common countermeasures such as onion routing or the use of mixnets are orthogonal to the Phoenix homeserver protocol and can be used in conjunction with it to mitigate such attacks.

### Adversary types

As suggested by the name, the snapshot adversary has access to snapshots of the server's persisted state. It can view an arbitrary number of individual snapshots, but does not have access to any part of the server's volatile memory.

The active observer adversary is strictly more powerful than the snapshot adversary. It can inspect both persisted server state, as well as any part of the server's volatile memory.

## STRIDE based threat model

This chapter contains a threat model for an MLS homeserver. For now, this threat model is based solely on its functional requirements. As we build the specification of the homeserver, we will update this document to consider any additional assets.

## Functional requirements

The homeserver is expected to fulfill the functional requirements described [here](./functional_requirements.md) and to interface with parties fulfilling the roles outlined in that document.

## MLS as underlying protocol

The homeserver facilitates communication between clients via the Messaging Layer Security (MLS) protocol, which already provides a number of [security guarantees](https://www.ietf.org/id/draft-ietf-mls-architecture-08.html#name-intended-security-guarantee) which we do not describe here.
