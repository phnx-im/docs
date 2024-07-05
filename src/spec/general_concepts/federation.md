# Federation

The specification supports federation, which means that users of one homeserver can

- establish connections with
- receive messages from and
- send messages to

users on other, federated homeservers. Two homeservers are federated if the configurations of both homeservers permit the federation.

## Federated access control and rate-limiting

Just like the clients of a given homeserver, federated homeservers are also subject to rate-limiting and have to provide [anonymous tokens](anonymous_tokens.md) to access the homeserver's endpoints.

A homeserver can thus control access by federated homeservers by restricting token issuance.

## Federated endpoints

There are two sets of endpoints that are used across homeservers.

One set is used by the federated homeserver, e.g. to fetch key material or to deliver messages from a group on the federated homeserver to a set of clients on the local homeserver.

The other set is used by clients of the federated homeserver, e.g. to fetch KeyPackages of a local user, or to send messages to a group on the local homeserver.