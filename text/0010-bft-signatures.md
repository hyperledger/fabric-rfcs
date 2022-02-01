---
layout: default
title: Fabric core block retrieval and signature verification
nav_order: 3
---

- Feature Name: Fabric core block retrieval and signature verification
- Start Date: (fill me in with today's date, 2022-02-01)
- RFC PR: (leave this empty)
- Fabric Component: peer, orderer
- Fabric Issue: (leave this empty)


# Table of Contents
1. [Summary](#summary)
2. [Motivation](#motivation)
3. [Existing Approaches](#existing-approaches-for-block-replication-and-block-signature-verification)   
4. [Proposed Techniques](#proposed-techniques)
5. [Implementation Plan](#implementation-plan)
6. [Testing](#testing)

# Summary
[summary]: #summary

The Hyperledger Fabric core (orderers and peers) are only crash fault tolerant, and cannot handle arbitrary behavior of its ordering service nodes.


The current Fabric peers and orderers are implemented to retrieve blocks from a single orderer
and there is currently no design of how blocks are to be retrieved in the Byzantine setting or how signatures on blocks should be verified.



Block retrieval from Byzantine orderers needs to tackle denial of service from malicious orderers.

The challenge with block signature validation in the Byzantine setting is avoiding the linear space overhead in the number of certificates that need to be stored as part of the signature information.

While forks of Fabric with a Byzantine Fault Tolerant ordering service [exist](https://github.com/SmartBFT-Go/fabric/), they have not been incorporated into the official Hyperledger Fabric project.

This RFC aims to:

- Give an overview of how denial of service resistant block retrieval and block signature validation is done in SmartBFT.
- Define how denial of service resistant block retrieval should be conducted in Hyperledger Fabric.
- Define how Byzantine block signature validation should be constructed and encoded in Hyperledger Fabric.

# Motivation
[motivation]: #motivation

Currently, Hyperledger Fabric only supports ordering service types that are Crash Fault Tolerant (CFT), not Byzantine Fault Tolerant (BFT).

There are two main attacks that are considered outside of the crash fault tolerant thread model, but must be protected against in the Byzantine fault tolerant setting:

**Block withholding**: A Byzantine orderer might be reluctant to send newly formed blocks to peers that connect to it, either because of malicious intent or to save egress network bandwidth charges from its hosting provider.
 Fabric architecture relies on peers receiving newly formed blocks as fast as possible, because otherwise client transaction submission times out and smart contract simulation results in stale reads.

**Block forgery**: A Byzantine orderer might try to forge a block and send it to peers or other ordering nodes that replicate from it, in order to alter the smart contract execution on peers by subterfuge.
 One way of doing this, is to omit a transaction from a block, or re-order transactions within a block.
 
To circumvent block forgery attacks, the following invariant should be preserved: *A set of signatures on a block should only be considered valid if it is impossible for the parties running the protocol to construct a set of signatures on a different block that is also considered valid*.

The aforementioned invariant can be implemented in different ways, such as requiring a **threshold of signatures** from different orderers on a block, or requiring a single **threshold signature** on a block.
The threshold itself can vary depending on the consensus algorithm and on the stage in it that the signatures is produced: Signing a block post consensus requires signatures from only `f+1` orderers (at the expense of latency), while signing during consensus requires signatures from `2f+1` orderers (at the expense of size and computation).


# Naive approaches for block replication and block signature verification

### Block replication

In a Byzantine setting, there is a fundamental assumption that there is an upper bound on the number of malicious or failed nodes
 among all the nodes that run the protocol, and that upper bound is denoted as `f`.
 In BFT, the ordering service is comprised of `n>3f` for some `f>0`, so we have up to `f` nodes which are either offline, unreachable or malicious,
 and `2f+1` nodes that are online, reachable and follow the protocol.

An easy way of ensuring malicious orderers do not refrain from sending newly formed blocks is to simply pull blocks from `f+1` distinct orderers.
This ensures that even if we happen to connect to all `f` malicious orderers that will not send us blocks, the remaining orderer will, since it is honest.

Obviously, such an approach increases the network bandwidth utilization of the orderers by a factor of `f` which is linear in the number of total ordering service nodes that run the consensus protocol.

### Block signature verification 

A Hyperledger Fabric block is designed to carry an arbitrary number of signatures in its metadata section:

```
message MetadataSignature {
    bytes signature_header = 1; // An encoded SignatureHeader
    bytes signature = 2;       // The signature over the concatenation of the Metadata value bytes, signatureHeader, and block header
}
```

The signature header contains the identity of the signer which carries an x509 certificate and the MSP identity that one of its certificate authorities issued that certificate:

```
message SignatureHeader {
    // Creator of the message, a marshaled msp.SerializedIdentity
    bytes creator = 1;

    // Arbitrary number that may only be used once. Can be used to detect replay attacks.
    bytes nonce = 2;
}
```

This means that when signed by `2f+1` (or `f+1`) orderers, each block contains a number of signatures that is linear in the number of ordering service nodes that participate in the consensus protocol.
When considering an ordering service that can tolerat two failures (`f=2`, hence `n=7`) this means that each block carries  3.5KB of x509 certificates,
even if the block itself contains only a single transaction (that is often 3.5KB in size itself).

Clearly, encoding identities each time in the signature header of each orderer signature is wasteful and therefore we should strive to eliminate this overhead.

## Existing approaches for block replication and block signature verification
[Existing approaches]: #existing-approaches-for-block-replication-and-block-signature-verification

### Attestation for block existence

An approach for block denial of service resistance taken by [SmartBFT](https://arxiv.org/abs/2107.06922) is to connect to (at least)`f+1` ordering service nodes, fetch blocks only from one of them and fetch attestations that blocks exist from the remaining `f` (or more).
If a block existence attestation for a sequence **i** is found valid but the orderer designated to send blocks withholds block **i** for too long, the role assignment of block source and attestation for block existence source is re-shuffled.

An attestation for block existence is simply a regular block (along with its `2f+1` signatures) pruned from its transactions. The signatures can still be verified despite the transactions being pruned from the block thanks
to the fact that in Fabric the signing algorithm is applied on the ASN1 encoding of the header block which carries the hash of the concatenation of all transactions:

```
message BlockHeader {
    uint64 number = 1; // The position in the blockchain
    bytes previous_hash = 2; // The hash of the previous block header
    bytes data_hash = 3; // The hash of the BlockData
}
```

Let's consider the two possible outcomes based on the orderer that is selected for block retrieval:

1. If orderer that blocks are pulled from is withholding newly formed blocks, then there is at least one honest orderer that will not withhold attestations of block existence. Hence, a re-shuffling of role assignment will occur until an honest orderer will be the block source.
2. Else, the orderer that blocks are pulled from is not withholding newly formed blocks. However, it is possible that new blocks are not formed due to lack of client activity. It is imperative to ensure that malicious ordering nodes cannot make honest ordering nodes be falsely suspected of block withholding.
As each "block existence attestation" contains `2f+1` signatures it is impossible to forge one. Hence, if a node sends a attestation for block existence, it means this a block with such sequence number was indeed appended to the blockchain.

In order for a peer or an orderer to convey the intent of receiving only blocks without transactions, 
the [SeekInfo](https://github.com/hyperledger/fabric-protos/blob/main/orderer/ab.proto#L45) message is expanded to contain also a `SeekContentType` field:

```
  // SeekContentType indicates what type of content to deliver in response to a request. If BLOCK is specified,
  // the orderer will stream blocks back to the peer. This is the default behavior. If HEADER is  specified, the
  // orderer will stream only a the header and the signature, and the payload field will be set to nil. This allows
  // the requester to ascertain that the respective signed block exists in the orderer (or cluster of orderers).
  enum SeekContentType {
      BLOCK = 0;
      HEADER_WITH_SIG =1;
   }
```

By making the default value be `BLOCK` the API is backward compatible to old consumers.

Indeed, the SmartBFT's approach for Byzantine fault tolerant block replication overcomes the linear overhead of the naive approach. However, a malicious ordering node can still withhold a block up until right before the suspicion timeout expires at the party that pulls the block.
Hence, while a malicious orderer may not withhold blocks indefinitely, it may slow down the replication of its consumers.

Another minor inconvenience is that the peer admin needs to configure the peer correctly to use the Byzantine fault tolerant implementation instead of the regular one.

### Signature identity referencing

In SmartBFT, like Raft, each configuration block contains a set of consenter (ordering service consensus protocol participant) definitions. 
Each such consenter definition contains various information that is used either to connect or to authenticate the consenter:

```
message Consenter {
    uint64 consenter_id = 1;
    string host = 2;
    uint32 port = 3;
    string msp_id = 4;
    bytes identity = 5;
    bytes client_tls_cert = 6;
    bytes server_tls_cert = 7;
}
```

When validating a signature signed by orderer with numerical identifier **j**, the mapping between **j** and the matching identity (which carries the public key) is taken from the latest configuration block.

Recall, the naive approach to encode several signatures on a block in its metadata involves encoding an amount of orderer identities that is linear in the number of ordering nodes.

To that end, SmartBFT substitutes the identity (over 700B in size) with its consensus node identifier (a 64 bit number). 
As a result, the `MetadataSignature` now contains three additional fields:

```
message MetadataSignature {
    bytes signature_header = 1; // An encoded SignatureHeader
    bytes signature = 2;       // The signature over the concatenation of the Metadata value bytes, signatureHeader, and block header
    uint64 signer_id = 3;
    bytes nonce = 4;
}
```

When a Fabric node validates a signature on a block, an ephemeral `SignatureHeader` is constructed where the `nonce` is taken from the
`MetadataSignature` and the `creator` is the `identity` matching the `signer_id`. 

This technique reduces the aforementioned overhead of 3.5KB to a meager 40 bytes.


While the approach of the SmartBFT library integration works, it creates a coupling between the consensus implementation and the rest of Fabric, and this limits incorporation of other consensus implementations into Fabric.
 
More specifically, it requires peers to understand how to decode the consensus metadata of ordering service nodes. That means that the peer needs to be aware of the type of ordering service it is working with, which breaks the abstraction of the ordering service as a service that receives as input transactions, and outputs blocks.


# Proposed techniques
[technique]: #proposed-technique

The proposed approach is heavily inspired by SmartBFT but is designed to be consensus agnostic, minimize implementation and operational complexity.

### Block replication

The proposed technique for block replication is essentially the approach of SmartBFT where a re-shuffling takes place not only upon suspicion of the block source but also in a timely fashion in order to be resilient to overloaded or deliberately slow orderers.   
In order to save network bandwidth and processing power when the network is idle, the automatic time-based re-shuffling will only take place if new blocks are received.
Moreover, there will not be a **need** to specify in the peer's local configuration (or an orderer's local configuration) which of the implementations to use. 
Notice that a block validation policy that mandates signatures from multiple ordering nodes, is **never** satisfied by **only** principal sets of size one. 
Or, equivalently, a block validation policy that is satisfied by **only** principal sets of size one never requires signatures from multiple ordering nodes. 
Evidently, policies of the former type are **always** BFT policies, while policies of the latter type might be either a crash fault policy or a BFT policy that utilizes a threshold signature scheme.

Therefore, Fabric nodes will pick the BFT implementation if either the block validation policy implies the need, or if an override is activated in the node's local configuration.  


### Block signature verification

The proposed technique for block signature verification should be generic and consensus agnostic, in order to facilitate incorporation of multiple consensus types without the need for Fabric nodes to understand which consensus implementation is employed.

Apart from the already existing block validation policy:

```
# BlockValidation specifies what signatures must be included in the block
# from the orderer for the peer to validate it.
BlockValidation:
    Type: ImplicitMeta
    Rule: "ANY Writers"
```

The `Orderer` section in the channel config will contain a mapping from numerical identifiers to tuples of serialized identities:

```
Orderer: &OrdererDefaults
    ...
    ConsenterMapping:
    - Id: 1
      MSPID: OrdererOrg1
      Identity: <base64 encoding of raw identity>
    - Id: 2
      MSPID: OrdererOrg2
      Identity: <base64 encoding of raw identity>
      ...
```

Upon a configuration update, it will be up to the (Fabric side of the) consensus implementation to validate that:

- The block validation policy matches the required threshold (e.g `2f+1`) of the consensus protocol.
- The `ConsenterMapping` is not in conflict with the opaque consensus metadata.

The problem of correctly composing the consenter mapping and block validation policy at bootstrapping is considered out of scope of this RFC,
as this is consensus specific and can be addressed with custom tooling, such as `configtxgen` enhancements or any other tooling.


The `MetadataSignature` shall be expanded to contain a `uint32` field of `signer_id` and the `bytes` field of the nonce` as in SmartBFT.
Accordingly, the procedure to build a signed data element from a `MetadataSignature` will first look if the `signature_header` is not empty, and if so it will proceed to regular processing.
Otherwise, if the `signature_header` is empty, an ephemeral one will be reconstructed from the `nonce`, `signer_id` and the latest channel config and then procesing can proceed as formerly mentioned.

# Implementation plan
[implementation]: #implementation-plan

The following components are to be changed:

- **Channel configuration**: Addition of `ConsenterMapping` to the channel configuration (gated behind a V3.0 [capability](https://hyperledger-fabric.readthedocs.io/en/latest/capabilities_concept.html)).
- **Peer and orderer local configuration**: Addition of BFT or CFT block replication toggle (automatic, BFT, CFT).
- **Orderer block replication primitive**: The `BlockPuller` is an [abstraction](https://github.com/hyperledger/fabric/blob/main/orderer/consensus/etcdraft/chain.go#L83-L87) that is given to leader based consensus plugins 
and is backed by underlying code in the `orderer/common/cluster` package. The abstraction can remain as-is, but its underlying implementation
  would need to change in order to be able to pull attestations of block existence (blocks without transactions). 
  Similarly,the same underlying implementation is used when onboarding orderers.
- **Peer block replication primitive**: The peer currently contains an [implementation](https://github.com/hyperledger/fabric/blob/main/internal/pkg/peer/blocksprovider/blocksprovider.go) that fetches blocks from a single orderer.
The implementation would need to be changed accordingly to protect against block denial of service attacks.
- **Common deliver service**: The [common deliver service](https://github.com/hyperledger/fabric/blob/main/common/deliver/deliver.go), used by both peers and orderers when sending blocks to subscribers, will be enhanced by an ability to 
redact the transactions from a block prior to its dispatch based on the intent derived from the `SeekInfo` message as done in SmartBFT.

This RFC suggests to build a re-usable component for block pulling that will be used by both the peer and the orderer, and it will have a simple API as the [BlockPuller](https://github.com/hyperledger/fabric/blob/main/orderer/common/cluster/deliver.go#L30-L55):

```
// BlockFetcher is used to pull blocks from ordering service nodes
type BlockFetcher interface {
	PullBlock(seq uint64) *common.Block
	Close()
}
```

The `BlockFetcher` component will perform all bookkeeping required for both CFT and BFT use cases and will then be incorporated in the orderer and peer block replication code.
This will let the project get rid of multiple implementations that perform the same thing, as currently there is a duplicity in the form of the peer deliver client and the orderer's cluster infrastructure.

# Testing
[testing]: #testing

All new components build will be designed to be testable and stable, and will have unit tests with good code coverage.
Additionally, until we have a Byzantine Fault Tolerant orderer up and running, integration tests will be made that
artificially sign Fabric blocks and existing peers will pull them from an artificial endpoint, in order to test the block signature validation code works properly.
This will also demonstrate the consensus agnosticism of the approach (since no real orderer nor a consensus protocol will be deployed in the test).