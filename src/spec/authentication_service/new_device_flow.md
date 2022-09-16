# Registering a new device

To register a new client, the user needs to be in possession of one of his existing clients. The flow to add a new client is as follows:

* The new client contacts the AS and asks for a rendez-vous code.
* The AS responds with a 16bit rendez-vous code.
    * It might be more than 16bit if the 16bit space was exhausted by an adversary.
* The new client samples a 16bit random string and prompts the user to enter both the rendez-vous code and the random string into the existing client.
* The new client uses the random string and the rendez-vous code to perform the first part of an SPAKE2 handshake, where the random string is used as the password and the rendez-vous code is used as the identity. It then establishes a websocket connection with the AS using the rendez-vous code and uploads the SPAKE2 outgoing message.
* The existing client performs the responder's part of the SPAKE2 handshake and also establishes a websocket connection to the AS using the rendez-cous code. It then uploads the resulting outgoing message to the AS, picking up the new client's outgoing message in the process and completes the first phase of the SPAKE2 handshake.
* The AS alerts the new client of the new upload.
* The new client downloads the outgoing message, completes phase 2 of the SPAKE2 and uploads the key confirmation message.
* The existing client downloads the confirmation message and posts its own confirmation message.
* The new client downloads the existing client's confirmation message and uploads a CSR, which it AEAD-encrypts using the agreed-upon key.
* The existing client downloads the CSR and submits it to the AS for signature, signing the request with its client credential.
  * TODO: CSR submission has to happen via another endpoint.
* The AS validates the signature and the CSR and sends the signed credential to the new client via the websocket connection.
  * TODO: The AS shouldn't send the signed CSR to the new client, but to the existing client, which then forwards it.
* The new client creates a KeyPackage for the EID, as well as for the user's all-clients group and uploads it, along with the corresponding intermediate certificate. All uploads are encrypted using the AEAD key.
* The AS notifies the existing client of the upload.
* The existing client downloads and validates it and uses the KeyPackage to add the new client to the EID. It sends the resulting commit to all other existing clients and the AS. The existing client also adds the new client to the user's all-clients group. It then uploads the Welcome for the EID and the all-clients group to the AS. Note that the Welcome for the all-clients group contains the EAR secrets for the group.
* The new client downloads the Welcome and joins the group. It then requests anonymous tokens from the AS, authenticating the request with its client credential.
* The AS repsonds with the tokens.
* The new client contacts the QS to create its queues.
* The existing client sends the client state to the new client via the all-client group. The new client state includes state-bundles for all groups that the user is in. These bundles include the group id, member EAR secrets and PublicGroupStates of each group.
* The new client injects itself into all of the user's group using the state-bundles. It sends attached the two commits for the EID state.

TODO: The new client needs to publish key packages.
TODO: The new client needs to post an update to complete cross signing.
TODO: Be more precise about what state is sync'ed to the new client.
TODO: Be more precise about token requests and queue creation.
TODO: The new client also needs the queue creation key early.
TODO: Update using the swimlanes diagram: https://swimlanes.io/#nVTBbtswDL37K3jcgCbAetghhwFem8PQbSjqYjsWss0kWmxJE+l12dePku3Gcep42CGAI5Hv8ZFPZM0VruArPkNRaTQMubVM7JVz2mzBeVsgUZKEiJs2YvEB0mwFD/izQWLwaEr8s/hlG4LClpgkxrJgPu4QBlmkalchgYJ373PNUFmB98qUtgbhC2TyJxDWjglYshtCD2xB0uUjt7yLx2PCmBcvTtC0kdRwjL81cThpFS7hY0Bq/0QiAw79xvoasvv0bn0txCH8dS4CXUqi5sMEr0Q4RfRsfblMkq23jVvBN63O8XbKGKzGvV335bZHKyCn9nj9RMiNe6pp+6SNZq3Y+iQZBYf8I9orqR7JWSnB91M66YWxYgPpf8MYq+3ascdW6/AqnBXWbLSvFWtroBabqC3S8l/1DLP/T9YZwkBd1/d0nd6Kfwp/cIzlYLBRWlAxSxZdaFQ9ZezYNI+Ko7lvsof5DkjQEGxcQED07etqH0KaAemtid8twUzNmUSL3BHPZNF3eLhXxV7GB/IMIs36060YWu7n1byEDnDm5PXcgUpsVctCUGXZD2dA2L3hwBEsSE0usfIGX27SbL4d37ESEgwpAejL5wyiP67auhd5Y8qwm4J4VVXtyGMEXcHGC0hJO+0urbehqFCgGFN2YydtIKJt1AUpwhV/FzYu2z0a2clii5HQx+7mWKI5bvZxiQLWhPGHrdvklaZdd3EcY3jN04MslImgG+Rid2qjN8eXNuhfLPxtZJRpT4xaBrDoF1KcwXJCj7NV1RbcAURBEf2H1YYm0UY7+SxkNWuotDcJoXSUKfkL