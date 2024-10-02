# Public

## Get AS credentials

| STRIDE property | Requirement                                                                                                                                   | Remark |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------- | ------ |
| Authentication  | AS credentials are publicly available to everyone                                                                                             |        |
|                 | Clients must be able to authenticate intermediate AS credentials using the corresponding AS credential                                        |        |
|                 | The basic [confidentiality and authentication requirements](./../security_requirements.md#basic-confidentiality-and-authentication) must hold |        |
| Integrity       | Not a risk as this is a read-only operation                                                                                                   |        |
| Non-repudiation | Not a risk. There is no need to trace                                                                                                         |        |
| Confidentiality | Basic [confidentiality and authentication requirements](./../security_requirements.md#basic-confidentiality-and-authentication) must hold     |        |
|                 | Enqueued messages must be encrypted according to the specification                                                                            |        |
| Availability    | Clients must always be able to obtain AS credentials                                                                                          |        |
| Authorization   | Not a risk. All clients are allowed to obtain AS credentials                                                                                  |        |
| Spam prevention | Not a risk as getting AS credentials is not message-sending                                                                                   |        |

