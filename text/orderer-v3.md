---
layout: default
title: Fabric ordering service consensus agnostic framework 
nav_order: 3
---

- Feature Name: A consensus agnostic framework for building a Fabric ordering service  
- Start Date: (fill me in with today's date, 2022-03-01)
- RFC PR: (leave this empty)
- Fabric Component: orderer
- Fabric Issue: (leave this empty)


# Table of Contents
1. [Summary](#summary)
2. [Motivation](#motivation)
3. [Ordering service framework building blocks](#identifying-building-blocks)
4. [Ordering service node construction](#connecting-the-pieces-together)
5. [Orderer to orderer communication and authentication](#orderer-to-orderer-authentication-and-communication)
6. [Testing](#testing)

# Summary
[summary]: #summary

The Fabric ordering service node is designed to support multiple consensus protocols through an abstraction of consensus plugins.
Consensus plugins order transactions into blocks on behalf of the Fabric ordering node, and the latter provides consensus plugins with 
networking, storage and cryptography software infrastructure.
Unfortunately, the consensus plugin approach not only makes it hard to integrate new types of consensus protocols into the Fabric ordering 
node, but also induces un-needed semantic dependency of the rest of the orderer codebase on the consensus plugins.  

This RFC proposes a paradigm for structuring the ordering service node that can be seen as the opposite one to the consensus plugin one.
Simply put, instead of having the ordering service node contain multiple consensus protocol implementations, it proposes to decouple 
the ordering service node into discrete software packages and then to have the consensus plugins tie them together, forming ordering 
service node binaries per consensus protocol. 

Additionally, the RFC discusses the current approach for orderer to orderer authentication and proposes an alternative design
which is more friendly to consume and deploy.

This RFC aims to:

- Define building blocks that consensus protocols may require to be integrated to Fabric.
- Outline how the aforementioned building blocks can be used to construct Fabric ordering service nodes.
- Design a more consumer friendly orderer to orderer authentication scheme.

# Motivation
[motivation]: #motivation


The Hyperledger Fabric ordering service is designed to support various types of consensus protocols.
The support for each consensus protocol is manifested in the form of a "plugin", a package that wraps around a consensus protocol implementation
and connects the rest of the ordering service code base to the consensus protocol.

The Fabric orderer codebase currently embeds the following consensus plugins:

- **Solo**: A single node orderer, useful for development, testing and proof of concepts.
- **Kafka**: Several ordering nodes relay transactions into an [Apache Kafka](https://kafka.apache.org/) system,
  pull transactions back in the same order and then cut them into identical blocks via synchronization messages also sent through Kafka.
- **Raft**: Several ordering nodes run the [Raft](https://raft.github.io/) protocol by embedding the [Raft library of etcd](https://github.com/etcd-io/etcd/tree/main/raft).

Apart from the official Hyperledger Fabric consensus implementation, there is also a [Fabric fork](https://github.com/SmartBFT-Go/fabric) of a [BFT library](https://github.com/SmartBFT-Go/consensus) called SmartBFT.

A BFT consensus protocol might have a different approach for onboarding and state snapshot replication compared to a CFT consensus protocol.
In particular, a state snapshot sent between nodes in CFT is always assumed to be trusted, while in BFT it is not the case. 
While it is tempting to have common code that performs state snapshot transfer that caters to both adversarial models, 
such code is most likely to be complex and hard to follow and test.

Each orderer type requires Fabric to understand how to initialize instances of it from the consensus metadata in the configuration blocks.
However, this makes it impossible for external contributors to use their own consensus implementation without either forking Fabric entirely, 
or making intrusive changes in the official Fabric orderer and peer code base.

Undoubtedly, an alternative architecture which doesn't require Fabric codebase changes to add support for new 
consensus protocols would be a sought-after property.

# Identifying building blocks
[building-blocks]: #identifying-building-blocks

Apart from consensus, which agrees on orders of transaction, the orderer contains infrastructure that manages various aspects of its lifecycle and operations:

 - **Chain management**: Spawns instances of the consensus protocols per channel and dispatches messages and operations to the right 
   consensus instance.
 - **Block replication service**: Server-side API that sends blocks upon request. 
 - **Block synchronization utility**: Pulls blocks from other ordering service nodes and appends them to the local ledger. 
 - **Communication service**: Authenticates client connections, receives messages from remote nodes and sends message to remote nodes.
 - **Ledger subsystem**: Appends blocks to persistent storage and retrieves blocks by their index.
 - **Channel participation service**: Manages local addition and removal of ledgers per channel.
 - **Transaction filtering**: Enforces access control and validity checks on transactions both received from clients or from nodes.
 - **Membership Service Provider subsystem**: Classifies identities into principals and drives policy authorization. 

In this RFC, it is proposed that these building blocks be made consensus independent, and the building blocks that cannot be consensus independent will be 
part of the consensus plugin that wraps around the consensus independent building blocks. 

Therefore, the new organization of the components in the new ordering service node of a given type will look as follows:

```
         ____________________________________________
        |                                            |
        | chain management--------consensus instance |
        |       |      |                             |
        |       |      -----------consensus instance |
        |       |                      |             |
        |       |_______               |             |
        |       ________|______________|____         |                         
        |       |                          |         |
        |       |Ordering service framework|         |                         
        |       |           (B)            |         | 
        |       |__________________________|         |    
        |                                            |
        |         ordering service node        (A)   |
        |____________________________________________|
```

The ordering service framework (denoted **B**) refers to all building blocks above which are consensus independent.
Wrapping around the components of the ordering service framework is a layer (denoted **A**) which is consensus specific 
and includes the chain management, and the consensus instances (one per channel).



# Connecting the pieces together
[connecting]: #connecting-the-pieces-together

Now, having common and consensus independent building blocks that perform various functions,
an ordering service for a given consensus protocol can be implemented without changing Fabric codebase. 

The developers who implement the ordering service node will just import the needed packages from the Fabric ordering service framework,
and will implement a binary that services the [Atomic Broadcast gRPC API](https://github.com/hyperledger/fabric-protos/blob/main/orderer/ab.proto#L80-L86).

However, how can we ensure that peers can successfully consume the blocks produced by the ordering service node 
which they have no knowledge about its underlying consensus protocol?

To that end, a [separate RFC](https://github.com/hyperledger/fabric-rfcs/pull/48) which deals with block consumption by peers (and other orderers) was created.
That RFC defines how peers: 
 - Identify from which ordering service node to pull blocks from and how to detect block withholding.
 - Know how many signatures to verify for each block.


### The benefits of the proposed approach 
**Ease of integration**: Once peers have a consensus independent mechanism to pulls blocks and verify them, an addition of an ordering service node type does not affect the Fabric core codebase.
In fact, by making the Fabric ordering service framework "import friendly" from external repositories,
the various development teams of the open source community could implement their own consensus protocols that 
fits their setting and integration with Fabric peers would not require changes in the Fabric codebase.

**Proper responsibility for support**: One of the inhibitors for integrating new consensus protocols and libraries into Fabric 
is that troubleshooting an issue requires knowledge of the consensus protocol and its implementation.
Since with the proposed approach a new consensus protocol can be introduced to Hyperedger Fabric without influencing the codebase of the Fabric core,
it will not be expected nor required from the Fabric core maintainers to troubleshoot or maintain ordering service implementations 
that are not part of the official Fabric core repositories. This fact could expedite the emergence of new scientific and industrial consensus implementations 
for Fabric.

# Orderer-to-orderer authentication and communication
[communication]: #orderer-to-orderer-authentication-and-communication

### gRPC API changes

The current [gRPC API](https://github.com/hyperledger/fabric-protos/blob/release-2.4/orderer/cluster.proto) that orderers use to communicate, declares the channel each message is routed to in the message itself:
```
// ConsensusRequest is a consensus specific message sent to a cluster member.
message ConsensusRequest {
    string channel = 1;
    bytes payload = 2;
    bytes metadata = 3;
}
```

In the implementation of the communication API, gRPC streams never share channels and a message for channel *foo* will never be sent through 
the same gRPC stream of channel *bar* and vice versa. 

The implemented approach is inefficient as it makes the channel name to be sent over and over redundantly.
We shall define a gRPC API that is similar to the existing one, but with the channel name sent only during authentication (the `Auth` message will contain the channel name):

```
// OrdererMessaging defines communication between orderers.
service OrdererMessaging {
    // Connect estabishes a one way stream between orderers
    rpc Connect(stream Request) returns (Response);
}


// Request wraps a message that is sent to an orderer.
message Request {
    oneof payload {
        // consensus_request is a consensus specific message.
        ConsensusRequest consensus_request = 2;
        // submit_request is a relay of a transaction.
        SubmitRequest submit_request = 3;
        // Auth authenticates the orderer that initiated the stream.
        Auth auth = 4;
    }
}

message Auth {
 ...
 string channel = 7;
 ...
}

```

The `ConsensusRequest` and `SubmitRequest` will be defined similarly as the [v2.x gRPC API](https://github.com/hyperledger/fabric-protos/blob/release-2.4/orderer/cluster.proto),
but without channels in them. The channel will be only encoded in the `Auth` message. The first message that is expected to be sent via
the gRPC stream is an `Auth` message, and the rest are either `ConsensusRequest` or `SubmitRequest` messages.



### The current authentication mechanism: Mutual TLS with TLS pinning 

The Raft consensus library uses numerical identifiers to represent nodes taking part in the consensus protocol.
When the Fabric runtime receives a request from the Raft library to send a message to a remote node, 
it establishes a gRPC stream over mutual TLS with the remote node, and sends the message over the stream.
Similarly, when a message is received from a remote node to a Raft orderer, the Fabric runtime locates the numerical identifier
of the remote node that corresponds to the TLS certificate extracted from the TLS connection and then passes the message tagged with the numerical identifier.
The conversion between the TLS certificate to the numerical identifier is done via a mapping that is found in the configuration blocks.

Clearly, it is imperative that unauthorized parties will not be able to impersonate ordering service nodes.
When performing in a Byzantine setting where malicious orderers might attempt impersonation, it is vital to authenticate which node
sent the message. While it is possible to have each orderer sign every message it sends, it is inefficient.

The current authentication mechanism of Fabric orderer-to-orderer communication relies on mutual TLS.
More specifically, whenever an orderer wants to send a message to another orderer, it establishes a TLS connection to the remote orderer and the remote orderer identifies
the connection initiator by its TLS certificate. This is a rather simple approach and its security entirely draws upon the native Go TLS runtime.

```
 __________       <orderer1.crt>      ___________
|          |    ------------------>   |          |
| orderer1 |    <------------------   | orderer2 |
|          |      <orderer2.crt>      |          |
|__________|                          |__________|
```

However, this approach has several drawbacks: 
1. **TLS keys might not be stored in HSM**: The Go TLS stack doesn't provide HSM integration capabilities, and therefore TLS private keys of orderer nodes are more susceptible for theft. 
A theft of a TLS private key means that an adversary may send messages on behalf of the victim node, which is catastrophic in a crash fault tolerant (CFT) setting.
Although it is possible to override the `PrivateKey` reference of the `tls.Certifciate` with a signer implementation that communicates with an HSM, existing users which already have key management systems 
   built will be reluctant to make the required changes.
   


2. **Mutual TLS authentication is challenging when combined with reverse proxies**: In case the organization hosting the orderer has a TLS terminating reverse proxy that routes ingress traffic into the organization,
it is challenging for that organization to host an ordering service node that relies on mutual TLS authentication.

3. **TLS certificates are often issued by commercial CAs, not internal CAs**: When a certificate is issued by a commercial CA, its lifespan might be significantly shorter. 
One of the benefits of having certificates issued by a commercial CA, is that it doesn't require clients to be configured with root TLS CA certificates, since the operating system's TLS CA certificate pool is used instead.
However, commercial CAs such as "Let's Encrypt" issue short term TLS certificates. Although a channel config update is not required when preserving the old public key, this option is left to the mercy of the commercial TLS CA API and implementation
   and is not always available.


### The proposed alternative authentication mechanism: Enrollment certificate handshake based authentication

Fabric has two separate certificate infrastructures: The enrollment certificate infrastructure, which governs signatures and access control, and the TLS infrastructure which governs authentication of TLS sessions.

Instead of using TLS certificates, we will use enrollment certificates for authentication. Namely, we will have a mapping of enrollment certificates to numerical identifiers in the configuration blocks.
The authentication protocol will consist of a single message that authenticates the initiator node to the responder node.
We do not need to authenticate the responder node to the initiator node, because the gRPC stream is one way.
````
  Iniatiator                             Responder
_____________                          ____________
                            
              --------[Auth]------>

````

**Auth**: The content of the `Auth` message is as follows:

```
message Auth {
  uint32 version = 1;
  bytes signature = 2;
  Timestamp timestamp = 3;
  uint64 from_id = 4;
  uint64 to_id = 5;
  bytes session_binding = 6;
  string channel = 7;
}
```

The `from_id` is the numerical identifier of the initiator of the connection. It is needed for the responder to identify from which node 
the subsequent messages received after the `Auth` message are sent from.

The `to_id` is the numerical identifier of the node that is being connected to. It is needed in order for the request to not be re-used for other nodes.
Without `to_id`, the responder node can authenticate to other nodes on behalf of the initiator node.

The `timestamp` is a timestamp that indicates the freshness of the request, and it needs to be within a margin of the responder's local time.

The `signature` is the signature that is verifiable under the initiator's public key over the ASN1 encoding of `<version, timestamp, from_id, to_id, session_id, channel>`.

The `version` field determines what fields the signature was computed over, and the current version is zero. 

The `session_binding` prevents MITM (Man In The Middle) attacks, and it is explained next.


### Protecting against Man-In-The-Middle attacks

The safety of the protocol described in the previous paragraph cannot be guaranteed in case the transport layer is hijacked by an adversary. 
Consider a scenario where an adversary obtains the private key that corresponds to the TLS certificate of the responder node. 
The adversary can hijack the network traffic between the initiator and the responder, wait for the `Auth` message, and then impersonate the 
initiator before the responder and impersonate the responder before the initiator by relaying the `Auth` through itself.

As noted earlier, TLS certificate private keys are more susceptible for theft as they might not be stored in HSM.
In such a case, it is still possible to detect a MITM attack by taking advantage of [Keying Material Exporters](https://datatracker.ietf.org/doc/html/rfc5705).
The [ExportKeyingMaterial](https://pkg.go.dev/crypto/tls#ConnectionState.ExportKeyingMaterial) API allows both ends of a TLS connection to derive a shared key from the
`client_random`, `server_random`, `master_secret` used to establish the TLS session. Since this key is identical to the initiator and responder, 
it can be used by the responder to detect whether the message is being relayed by an attacker. 

The detection is quite simple: The initiator node and the receiver node both compute: `r = Hash(version || timestamp || from_id || to_id, channel)`
and then `session_binding = ExportKeyingMaterial(r)`. The initiator node encodes `session_binding` in the `Auth` message, and the responder node 
expects the `session_binding` from the `Auth` message to match what it computed on its own.

In case an attacker manages to obtain the TLS private key of the responder node and performs a MITM attack, the function `ExportKeyingMaterial`
on the initiator node and the real responder node would yield different values, because the `master_secret` will differ.
The `master_secret` will differ because Fabric orderers will be restricted to use ECDHE key exchange. With ECHDE key exchange, a MITM with the same `master_secret` 
across both the initiator and responder means that the initiator and responder each picked the public key share, but the attacker managed to compute the shared key,
thus breaking the [CDH assumption](https://en.wikipedia.org/wiki/Computational_Diffie%E2%80%93Hellman_assumption).

In case the responder node is located behind a TLS terminating reverse proxy, it can be configured to ignore the `session_binding` check, 
but the check will be enabled by default.

# Testing
[testing]: #testing

All new components build will be designed to be testable and stable, and will have unit tests with good code coverage.
As this RFC deals with building a framework, there are no integration tests to be built, but only unit tests.