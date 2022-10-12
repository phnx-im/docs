# Threat model methodology

We describe our threat model by roughly following the STRIDE framework as outlined [here](https://www.securesoftware.nl/resources/FrameworkSecureSoftware_v1.pdf). However, we modify the methodology slightly to fit use-case.

## Authentication

The "Authentication" property as listed in the STRIDE model can apply to multiple entities in a given operation provided by the homeserver. For example, the _message delivery_ action described in the [functional requirements](../functional_requirements.md) involves the client requesting that the homeserver deliver the message, the homeserver receiving and processing the request, a potential federated homeserver that stores and forwards the request and finally the user receiving the message on the other end.

We thus have three kinds of authentication rather than just one: Client to server (C2S), server to server (S2S) and client to client (C2C), all of which we consider as sub-properties of the authentication property of the STRIDE model.

## Spam reduction

Spam (outside of its potential for a denial-of-service attack on the homeserver) is a general risk for a messaging service. We talk about spam reduction rather than spam prevention, because we don't believe spam is a problem that can be avoided entirely in a privacy-preserving messenger and instead aim to reduce it to a minimum.

In the context of our threat model, we consider spam potential an additional security risk for every action that leads to messages being sent to users or that otherwise alerts the user, e.g. by triggering a notification. This is reflected by adding the security property _spam reduction_ to the existing STRIDE properties.



