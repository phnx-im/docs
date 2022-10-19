# Application assets

The homeserver will have to perform all actions listed for the individual roles in the [functional requirements section](./functional_requirements.md).

To fulfill these requirements, the homeserver will keep the following state (information assets in STRIDE terminology). Each piece of state is annotated by the actions involved in its lifecycle.

* Group state (for message delivery, including associated metadata such as group membership lists)
    * Create: Group creation
    * Read, Update: Message delivery (with inline MLS group management)
    * Delete: Group deletion
* [KeyPackages](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#name-key-packages) for retrieval by clients
    * Create/Update/Delete: KeyPackage publishing (publication of new KeyPackages implies deletion of old ones)
    * Read/Delete: KeyPackage retrieval (reading implies deletion, except for [KeyPackages of last resort](https://www.ietf.org/archive/id/draft-ietf-mls-protocol-16.html#name-keypackage-reuse))
* Authentication key material of users and their clients
    * Create/Update/Delete: Client management
    * Update: Account reset
    * Delete: Account deletion
    * Read: Client authentication
* User names
    * Create: User registration
    * Update: User name change
    * Read: User discovery
    * Delete: Account deletion

This section will be extended with more information assets as the specification is written.