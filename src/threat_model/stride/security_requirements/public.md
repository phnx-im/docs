# Public

## User registration

| STRIDE property | Requirement                                                                                             | Remark |
| --------------- | ------------------------------------------------------------------------------------------------------- | ------ |
| Authentication  | Not a risk. Registration is publicly accessible                                                         |        |
| Integrity       | The homeserver must perform checks to ensure the user id entered is valid as per specification          |        |
| Non-repudiation | The user or its client must prove possession of authentication key material                             |        |
| Confidentiality | No requirements besides the [general confidentiality requirements](../security_requirements.md)         |        |
| Availability    | User registration MAY be restricted when the homeserver has limited resources                           |        |
| Authorization   | Not a risk. No authorization required for registration                                                  |        |
| Spam prevention | To mitigate spam attacks downstream of user registration, the homeserver MAY restrict user registration |        |

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

