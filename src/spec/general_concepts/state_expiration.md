# State expiration

## Expiration of clients

The MLS protocol and this homeserver specification are built to support a private and secure messaging experience. One of the security properties of a modern secure messaging application is the ability of clients to recover from a potential compromise. This is realized by clients periodically rotating their key material. However, this rotation is not always possible, for example if a client is offline for longer period of time.

Thus, for security reasons, clients that are inactive for a period of time are automatically removed from a group.

The [DS](../delivery_service.md) enforces this by monitoring the expiration data in the [Leaf Credential](../authentication_service/credentials.md#leaf-credentials) of the clients of each individual group.

TODO: Not sure if the Leaf Credential expiration is the best way to monitor user activity. We should probably introduce a LeafNode extension that contains a time stamp.
TODO: The time duration should probably be configurable on a per-homeserver basis. How do clients learn this duration? We could publish it in the same well-known place we publish public keys. Generally, how do clients learn of changes to such configuration values?
TODO: How to enforce this on other endpoints, e.g. the AS, where we don't have a Leaf Credential available?

## Expiration of homeserver state

All pieces of state on the homeserver are associated with timestamp. This is in particular true for group state data on the DS, queues and KeyPackages on the QS and authentication data on the AS. Since there is already a period of time after which a [client expires](./state_expiration.md#expiration-of-clients), all state that is associated with a given client expires in the same way if the client expiration time has elapsed since the client has last interacted with the given piece of state.

For example, a group state is deleted if no group member has updated their state during one client expiration period. Similarly, a queue is deleted if a client has not fetched messages from that queue for the same period of time.

This mechanism prevents stale data from accumulating, which acts as a privacy preserving measure for clients and a resource saving mechanism for the homeserver operator.

TODO: Detail time stamping for all pieces of state in all services. Alternatively push it to future work.