---
layout: default
title: Fabric Gateway
nav_order: 3
---

- Feature Name: Fabric Gateway
- Start Date: 2020-09-08
- RFC PR: (leave this empty)
- Fabric Component: fabric-gateway
- Fabric Issue: (leave this empty)

# Summary
[summary]: #summary

The Fabric Gateway is a new component that will run either as a standalone process, or embedded with the peer, and will implement as much of the high-level 'gateway' programming model as possible.  This will remove much of the transaction submission logic from the client application which will simplify the maintenance of the SDKs.  It will also simplify the administrative overhead of running a Fabric network because client applications will be able to connect and submit transactions via a single network port rather than the current situation where ports have to be opened to multiple peers across potentially multiple organisations.

# Motivation
[motivation]: #motivation

The high-level "gateway" programming model has proved to be very popular with users since its introduction in v1.4 in Node SDK.  Since then, it has been implemented in the Java and Go SDKs.  This has led to a large amount of code that has to be maintained in the SDKs which increases the cost of implementing new feature which have to be replicated to keep the SDKs in step.  It also increases the cost of creating an SDK for a new language.  Moving all of the high-level logic into its own process (or peer) and exposing a simplified gRPC interface will drastically reduce the complexity of the SDKs.

By providing a single entry point to a Fabric network, client applications can interact with complex network topologies without multiple ports having to be opened and secured to each peer and ordering node.  The client will connect only to the gateway which will act on its behalf with the rest of the network.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Fabric Gateway is an embodiment of the high-level 'gateway' Fabric programming model in a server component that will form part of a Fabric network alongside Peers, Orderers and CAs.  It will either be stood up inside its own process (optionally in a docker container), or it will be hosted inside a peer.  Either way, it exposes its functionality to clients via a gRPC interface.  The Gateway server component will be implemented in Go.

Lightweight client SDKs are used to connect to the Gateway for the purpose of invoking transactions.  The gateway will interact with the rest of the network on behalf of the client eliminating the need for the client application to connect directly to peers and orderers.

The concepts and APIs will be familiar to Fabric programmers who are already using the new programming model.

The scope of functionality exposed to the client by this component will include transaction evaluate/submit and event listening.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The Gateway runs as a client process associated with an organisation and requires an identity (issued by the org's CA) in order to interact with the discovery service and to register for events (Deliver Client).  

The Gateway also runs as a gRPC server and exposes the following services to client applications (SDKs):

```
service Gateway {
    rpc Endorse(ProposedTransaction) returns (PreparedTransaction) {}
    rpc Submit(PreparedTransaction) returns (stream Event) {}
    rpc Evaluate(ProposedTransaction) returns (Result) {}
}
```

This exact detail of this protobuf definition may evolve slightly depending on implementation experience, but this shows the general design.

The client credentials (private key) are never passed to the Gateway.  Client applications are responsible for managing their user keys (optionally using SDK Wallets) and signing the protobuf payloads.

The SDKs will use these services to implement the `SubmitTransaction` and `EvaluateTransaction` functions as follows:

__Submitting a transaction__ is a two step process (performed by the client SDK):
- __Endorse__
  - The client creates a ProposedTransaction message, which contains a SignedProposal message as defined in `fabric-protos/peer/proposal.proto` and signed with the user's identity.  The ProposedTransaction message also contains other optional properties, such as the endorsing orgs if the client wants to specify that.
  - The `Endorse` service is invoked on the Gateway, passing the ProposedTransaction message
    - The Gateway will determine the endorsement plan for the requested chaincode and forward to the appropriate peers for endorsement. It will return to the client a `PreparedTransaction` message which contains a `Envelope` message as defined in `fabric-protos/common/common.proto`.  It will also contain other information, such as the return value of the chaincode function and the transaction ID so that the client doesn't necessarily need to unpack the `Envelope` payload.
- __Submit__
  - The client signs the hash of the Envelope payload using the user's private key and sets the signature field in the `Envelope` in the `PreparedTransaction`.
  - The `Submit` service is invoked on the Gateway, passing the signed `PreparedTransaction` message.  A stream is opened to return multiple return values.
    - The Gateway will register transaction event listeners for the given channel/txId.
    - It will then broadcast the `Envelope` to the ordering service.
    - The success/error response is passed back to the client in the stream
    - The Gateway awaits sufficient tx commit events before returning and closing the stream, indicating to the client that transaction has been committed.

__Evaluating a transaction__ is a simple process of invoking the `Evaluate` service passing a SignedProposal message.  The Gateway passes the request to a peer of it's choosing according to a defined policy (probably same org, highest block count) and returns the chaincode function return value to the client.

## Launching the Gateway

#### Standalone process
When running standalone, the Gateway is effectively a 'client' application to a set of peers.  A gateway instance is associated with an organization and will be configured to connect to a peer in the same organization in order to invoke discovery.  To do this it will need a signing identity in the channel writers policy.

To run the gateway server, the following parameters must to be supplied:
- The url of at least one peer in the org.  Once connected, the discovery service will be invoked to find other peers.
- The MSPID associated with the gateway.
- The signing identity of the gateway (cert and key), e.g. location of PEM files on disk.  
- The CA certificate so that the gateway can make TLS connections to other components (peers/orderers) in the organization

Note that the signing identity is required so the gateway server can make discovery requests and to register event listeners.  It is not used for submitting any transactions on behalf of end users.

#### Embedded in Peer
When embedded in a peer, the gateway will register its gRPC service with the host peer's gRPC server.
The gateway does not need its own (client) identity since it can get all the discovery and event information directly from the host peer rather than via a gRPC signed request.

### Discovery

On startup, the gateway will connect to its primary peer(s), as specified in the command line option (or its host, if embedded in a peer).  The discovery service will be invoked to build and maintain a cache of the network topology per channel.  This will be used to identify endorsers based on the chaincode endorsement policy.

#### Collection endorsement policy

Discovery can be used to get endorsing peers for a given chaincode where the transaction function will write to a given collection, and this will take into account the collection level endorsement policy.
Ideally we could take care of these endorsement requirements in the Gateway without the client needing to have any implicit knowledge of them. Suggest extending contract metadata to include information on collections, which can be queried by Fabric Gateway. If that isn't possible then we might need some way for the client to specify the collections that will be used (not desirable).

### State based endorsement

This is driven by business logic and rules. Requires the client to supply constraints on endorsing organisations as part of the proposal endorsement.

### Chaincode-to-chaincode endorsement

Augment contract metadata for transactions with additional chaincodes / transactions that will be invoked, then we use discovery to narrow the set of endorsers.

### Proposal response consistency checking

Current client SDK implementations do minimal checking of proposal responses before sending to the orderer. There have been some requests for more rigorous checking of proposal responses prior to submit to the orderer.

Prior to sending to the orderer, Fabric Gateway should check that proposal responses are:
- Consistent, including having matching response payloads.
- Meet all applicable endorsement requirements.

### Ensure query results reflect latest ledger state

Query results should be obtained from peers whose ledger height is at least that of the most recent events sent to clients.

Scenario:
- Client receipt of chaincode event triggers business process.
- Business process queries ledger to obtain private data, which cannot be included in event payload since it is private.
The private data must be consistent with the state of the ledger after commit of the transaction that emitted the chaincode event, so the query must be processed by a peer with ledger height of at least the block number that contained the chaincode event.

Implementation options:
- Only direct queries (and endorsement requests) to the peer(s) with the highest ledger height.
- Client proposals include last observed ledger height so Gateway can target any peers with at least that ledger height.
 A view of ledger height can be tracked by the Gateway from block events it has received.

## Scaling and load balancing

Currently a client application is responsible for load balancing its requests between multiple peers.  The Gateway will handle this on behalf of the set of currently connected client applications.

Ideally, the Gateway itself will be stateless (i.e. not maintain client session state) allowing clients to be routed though an appropriate load balancer.

## Trust assumptions

The Gateway is acting as a proxy between the client application and the peers/orderers.  The signed transaction proposal is created in the client using one of the SDKs, and gateway passes that directly to the endorsing peers without any modification.  If the client wishes, it can inspect the contents of the 'PreparedTransaction' message returned by the Endorse function prior to signing and submitting.   The Gateway will partially unpack the proposal responses in order to add the chaincode function return value and the transaction id to the PreparedTransaction message.  This allows the clients to access these commonly used values without the overhead of unpacking the deeply nested transaction 'Envelope' payload.

Trusting clients can call the SDK's submitTransaction function to endorse/sign/submit in a single line of code for convenience. Users of the existing high level programming model are already comfortable with this procedure. 

## Authentication and Authorization

The Gateway itself will not add any message level authentication mechanism other than that already provided by the peers.  As a proxy, authentication and authorization is handled by peer/orderer nodes exactly as if the client was connecting to them directly.

Connection of the client applications to the Gateway can be secured by mutual TLS if required.

The standalone Gateway does need to have an identity that allows it to connect to peers as a client in its own right to:
- Do network discovery to obtain network topology and endorsement plans.
- Listen to events to allow it to check for commit disposition of submitted transactions that it has routed to the ordering service on behalf of a client.

## Gateway SDKs

A set of SDKs will be created to allow client applications to interact with the Fabric Gateway.  The APIs exposed by these SDKs will be, where possible, the same as the current high-level 'gateway' SDKs.

The following classes/structures will be available as part of the gateway programming model:
- Gateway
- Network
- Contract
- Transaction

The existing `submitTransaction()` and `evaluateTransaction()` API methods will be implemented using the Gateway gRPC services described above.

Based on user feedback of the existing gateway SDKs, the following additions are planned to be supported by this implementation:
- Support for 'offline signing', allowing clients to write REST servers that don't directly have access to users' private keys.
- Support for 'asynchronous submit', allowing client applications to continue processing while the transaction commit proceeds in the background. For Go this will be implemented by returning a channel; for Node by returning a Promise; and for Java by returning a Future.

SDKs will be created for the following languages:
- Node (Typescript/Javascript)
- Go
- Java

It is proposed that these SDKs will be maintained in the same GitHub repository as the gateway itself (`hyperledger/fabric-gateway`) to ensure that they all stay up to date with each other and with the core gateway.  This will be enforced in the CI pipeline by running the end-to-end scenario tests across all SDKs.

Publication of SDK releases will be as follows:
- Node SDK published to NPM.
- Java SDK published to Maven Central.
- Go SDK published to separate GitHub repo (e.g. fabric-gateway-go) and tagged.

Contributors are invited to create SDKs for other languages, e.g. Python, Rust.


# Drawbacks
[drawbacks]: #drawbacks

When run standalone, the gateway would add another process to an (arguably) already complex Fabric architecture.  Also it will require its own signing identity in the channel writers policy in order to invoke the discovery service, which might be a security concern in some environments. These concerns are mitigated by embedding the gateway server within the organisation's existing peers.  

However, if embedded in an existing peer, then the gateway will not be able to be scaled independently of the peers.  Also, there are scenarios where organisations don't run their own peer(s).  In both of these cases, running a standalone gateway will be advantageous.

# Rationale and alternatives
[alternatives]: #alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

The Gateway SDK has been available as the high level programming model in the Fabric SDKs since v1.4.  It has been thoroughly tested in many scenarios.  This Fabric Gateway component is an embodiment of that programming model in a server component.

# Testing
[testing]: #testing

In addition to the usual unit tests for all functions, a comprehensive end to end 
scenario test suite will be created from the outset.  A set of 'cucumber' BDD style 
tests will be taken from the existing Java and Node SDKs to test all the major functions
of the gateway against a multi-org multi-peer network.  These tests will include:
- evaluation and submission of transactions
- event handling scenarios
- transient / private data
- error handling

The CI pipeline will run the unit tests for the Gateway and its SDKs as well as the end-to-end scenario (cucumber) tests against all SDKs.

# Dependencies
[dependencies]: #dependencies

- This will depend on the peer implementation in `hyperledger/fabric` and the protobuf definitions in `hyperledger/fabric-protos` repo.

# Unresolved questions
[unresolved]: #unresolved-questions

### Client inspection of proposal responses

Fabric is designed such that the client should have the opportunity to inspect the transaction response payload and then make a decision on whether or not that transaction should be sent to the orderer to be committed.

There is the possibility of a client being able to send proposals that are subsequently not submitted to the orderer and recorded on the ledger, either because:
- The proposal is not successfully endorsed; or
- The client actively chooses not to submit a successfully endorsed transaction.

This is viewed by some as a potential security vulnerability as it allows an attacker to probe the system.

Fabric Gateway should provide the capability for some kind of audit of submitted proposals, including their transaction ID so they can be reconciled against committed transactions.
