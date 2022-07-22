# Open MLS Infrastructure Project Status

## Update 1, July 22 2022

Today, we publish a [threat model](./threat_model.md) for an abstract homeserver that aims fulfills the functional requirements. The threat model also contains privacy and security requirements. This is a first iteration and just like the functional requirements, we expect this to be a living document that grows as we move forward with this project.

## Update 0, July 19 2022

A few days ago, we published an initial draft of the functional requirements for a basic homeserver design that enables MLS based secure messaging functinonality. Soon to come are an initial draft for the threat model, as well as a PoC implementatin for a PrivacyPass based anonymous rate-limiting module, which is (in all likelihood) going to be part of the final, modular homeserver implementation.

Meanwhile, we are working on refactoring the TreeSync code in OpenMLS so that we can re-use some of the code as part of the message delivery module.