---
layout: default
title: Chaincode Builder for Kubernetes
nav_order: 3
---

- Feature Name: Chaincode Builder for Kubernetes
- Start Date: 2022-12-16
- RFC PR: (leave this empty)
- Fabric Component: fabric-builder-k8s
- Fabric Issue: (leave this empty)

# Summary
[summary]: #summary

Promote the [Fabric Chaincode Builder for Kubernetes Hyperledger Labs project](https://github.com/hyperledger-labs/fabric-builder-k8s) to a Fabric subproject.

# Motivation
[motivation]: #motivation

The experimental Fabric Chaincode Builder for Kubernetes project has been well received, forming part of the successful [full stack asset transfer workshop](https://github.com/hyperledger/fabric-samples/tree/main/full-stack-asset-transfer-guide) presented at the Hyperledger Global Forum in Dublin, and the next step is for it to graduate to an official Fabric subproject.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Starting with Fabric 2.0, [External Builders and Launchers](https://hyperledger-fabric.readthedocs.io/en/latest/cc_launcher.html) enable the peer to be extended with programs that can build, launch, and discover chaincode.
The Chaincode Builder for Kubernetes enables peers to launch containerized chaincode using Kubernetes.

Advantages:

- prepublished chaincode images avoids compile issues at deploy time
- standard CI/CD pipelines can be used to publish containerized chaincode images
- chaincode traceability using [immutable image digests](https://docs.docker.com/engine/reference/commandline/pull/#pull-an-image-by-digest-immutable-identifier)

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is an existing [Hyperledger Labs project](https://github.com/hyperledger-labs/fabric-builder-k8s) which works with Fabric's [External Builders and Launchers](https://hyperledger-fabric.readthedocs.io/en/latest/cc_launcher.html) feature.

# Drawbacks
[drawbacks]: #drawbacks

Adding another Fabric subproject increases the overall maintenance effort required for Hyperledger Fabric, however `fabric-builder-k8s` is a small self-contained repository so the impact should be minimal.

# Rationale and alternatives
[alternatives]: #alternatives

The existing built-in chaincode types have significant issues with reliability and repeatability, since they actually build the chaincode source code at deploy time.
They also do not work well in a cloud environment due to the requirement for Docker in Docker.

There are other external builders and launchers:

- The Chaincode as an External Service builder which was recently added to the main Fabric repository.  
This is a good option for debugging chaincode at development time however it breaks the chaincode lifecycle since you end up approving _where_ the chaincode is running, not _what_ is running.
- Various third party builders, some of which are open source.  
These tend to follow the same pattern as the built-in chaincode types, with similar disadvantages.

# Prior art
[prior-art]: #prior-art

[Original Hyperledger Labs project proposal](https://labs.hyperledger.org/labs/fabric-builder-k8s.html)

# Testing
[testing]: #testing

An end to end test will require a Kubernetes environment and should focus on how the builder behaves when Kubernetes pods already exist, or are stopped unexpectedly.

# Dependencies
[dependencies]: #dependencies

The Fabric [External Builders and Launchers](https://hyperledger-fabric.readthedocs.io/en/latest/cc_launcher.html) feature.

# Unresolved questions
[unresolved]: #unresolved-questions

The core implementation is already complete, however there is scope for informing changes to the current Fabric [External Builders and Launchers](https://hyperledger-fabric.readthedocs.io/en/latest/cc_launcher.html) implementation in the future which might improve the integration with Kubernetes, for example to allow the Chaincode Builder for Kubernetes to start chaincode as an external service.
