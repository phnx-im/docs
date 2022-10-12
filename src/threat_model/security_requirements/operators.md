# Operators

The actions by the operator only involve the operator and the homeserver. They thus don't involve other authentication beyond the homeserver authenticating the operator.

Since operators are trusted not to spam their users as per [security assumptions](../security_assumptions.md), spam prevention will not be listed for operator actions.

## Homeserver management

| STRIDE property | Requirement                                                             | Remark |
| --------------- | ----------------------------------------------------------------------- | ------ |
| Authentication  | Only operators can perform management actions                           |        |
| Integrity       | The homeserver must perform checks to ensure the configuration is valid |        |
| Non-repudiation | Not a risk. There is only one homeserver operator                       |        |
| Confidentiality | Not a risk. Homeserver management actions are not confidential          |        |
| Availability    | Operators should always be able to perform management actions           |        |
| Authorization   | Not a risk. There is only one homeserver operator role                  |        |

## Homeserver setup

| STRIDE property | Requirement                                                             | Remark |
| --------------- | ----------------------------------------------------------------------- | ------ |
| Authentication  | Not a risk. The entity setting up the server can only be the operator   |        |
| Integrity       | The homeserver must check if the homeserver domain is a FQDN at startup |        |
| Non-repudiation | Not a risk. At the time of setup, identities don't exist yet            |        |
| Confidentiality | Not a risk. The home domain is public                                   |        |
| Availability    | This is a setup action. The homeserver is not running at that time      |        |
| Authorization   | Not a risk. Setup is done by a single entity.                           |        |

## Federation configuration

| STRIDE property | Requirement                                                                          | Remark |
| --------------- | ------------------------------------------------------------------------------------ | ------ |
| Authentication  | Only administrators can configure federation                                         |        |
| Integrity       | Not a risk. Operators are responsible for checking the validity of the configuration |        |
| Non-repudiation | Not a risk. There is only one homeserver operator                                    |        |
| Confidentiality | Not a risk. Federation configuration actions are not confidential                    |        |
| Availability    | Operators should always be able to manage homeserver federation                      |        |
| Authorization   | Not a risk. There is only one homeserver operator role                               |        |