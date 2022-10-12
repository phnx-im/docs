# State expiration

All pieces of state on the homeserver either have an explicit expiration date, or a timestamp that indicates the last time of use. For each type of state that is timestamped, the homeserver component has a configuration option detailing when the state expires and can be deleted.

Since this rule also goes for QS user records, users (or at least one of their clients) have to be active periodically as not to be deleted from the homeserver.

Similarly, key material with an expiration date (such as KeyPackages or [AS credentials](../authentication_service/credentials.md#as-credentials)) has to be rotated in time.

## Expiration of clients

The MLS protocol and this homeserver specification are built to support a private and secure messaging experience. One of the security properties of a modern secure messaging application is the ability of clients to recover from a potential compromise. This is realized by clients periodically rotating their key material. However, this rotation is not always possible, for example if a client is offline for longer period of time.

Thus, for security reasons, clients that are inactive for a period of time are automatically removed from a group.

The [DS](../delivery_service.md) enforces this by monitoring activity times for each member of a group. The activity time is refreshed whenever the group member sends a commit message (thus updating its key material). The time after which an inactive client is removed from the group depends on the [configuration](../delivery_service.md#ds-configuration-options) of the DS.

The [AS](../authentication_service.md) enforces this by removing expired client credentials.

## Expiration of groups

A group state is deleted if no group member has updated their state in a pre-configured interval. To enforce this, the DS stores a time stamp together with EAR encrypted group states.
