# Queue ID encryption and enqueue authorization

* Client queue ids are encrypted under their QS' queue ID encryption key
* The ciphertext is tagged with the QS' domain

TODO: It's not clear how the authorization works. Are you authorized to enqueue just by having one such ciphertext? If we want the client to publish these ciphertexts with the client's KeyPackages, we can't include the DS' domain (because we don't know a priori where it's used). We could just encrypt a token along with the queue ID, although then we might as well just use the ciphertext.