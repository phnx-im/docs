# Public

## User registration

| STRIDE property | Requirement                                                                                             | Remark |
| --------------- | ------------------------------------------------------------------------------------------------------- | ------ |
| Authentication  | Not a risk. Registration is publicly accessible                                                         |        |
| Integrity       | The homeserver must perform checks to ensure the user name entered is valid as per specification        |        |
| Non-repudiation | The user or its client must prove possession of authentication key material                             |        |
| Confidentiality | No requirements besides the [general confidentiality requirements](../security_requirements.md)         |        |
| Availability    | User registration MAY be restricted when the homeserver has limited resources                           |        |
| Authorization   | Not a risk. No authorization required for registration                                                  |        |
| Spam prevention | To mitigate spam attacks downstream of user registration, the homeserver MAY restrict user registration |        |