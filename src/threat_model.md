# Threat model

This document contains a threat model for an MLS homeserver. For now, this threat model is based solely on its functional requirements. As we build the specification of the homeserver, we will update this document to consider any additional assets.

## Functional requirements

The homeserver is expected to fulfill the functional requirements described [here](./functional_requirements.md) and to interface with parties fulfilling the roles outlined in that document.

## MLS as underlying protocol

The homeserver facilitates communication between clients via the Messaging Layer Security (MLS) protocol, which already provides a number of [security guarantees](https://www.ietf.org/id/draft-ietf-mls-architecture-08.html#name-intended-security-guarantee) which we do not describe here.