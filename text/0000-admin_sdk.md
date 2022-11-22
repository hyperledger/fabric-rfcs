---
layout: default
title: Fabric admin SDK
nav_order: 3
---

- Feature Name: Fabric admin SDK
- Start Date: 2022-11-11
- RFC PR: (leave this empty)
- Fabric Component: fabric-admin
- Fabric Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes a new administrative SDK for Fabric to support implementers of Kubernetes operators, CLI commands, and automated deployment pipelines. The primary target implementation language for this API is Go, with the aim to deliver implementations in the same set of languages as the client application APIs, namely: Go, Node (TypeScript / JavaScript) and Java. Key objectives are to provide consistent capability across all implementation languages, and to avoid any dependency on core Fabric since this is not intended to be consumed as a library.


# Motivation
[motivation]: #motivation

There are two primary motivators for this work:

1. Legacy client application SDKs developed alongside Fabric v1 provide varying levels of administrative capability, and this is relied upon by a subset of the Fabric user base to support automated deployment. Client application APIs developed alongside Fabric v2 focus only on the Fabric programming model, supporting business applications to submit and evaluate transactions, and to receive events. Fabric v2.5 plans to deprecate the legacy SDKs, leaving a gap in support for programmatic deployment of Fabric.

2. The `peer` command provides both the server-side peer and the Fabric CLI implementations. Therefore the CLI commands are tightly coupled to the core Fabric codebase. A new Fabric admin API would provide a cleaner base on which to build Fabric CLI commands, decoupled from the core Fabric codebase.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Fabric admin API aims to deliver the following minimum set of capability:

- Chaincode deployment (using v2 chaincode lifecycle):
  - Install chaincode
  - Query installed chaincode
  - Approve installed chaincode
  - Query approved chaincode
  - Commit approved chaincode
  - Query committed chaincode

- Channel configuration:
  - Create channel
  - Join peers
  - Set anchor peers

Additional capability may be included to aid deployment or common configuration tasks, where this does not represent duplication of capability already easily achievable using existing programmatic APIs.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The implementation takes the following design approach:

- Expose a relatively simple API that provides a layer of abstraction around the mechanics of invoking gRPC and HTTP/RESTful services provided by Fabric to achieve administrative tasks.
- Loose coupling between API implementation and network connection management, allowing the caller to retain control of creation and configuration of network connections.
- Loose coupling between API and cryptographic implementations, allowing the caller to inject the cryptographic credentials and signing implementation.
- Consistency of capability across different language implementations while following language idioms.

The admin SDK interacts with Fabric using only well-defined gRPC services and HTTP/RESTful APIs. To simplify the implementation and provide consistency with current client application APIs, this includes the use of Gateway gRPC services, either directly using gRPC client APIs or using the Fabric Gateway client API.

## Example API methods

There follow some examples of proposed API calls provided by the admin SDK.

### Install chaincode

```go
func Install(
    ctx            context.Context,
    peerConnection grpc.ClientConnInterface,
    signer         identity.SignerSerializer,
    packageReader  io.Reader,
    callOptions    ...grpc.CallOption,
) error
```

- `ctx` allows the caller to cancel the operation, either directly or after a timeout period.
- `peerConnection` caller provided gRPC connection used to make the call, which has been configured appropriately and may be shared between multiple calls.
- `signer` encapsulates the signing implementation and client credentials.
- `packageReader` supplies the chaincode package content.
- `callOptions` are call-specific gRPC options.

### Query installed chaincode

```go
func QueryInstalled(
    ctx            context.Context,
    peerConnection grpc.ClientConnInterface,
    signer         identity.SignerSerializer,
    callOptions    ...grpc.CallOption,
) (*lifecycle.QueryInstalledChaincodesResult, error)
```

The parameters are common with the proposed install chaincode API. The return value is a protocol buffer message, defined by Fabric and included in [fabric-protos](https://hyperledger.github.io/fabric-protos/).

# Drawbacks
[drawbacks]: #drawbacks

An additional SDK requires additional development effort, support and ongoing maintenance.

While significantly simplifying the implementation and maintenance burden, making use of Gateway gRPC services and/or the Fabric Gateway client API limit use of the admin SDK to Fabric v2.4 and later. The admin capability in legacy SDKs is available for earlier Fabric versions.

# Rationale and alternatives
[alternatives]: #alternatives

Administrative tasks are currently possible using the Fabric CLI and an alternative is to continue with them as the only supported administrative tool. However, there is real community interest in programmatic configuration, and community members already actively contributing to an admin SDK implementation. A Go admin SDK dramatically simplifies the development and maintenance effort required to implement both Kubernetes operators and new Fabric CLI commands.

# Prior art
[prior-art]: #prior-art

Legacy SDKs provide varying levels of admin capability. Their development in entirely separate codebases has naturally led to inconsistency in capability and API design. As part of larger packages, not focused purely on admin tasks, the ongoing maintenance and evolution of their admin capability has become impractical.

Active development on a new admin SDK is taking place at [Hyperledger-TWGC/fabric-admin-sdk](https://github.com/Hyperledger-TWGC/fabric-admin-sdk). This RFC proposes adopting that as the basis of a Hyperledger Fabric admin SDK.

# Testing
[testing]: #testing

In additional to typical test-driven development using unit tests for specific language implementations, integration tests will be required to confirm that admin capabilities work correctly with a real Fabric deployment. These integration tests should be fully automated and run as part of the continuous integration pipeline.

Since one of the goals is to provide consistent capability across different language implementations of the admin SDK, it is desirable to run a consistent set of integration tests against each language implementation. One possible approach for achieving this is to produce language-agnostic test definitions that are run against all language implementations. The approach has been used successfully in the [Fabric Gateway client API](https://github.com/hyperledger/fabric-gateway/tree/main/scenario/features) using [Cucumber](https://cucumber.io/) as the test framework.

# Dependencies
[dependencies]: #dependencies

The admin SDK is expected to include [fabric-protos-go-apiv2](https://github.com/hyperledger/fabric-protos-go-apiv2) and possibly also [fabric-gateway](https://github.com/hyperledger/fabric-gateway) as direct dependencies. There will be dependencies on [gRPC](https://grpc.io/) APIs to interact with gRPC services provided by Fabric, both directly and indirectly through the protocol buffer bindings.

Dependencies **must not include** core Fabric, which is not intended to be consumed as a library. Where utilities contained within core Fabric are found to be broadly applicable both in Fabric and the admin SDK (and ideally also in other projects), those utilities could be extracted to a separate project, such as [fabric-lib-go](https://github.com/hyperledger/fabric-lib-go) and [fabric-config](https://github.com/hyperledger/fabric-config).

# Unresolved questions
[unresolved]: #unresolved-questions

The fine details of the admin API are expected to evolve as capability is implemented as user stories. The end-user usage experience provided by the API in specific scenarios, such as implementation of a Kubernetes operator and automated Fabric deployment, will guide the design and implementation. The intention is to socialize the API with the community throughout the development process to solicit feedback.

While one of the motivators for an admin API is to provide a foundation on which new Fabric CLI commands are built, those CLI commands are out of scope for this RFC.
