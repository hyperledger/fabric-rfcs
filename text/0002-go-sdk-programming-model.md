---
layout: default
title: Implement Fabric programming model in the Go SDK
parent: RFCs
---

- Feature Name: Implement Fabric programming model in the Go SDK
- Start Date: 2019-11-21
- RFC PR: (leave this empty)
- Fabric Component: fabric-sdk-go
- Fabric Issue: (leave this empty)

# Summary
[summary]: #summary

This feature will implement the new Fabric programming model for the Go SDK.  The new Fabric programming model provides a set of higher level abstractions designed to enable developers to easily write smart contracts and client applications with minimal boilerplate code.  

# Motivation
[motivation]: #motivation

The motivation of this work is to provide a consistent set of concepts and APIs across all of the SDKs supported by Fabric.  The ideal outcome is that the Go SDK become an 'officially supported' SDK alongside the Node and Java SDKs, and included in the official Fabric docs and samples.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Fabric documentation (v1.4 onwards) contains a chapter 'Developing Applications' which describes how to develop smart contracts and client applications using the new programming model.  The chapter illustrates the concepts using the example of a commercial paper, and gives code examples as it guides the user through the development steps.  This documentation, and related samples, currently applies to the Node and Java SDKs and chaincode contract APIs.  This RFC describes the implementation of this in the Go SDK and is related to [RFC 0001](https://github.com/hyperledger/fabric-rfcs/blob/master/text/0001-chaincode-go-new-programming-model.md) which describes the Go implementation of the chaincode contract API.

Note that this new programming model does not break any existing APIs.  Client applications written using these APIs will continue to work unchanged.

As described in the documention for [client applications](https://hyperledger-fabric.readthedocs.io/en/release-1.4/developapps/application.html), the following structures are available:

### Wallet
The wallet is used to manage the storage of identities for use within the client application.  The SDK provides two wallet implementations out of the box:
- File system wallet.  Stores the identity on the local file system.  The format used is compatible with the Node and Java SDKs, and can be used interchangably.
- In-memory wallet.  Provided primarily for testing purposes.

Developers can create their own wallet implementations using the interface (and samples) provided if they wish to use custom storage mechanisms

### Gateway
Logical concept representing the connection point a client makes to the Fabric network.  The user configures the gateway using a network config (json/yaml) and an identity from the wallet.  Other options can also be configured on the gateway in a declarative manner.

### Network channel

Represents a channel available to the gateway peer.

### Contract

Represents an instance of a smart contract available on a network channel.  It has two primary methods:

- `EvaluateTransaction()` - Invokes a transaction function in the smart contract (chaincode).
- `SubmitTransaction()` - Invokes a transaction function in the smart contract and commits its RW-set to the ledger, subject to consensus.

### Transaction

Enables finer control of transaction parameters (e.g. transient data).


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Given the concepts and structures introduced in the previous section, a client application written to this programming model would have the following general structure (note this has been derived from an early prototype):

```
package main

import (
    "fmt"
    "github.com/hyperledger/fabric-sdk-go/pkg/gateway"
)

func main() {
    wallet := gateway.NewInMemoryWallet()
    wallet.Put("user", gateway.NewX509Identity(testCert, testPrivKey))

    gw, err := gateway.Connect("connection-tls.json", 
                    gateway.WithIdentity(wallet, "user"),
                    gateway.WithDiscovery(false),
                    gateway.WithCommitHandler(gateway.DefaultCommitHandlers.OrgAny))

    network, err := gw.GetNetwork("mychannel")

    if err != nil {
        fmt.Printf("Failed to get network: %s", err)
        return
    }

    contract := network.GetContract("fabcar")
    
    result, err := contract.EvaluateTransaction("queryAllCars")

    if err != nil {
        fmt.Printf("Failed to evaluate transaction: %s", err)
        return
    }
    fmt.Println(result)

    result, err = contract.SubmitTransaction("createCar", "CAR10", "VW", "Polo", "Grey", "Mary")

    if err != nil {
        fmt.Printf("Failed to submit transaction: %s", err)
        return
    }
    fmt.Println(result)

    // the user might prefer a 'helper' method to reduce boilerplate error handling
    handle(contract.EvaluateTransaction("queryCar", "CAR10"))

    // create a Transaction object to pass transient data or override endorsers
    transient := make(map[string][]byte)
    transient["price"] = []byte("8500")
    txn, err := contract.CreateTransaction("changeCarOwner", gateway.WithTransient(transient))
    result, _ = txn.Submit("CAR10", "Archie")

    handle(contract.EvaluateTransaction("queryCar", "CAR10"))
}

func handle(result string, err error) {
    if err != nil {
        panic(err)
    }
    fmt.Println(result)
}
```

### Wallet
The wallet interface provides the following methods:
- `Put(label, identity)`
- `Get(label)`
- `Exists(label)`
- `List()`
- `Remove(label)`

### Gateway
The gateway is configured and constructed using a set of options, in common with the Java and Node SDKs.  The `Connect()` function accepts functional option arguments which is idiomatic in Go and consistent with the existing Go SDK. The GatewayOption functions are available:
- `WithIdentity(wallet, label)` - select an identity from a wallet to associate with the gateway
- `WithCommitHandler(handler)` - override the default commit handler strategy
- `WithQueryHandler(handler)` - override the default query handler strategy
- `WithDiscovery(enabled)` - Enables (default) or disables service discovery
- `Connect(networkConfig, ...GatewayOption)` - instantiate the Gateway with the the path to network config (json/yaml) plus any of the above options

The Gateway interface has the following method:
- `GetNetwork(name)` - returns an initialised network channel

### Network
The Network interface has the following method:
- `GetContract(name)` - returns a reference to the named contract in the chaincode

### Contract
The Contract interface has the following methods:
- `EvaluateTransaction(name, ...args)`
- `SubmitTransaction(name, ...args)`
- `CreateTransaction(name, ...TransactionOption)`

The `EvaluateTransaction` method runs the transaction (proposal) function in the smart contract and returns whatever value that function returns.  The peer(s) that gets targetted is controlled by the QueryHandler.
The `SubmitTransaction` method sends the transaction proposal to endorsing peers (determined by the discovery service), collates the resonses and sends to the orderer.  This method awaits commit events from peers, as controlled by the CommitHandler, before returning to the client.
The `CreateTransaction` method creates a transaction object representing the named transaction function.  Additional information can be associated with the transaction request using the `TransactionOption` functional arguments before it is submitted.  The following options are supported:
- `WithTransient(transientMap)` - associates transient data with the transaction proposal
- `WithEndorsingPeers(...peer)` - targets specific peers for this transaction request, overriding the discovery service.  Useful for targetting private data.


#### CommitHandler
The commit handler is a pluggable mechanism that controls how many commit events should be received before the client can continue processing.  It is selected as an option on the Gateway. The following policies are provided out of the box:
- Wait for __all__ available peers in __your organisation__ to emit commit events – DEFAULT
- Wait for __first__ peer in __your organisation__ to emit commit event
- Wait for __all__ available peers in the __network channel__ to emit commit events 
- Wait for __first__ peer in the __network channel__ to emit commit event
Users can also implement their own commit handlers if they wish to have other behaviours.

#### QueryHandler
The query handler is a pluggable mechanism that controls which peers to target when evaluating a transaction function using `EvaluateTransaction()`. It is selected as an option on the Gateway. The following policies are provided out of the box:
- Stick to a __single peer__.  Move to another one if it stops responding – DEFAULT 
- __Round-robin__ between available peers within your organisation.
Users can also implement their own query handlers if they wish to have other behaviours.

### Transaction
- `Evaluate(...args)` - As with EvaluateTransaction() above, but with TransactionOptions (e.g. transient data).
- `Submit(...args)` - As with SubmitTransaction() above, but with TransactionOptions (e.g. transient data).

# Drawbacks
[drawbacks]: #drawbacks

It could be argued that this is not needed because all of its capabilities can already be done in the existing Go SDK.

This is being proposed not because of any underlying deficiencies in the existing APIs, but rather to provide a __consistent__ set of concepts and structures across __all__ of the __supported__ developer APIs in Fabric.  

# Rationale and alternatives
[alternatives]: #alternatives

The primary rationale is around consistency with the other SDKs and to align with the programming model as described in the official Fabric documentation.  Alternatives could always be proposed, but they would not necessarily be consistent with the other SDKs or the programming model.

# Prior art
[prior-art]: #prior-art

Significant experience and feedback has now been gained from user adoption of this programming model in Node and Java which have led to incremental improvements.

# Testing
[testing]: #testing

In common with the Node and Java SDKs, there is a strong emphasis on very high coverage of unit tests and scenario (integration) tests.  Appropriate, popular frameworks are adopted for each technology.

Common cucumber scripts (BDD tests) have been implemented across Node and Java SDKs.  It is intended to implement these for Go to enforce common behaviour across all SDKs.

# Dependencies
[dependencies]: #dependencies

This proposal builds (and depends) on the existing Go SDK as implemented in the `hyperledger/fabric-sdk-go` repo.

New client samples for Fabcar and Commercial Paper, implemented in Go, will be added to the `hyperledger/fabric-samples` repo.

This is also related to, but does not depend on, [RFC 0001](https://github.com/hyperledger/fabric-rfcs/blob/master/text/0001-chaincode-go-new-programming-model.md) which brings the new programming model to the Go chaincode API.

# Unresolved questions
[unresolved]: #unresolved-questions
