---
layout: default
title: chaincode_go_new_programming_model
parent: RFCs
---

- Feature Name: chaincode_go_new_programming_model
- Start Date: 2019-11-15
- RFC PR:
- Fabric Component: fabric-contract-api-go
- Fabric Issue: FAB-16640

# Summary
[summary]: #summary

The updated programming model has been implemented in Java and Node [FAB-11246](https://jira.hyperledger.org/browse/FAB-11246). This RFC is therefore suggesting that Go chaincode be updated to follow this new model.

This new implementation will be situated in a new repo `fabric-contract-api-go` with myself (@awjh-ibm) and @heatherlp as initial maintainers.

# Motivation
[motivation]: #motivation

The existing programming model in Go requires a certain amount of boiler plate code to be written by the developer, things such as routing in the Invoke function, converting data from its arg type to the usable type in the transaction and from its transaction type to the return type. The new programming model as implemented in Java and Node reduces this boiler plate for developers and minimizes the amount of code they have to write to implement their smart contract. 

The existing programming model in Go provides zero way for an outside application or developer (with access to the network) to know the interface that the chaincode provides. By moving to the new programming model metadata will be made available for Go chaincode allowing developers to build external applications for interacting with the smart contract without needing to see the code.

There is further motivation in including this in Go for synchronisation across supported fabric languages.

Motivation for the new programming model as used in Java and Node can be found [here](https://docs.google.com/document/d/1_np3fnT_OludRGcF3PbubDooNsH8J-_G7UaWhk8a_cU/edit?usp=sharing)

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explanation was discussed on the community call and shared in the mailing list. The PDF of this can be found in the mailing list here: https://lists.hyperledger.org/g/fabric/topic/55666355

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Explanation was discussed on the community call and shared in the mailing list. The PDF of this can be found in the mailing list here: https://lists.hyperledger.org/g/fabric/topic/55666355

# Drawbacks
[drawbacks]: #drawbacks
N/A

# Rationale and alternatives
[alternatives]: #alternatives

The rationale behind this is explained in the motivation.

An alternative would be to remain as is and leave Go chaincode without the benefits of the new programming model and out of step with Node and Java.

# Prior art
[prior-art]: #prior-art

Other chaincode languages already include the feature:
- Node https://github.com/hyperledger/fabric-chaincode-node
- Java https://github.com/hyperledger/fabric-chaincode-java

Proposed implementation for Go can be found here:
https://github.com/awjh-ibm/fabric-contract-api-go

# Testing
[testing]: #testing

- Unit tests will be written for new code
- Functional tests in cucumber to cover new package
- Integration tests added to the upcoming chaincode test repository [FAB-15254](https://jira.hyperledger.org/browse/FAB-15254)

# Dependencies
[dependencies]: #dependencies

- New samples will need to be written to cover the new programming model like has been added for Java and Node:
    - fabcar ([implementation](https://github.com/awjh-ibm/fabric-samples/tree/fabcar-go/chaincode/fabcar/go))
    - commercial paper ([implementation](https://github.com/awjh-ibm/fabric-samples/tree/commercial-paper/commercial-paper))

# Unresolved questions
[unresolved]: #unresolved-questions
N/A