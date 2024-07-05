# Federation Protocol

The [federation functionality](./use_cases.md#federation-functionality) of the Phoenix homeserver protocol allows users to communicate across homeserver instances.

## Overview

The federation functionality includes the following types of interactions:

- Client to remote DS: Allows clients to send messages (and generally act as members of) groups on DS’ that are not their own, or fetch the remote DS’ authentication key material.
- DS to remote QS (via the DS’ local QS): Allows a DS to distribute messages to clients that don’t have their queue on the local QS, or fetch the remote QS’ authentication key material.
- Client to remote AS: Allows clients to fetch tokens which in turn allows them to interact with the DS of the remote homeserver, or fetch the remote AS’ authentication key material.
- As to remote AS: Allows the AS to fetch the authentication key material of the remote AS.

The majority of the interactions listed above are exactly the same as when the equivalent local parties communicate. The following details the points where the interactions are different, as well as an additional protocol for QS-to-QS communication.

## QS-to-QS communication

The QS has a [separate endpoint](./../queuing_service.md#federated-qs-to-qs-communication) for federated communication that allows a remote QS to deliver messages to a local client. The payload delivered to this endpoint is the same as that of a local DS distributing messages to its local QS.

However, the sending QS frames that payload with an additional authentication frame as defined in the description of the endpoint linked above.

## Client to remote AS communication

Clients can request tokens from a remote AS via the [appropriate endpoint](../authentication_service.md#token-issuance) for redemption at the DS of the same remote homeserver. The protocol on the wire is exactly the same as for a local client requesting tokens from their own local AS. However, for authentication, the remote AS might have to fetch the authentication key material of the sending client’s AS (which it can do just like local clients of that AS) s.t. it can authenticate the request. It can then rate-limit the request both on a per-client basis, as well on the basis of the homeserver the client originates from.
