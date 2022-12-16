# Authentication in messaging applications

All of the commonly used messaging applications aim to provide authentication as one of their primary security guarantees. The goal is that recipients of a given message can authenticate the sender based on (public) cryptographic key material which we call the sender’s *cryptographic identity* (for brevity sometimes just *identity*). The cryptographic identity is typically a signature public key or a public key that can be used in an authenticated key exchange. The holder of the corresponding private key material is the owner of the cryptographic identity.

The recipient can use the cryptographic identity of the sender to either authenticate individual messages (e.g. verifying the sender’s signature on an incoming message using the sender’s public signature key) or to establish an authenticated channel to the sender by way of an initial, authenticated key agreement (e.g. a Diffie-Hellman (DH) style key exchange involving the DH public key of both parties). In both cases, the recipient relies on the fact that the cryptographic identity it uses on for authentication is indeed owned by whom the recipient thinks the sender of the message is.

This leads to the question of how the sender and receiver can ensure (or at least increase their confidence) that the cryptographic identity each has of its peer is correct.

The problem is typically solved by the sender and receiver using a trusted channel to either exchange cryptographic identities or verify them later.

## Meddler-in-the-middle (MITM) attacks

In both cases, the threat is that an adversary, often called a meddler-in-the-middle [sits between sender and receiver](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) and controls the channel they use to exchange messages. When the sender and receiver exchange their respective cryptographic identities, the adversary replaces them with ones that it controls. After such an attack, the adversary can impersonate one victim towards the other, as long as sender and receiver use a channel for communication that the adversary controls.

## Contents

In the first part of this chapter, we examine existing authentication concepts and techniques. In the second part, we provide a list of existing messaging applications, each with its corresponding approach to authentication. Finally, we provide a brief comparison and overview over the individual applications.
