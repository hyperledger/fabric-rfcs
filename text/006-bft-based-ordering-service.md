
Byzantine Fault Tolerant ordering servic for Hyperledger Fabric
====================================================================


- Feature Name: (Byzantine Fault Tolerant ordering service)
- Start Date: (TBD)
- RFC PR: (leave this empty)
- Fabric Component: (mostly the orderering service)
- Fabric Issue: (leave this empty)


## Table of contents 

- [Summary](#summary)
- [Motivation](#motivation)
  * [Hyperledger Fabric and BFT consensus](#hyperledger-fabric-and-bft-consensus)
- [BFT Consensus Library](#bft-consensus-library)
  * [The protocols implemented by the library](#the-protocols-implemented-by-the-library)
  * [Embedding the library in a distributed application](#embedding-the-library-in-a-distributed-application)
- [The SmartBFT ordering service](#the-smartbft-ordering-service)
  * [Integrating the SmartBFT consensus library into a Fabric orderer](#integrating-the-smartbft-consensus-library-into-a-fabric-orderer)
  * [Implementing the dependencies required by the library](#implementing-the-dependencies-required-by-the-library)
  * [Implicit byzantine majority signature policy](#implicit-byzantine-majority-signature-policy)
  * [Changes to the peer](#changes-to-the-peer)
  * [Changes to client SDK](#changes-to-client-sdk)
  * [Configuration](#configuration)
  * [Integration tests](#integration-tests)
  * [Performance evaluation](#performance-evaluation)
- [Possible questions and their answers](#possible-questions-and-their-answers)



# Summary
[summary]: #summary

This RFC proposes both a **design** and an **implementation** of a Byzantine Fault Tolerant (BFT) ordering service for Hyperledger Fabric.

The proposal is based on the only publicly available [fork](https://github.com/SmartBFT-Go/) of Fabric that is known to **properly** integrate a BFT consensus library
into the Fabric core while making minimal changes to the code base and architecture. 

The **paper** that describes the library and its integration to Fabric can be found [here](https://smartbft-go.github.io/paper.pdf).

**Consensus Library**: The SmartBFT [consensus library](https://github.com/SmartBFT-Go/consensus) protocols draw upon the [peer reviewed paper](http://www.di.fc.ul.pt/~bessani/publications/edcc12-modsmart.pdf) dubbed "BFT-SMaRT" which describes a BFT algorithm that is similar to PBFT but simpler.
The library is implemented in Go and is well [tested](https://github.com/SmartBFT-Go/consensus/tree/master/test) and [documented](https://smartbft-go.github.io/).

**Integration into Fabric**: The design of the [library integration into Fabric](https://github.com/SmartBFT-Go/fabric) is elegant and re-uses the [existing communication and replication infrastructure](https://github.com/hyperledger/fabric/tree/master/orderer/common/cluster) that is used for the Raft ordering service.

**Administration and operation**: The administration and operation of the BFT ordering service is identical to the Raft ordering service, for instance, dynamic addition and removal of consenters is done in the same manner as Raft.

**Trying out the BFT ordering service**: For those of who want to try things out, take a look at the [sample](https://github.com/SmartBFT-Go/fabric-samples/tree/release-1.4-BFT/first-network) and do `./byfn.sh up -o smartbft`

# Motivation
[motivation]: #motivation

Byzantine Fault Tolerant (BFT) consensus is crucial to any distributed application that operates without a trusted third party.
In the blockchain domain, a consensus algorithm that is only Crash Fault Tolerant (CFT) endangers the safety and liveness of the entire platform.
The reason is that if a block proposer is controlled by a malicous party and participates in CFT consensus, it can make a portion of the network nodes to see one version of reality, and make the rest of the network nodes to see a different reality.
This is due to the fact that a CFT consensus algorithm, nodes are assumed to never send conflicting messages. 

In the blockchain domain, such an attack often leads to a double spending of assets or currency which incurs monetary loss in the short term, and trust degradation and service termination in the long term.

## Hyperledger Fabric and BFT consensus

Hyperledger Fabric is one of (if not the most) most popular permissioned blockchain platforms as of today. 

Yet, surprisingly and regrettably it lacks what **[every](https://docs.corda.net/docs/corda-os/4.5/running-a-notary.html#byzantine-fault-tolerant-experimental) [other](https://docs.goquorum.com/en/latest/Consensus/ibft/ibft/) [competitor](https://www.hyperchain.cn/en/products/hyperchain) [of](https://docs.tendermint.com/master/introduction/what-is-tendermint.html) [it](https://sawtooth.hyperledger.org/faq/consensus/#what-consensus-algorithms-does-sawtooth-support) [possesses](https://docs.kaleido.io/kaleido-platform/blockchain-essentials/consensus/)**, namely innate token support and tolerance to byzantine faults in its transaction ordering.

Even though requirement for a BFT ordering service was raised early on, [in 2016](https://jira.hyperledger.org/browse/FAB-33), Fabric still lacks what is often described as a key component of a blockchain, be it permissioned or permissionless.

# BFT Consensus Library

This proposal suggests to [incorporate](https://github.com/SmartBFT-Go/fabric) the [SmartBFT consensus library](https://github.com/SmartBFT-Go/consensus) into Hyperledger Fabric. 
Despite being developed as a consensus library for Fabric, the SmartBFT consensus library is blockchain agnostic, and can be used for any application that requires total order of transactions.

## The protocols implemented by the library

**Agreement on total order of requests sent by clients**:  
The consensus algorithm provides both safety and liveness under the assumption that up to `f` nodes are faulty. A consenus round starts by clients that issue requests to all nodes and the current leader batches requests into a proposal and initiates the three stage consensus round at which end all `n` nodes except up to `f` of them commit the proposal, or a view change occurs and the proposal is transferred to the leader of the next view.

The three stages of the agreement protocol are `pre-prepare`, `prepare`, and `commit`. The `pre-prepare` and `prepare` phases are used to totally order requests sent in the same view even when the leader, which proposes the ordering of requests, is faulty. 
The `prepare` and `commit` phases are used to ensure that requests that commit are totally ordered across views. 

![alt text](https://raw.githubusercontent.com/SmartBFT-Go/SmartBFT-Go.github.io/master/flow.png)


In the `pre-prepare` phase, the leader batches requests to assemble a proposal, and it then assigns a sequence number to the proposal. Next the leader sends a `pre-prepare` message to all followers containing the proposal, its sequence number and the current view number `v`.

A follower accepts the `pre-prepare` message if the proposal is valid, it is in view `v`, it is expecting this sequence number, and it has not accepted a different `pre-prepare` message for this view and sequence number.
Once a follower accepts a `pre-prepare` message it enters the `prepare` phase by broadcasting a `prepare` message.
The `prepare` message contains a digest of the accepted proposal, the sequence number, and `v`.

Next each node waits for a quorum of `prepare` messages with the same digest, view `v`, and sequence number. 
The quorum size should be big enough to ensure that in the intersection of any two quorums there is at least one correct node. So for example, if `n=3f+1` then the optimal quorum size is `2f+1`. 

After the node is "prepared" it sends a `commit` message with the same content as the `prepare` message, as well as a signature on the proposal.
Each node collects a quorum of `commit` messages and their attached signatures. 
These signatures are later used to notarize the proposal, essentially creating a checkpoint. 
Upon receiving a quorum of validated `commit` messages, the node delivers its proposal and the signatures to the application layer for storage and processing.

The quorum size is calculated in the following manner: Let `n` be the total number of nodes in the algorithm and denote `f = ((n - 1) / 3)`. Then, the quorum is: `Ceil((N + F + 1) / 2)` where `Ceil` is the ceiling function. In layman's terms, the quorum is the minimal set of nodes such that every two such sets intersect in at least 1 honest node.  

**View change**:
The view change protocol is inspired by the _Synchronization Phase_ in this [BFT-SMaRt paper](http://www.di.fc.ul.pt/~bessani/publications/edcc12-modsmart.pdf) (illustrated in the attached figure taken from the BFT-SMaRt paper).
The view change protocol is comprised of three messages: `ViewChange`, `ViewData`, and `NewView` (`Stop`, `StopData`, and `Sync` in BFT-SMaRt).

![alt text](https://raw.githubusercontent.com/wiki/SmartBFT-Go/consensus/images/viewChange.png)

The `ViewChange` message (the first message) is sent to all nodes by some node who suspects the leader is faulty. This message is very lightweight and includes only the next view sequence number. 
Once collecting `2f+1` `ViewChange` messages, a non faulty node is convinced that enough nodes consider the leader as faulty, and it will send a `ViewData` message to the next view's leader.

The `ViewData` message is similar to the `ViewChange` message suggested by [PBFT](http://pmg.csail.mit.edu/papers/osdi99.pdf). It is signed, as it will serve as proof in the next phase. The message contains the last checkpoint (last decided proposal) and the next proposal, if it exists, with its state (proposed or prepared). 
However, as opposed to the PBFT `ViewChange` message, the `ViewData` message is sent only to the next leader.

The new leader, after receiving `2f+1` `ViewData` messages, sends a `NewView` message to all nodes. This message is similar to the `NewView` message suggested by [PBFT](http://pmg.csail.mit.edu/papers/osdi99.pdf).
It includes a proof of the validity of the new view, the `2f+1` `ViewData` signed messages.
The nodes check if they need to catch up with the last decision or if they need to agree on the next proposal, all contained in the `NewView` message.

**Leader liveness monitoring and failover**:
A Byzantine tolerant consensus algorithm must ensure the liveness of the state machine replication operation in an event where the leader has crashed, is unreachable, or is malicious, for example, censoring client requests. 

Malicious leaders may not include client requests in their proposals, or may try to suspend progress by not sending proposals to nodes. To that end, whenever a valid request arrives to a follower - it starts a timer and waits for the request to be included in some proposal. If the timer times out, it sends the request to the leader and resets the timer. Upon a second timeout - the follower suspects the leader, and starts lobbying for a view change to overthrow the leader.
The use of two timeouts is as suggested by [BFT-SMaRt paper](http://www.di.fc.ul.pt/~bessani/publications/edcc12-modsmart.pdf). In the first timeout the client is suspected to not submitting the request to the leader and only in the second timeout the leader is suspected.

However, even if the leader is honest it might crash or simply be disconnected from the rest of the nodes, and therefore cannot broadcast new proposals.
If the leader crashes or its communication with the cluster is severed at a time where there is no application traffic, it is imperative to detect this as fast as possible and elect a new leader to prevent availability from being impaired. 
Hence, the leader is expected to periodically send heartbeat messages to the followers.
If a follower hasn't received a heartbeat from the leader within a timely manner, it suspects the leader has crashed or is unavailable and resolves to start a view change.

**Dynamic reconfiguration**: The library seamlessly supports addition and removal of nodes without any downtime. In contrast to many consensus libraries which force the application to issue a special command which is replicated throughout the nodes, the library is notified by the application about a reconfiguration through the API which it interacts with it.
Since the agreement protocol ensures there is at most a single in-flight proposal at any time (no pipelining), no corner cases (such as a series of proposals with a node addition in between) need to be addressed by complex machinery like no-op proposals to drain the pipeline as proposed in the literature. 

## Embedding the library in a distributed application

 The [SmartBFT library](https://github.com/SmartBFT-Go/consensus) presentes a minimal interface to the application that uses it:

**Input**: To submit requests to be ordered, the application invokes `SubmitRequest` and passes a byte representation of the request.
````
SubmitRequest(req []byte) error
````

The library offloads communication to the application that embeds it, hence it is the responsibility of the application to feed the messages from other nodes, or requests forwarded from other nodes into the library instance:

````
HandleMessage(sender uint64, m *Message)

HandleRequest(sender uint64, req []byte)
````

**Output**: The library outputs requests in totally ordered batches by notifying the application via a callback:

````
Deliver(proposal bft.Proposal, signature []bft.Signature) Reconfig
````

The application is free to implement the callback as it sees fit in order to realize its desired functionality. 
When embedded inside a blockchain application, the proposal would often denote a block constructed by the leader of that round, and the signatures array would contain a quorum of signatures collected by the library from the various nodes. The Reconfig is an object that allows the application to convey to the library to reconfigure itself (for example, to expand or reduce the number of nodes).

**Dependencies**: The library interacts with the external world through the application, by delegating capabilities such as cryptography, communication and storage to the application. 
The full list of dependencies can be found [here](https://smartbft-go.github.io/godoc/pkg/github.com/SmartBFT-Go/consensus/pkg/api/index.html), and below the important ones are listed:

- **[Assembler](https://smartbft-go.github.io/godoc/pkg/github.com/SmartBFT-Go/consensus/pkg/api/index.html#Assembler)**: The library aggregates requests into batches and then executes a 3 staged BFT protocol which first disseminates the batch from the leader, and then notarizes the batch across a byzantine majority of nodes. In some cases, the application attaches additional information to each batch, such as a header of the batch. The library delegates the construction of the batch to the application by calling `AssembleProposal(metadata []byte, requests [][]byte) Proposal`, thus feeding into the application requests as well as metadata used by the library, and it receives in return an opaque proposal which it can then feed into the application for verification and commit. 
- **[Verifier](https://smartbft-go.github.io/godoc/pkg/github.com/SmartBFT-Go/consensus/pkg/api/index.html#Verifier)**: The library delegates verification of signatures, requests and proposals to the application. For example, whenever a follower receives a batch (proposal) from the leader, it consults the application if the proposal only contains requests that are semantically sound to the application by invoking `VerifyProposal(proposal bft.Proposal) ([]]RequestInfo, error)` which returns digests of all requests in the batch.
- **[Signer](https://smartbft-go.github.io/godoc/pkg/github.com/SmartBFT-Go/consensus/pkg/api/index.html#Signer)**: The library invokes `SignProposal(bft.Proposal) *bft.Signature` in order to notarize a batch of requests, and also uses `Sign([]byte) []byte` in its view change protocol.
- **[Write Ahead Log (WAL)](https://smartbft-go.github.io/godoc/pkg/github.com/SmartBFT-Go/consensus/pkg/api/index.html#WriteAheadLog):** At startup, the library is given the entire content of the WAL and from that point onwards, it interacts with the WAL by invoking `Append(entry []byte, truncateTo bool) error`. In some points of its execution, it may be safe to truncate the WAL, and then the library notifies the WAL in its next invocation with the `trucateTo` flag.
While the library abstracts out the WAL to an interface, the library itself contains an [implementation](https://github.com/SmartBFT-Go/consensus/tree/master/pkg/wal) of a WAL and the application may choose to use the one provided by the library.
- **[Synchronizer](https://smartbft-go.github.io/godoc/pkg/github.com/SmartBFT-Go/consensus/pkg/api/index.html#Synchronizer)**: In many cases, a node might be disconnected for a long time and therefore it is more efficient to make it catch up with the rest of the nodes. Whenever the library instance detects it should synchronize, it invokes the `Sync() bft.SyncResponse` which delegates control to the application and replicates the data. In case a reconfiguration is due after synchronizing the data, the library recognizes this by looking at the return value and acts accordingly.

# The SmartBFT ordering service

The SmartBFT consensus library has been integrated into a fork of Fabric which was branched from Fabric version 1.4.x and has been tested and benchmarked.
Whenever a commit is merged to the production branch, a docker image is automatically built and published to [dockerhub](https://hub.docker.com/u/smartbft).

## Integrating the SmartBFT consensus library into a Fabric orderer


**High level architecture and component interactions**: At the heart of the SmartBFT ordering service node, there are instances of the consensus protocol for each channel.
As can be seen in the figure below, there is a single application channel and a system channel and they each have their own independent SmartBFT library consensus instance (1).

![alt text](https://raw.githubusercontent.com/SmartBFT-Go/SmartBFT-Go.github.io/master/fabric-arch.png)

To communicate with other consensus instances, the SmartBFT consensus instance utilizes the builtin Fabric communication infrastructure (2) which was designed to operate within a malicious setting.
Whenever a block is notarized by a quorum of nodes, the consensus instance writes it into the orderer's leder (3). 
In case the orderer detects it is too far behind the rest of the nodes, it reaches out to the builtin replication infrastructure (4) and uses it to replicate blocks, which also makes its library able to catch up with the 
rest of the nodes due to the fact that Fabric blocks contain consensus specific metadata that is used by the consensus protocol.
Lastly, whenever clients or peers need to pull blocks from the ordering service, they do so in the same manner by using the regular gRPC service (5) that delivers blocks.

## Implementing the dependencies required by the library

In order to integrate the library, the orderer must implement all needed [dependencies](https://smartbft-go.github.io/godoc/pkg/github.com/SmartBFT-Go/consensus/pkg/api/index.html) required by the library.
Below we enumerate how the dependencies were realized in the SmartBFT orderer:

- **[Assembler](https://github.com/SmartBFT-Go/fabric/blob/release-1.4-BFT-2/orderer/consensus/smartbft/assembler.go#L39-L83)**: The assembler isolates config transactions in their own batch, as required by Fabric semantics, and constructs a proposal that contains an un-signed block.
- **[Verifier](https://github.com/SmartBFT-Go/fabric/blob/release-1.4-BFT-2/orderer/consensus/smartbft/verifier.go)**: The verifier ensures that all creators of transactions satisfy the ChannelWriters policy. It also ensures that blocks proposed from the leader maintain the hash chain, and contain only legal transactions. 
    In case of a configuration transaction, it simulates it and rejects if the simulation fails. It is also reponsible to verify consenter signatures.
- **[Signer](https://github.com/SmartBFT-Go/fabric/blob/release-1.4-BFT-2/orderer/consensus/smartbft/signer.go)**: The signer taps into the orderer's local MSP to sign messages. Unlike Fabric, signature headers do not carry the full certificate, but only the identifier of the node that signed it. This preserves space both in transit and in rest, and in fact despite blocks contain a quorum of signatures, their space overhead is negligible and smaller than the overhead of a Raft node containing a single signature.
- **[Synchronizer](https://github.com/SmartBFT-Go/fabric/blob/release-1.4-BFT-2/orderer/consensus/smartbft/synchronizer.go)**: The synchronizer uses the Fabric builtin block replication infrastructure, but unlike the CFT behavior of the vanilla Fabric, it verifies a quorum of signatures in every block.
- **Communication**: Both [egress](https://github.com/SmartBFT-Go/fabric/blob/release-1.4-BFT-2/orderer/consensus/smartbft/egress.go) and [ingress](https://github.com/SmartBFT-Go/fabric/blob/release-1.4-BFT-2/orderer/consensus/smartbft/ingress.go) communication pathways completely re-use the builtin Fabric cluster infrastructure, as it was designed to be resilient to malicious nodes and is non-blocking. A malicious or slow node cannot effect any other node that communicates with it. 

## Implicit byzantine majority signature policy

A Fabric block has to be signed by a quorum of consenters, and only by the specific consenters that were defined as consenters during the agreement on that block.
To that end, the SmartBFT Fabric block verification [defines a new policy](https://github.com/SmartBFT-Go/fabric/blob/release-1.4-BFT-2/common/policies/orderer/implicit_orderer.go) which is satisfied only by a threshold of signatures out of the identities that comprise the consenters that appeared in the channel configuration at the time of agreeing on the block.


## Changes to the peer
Once a node collects a quorum of signatures on a proposal (a Fabric block), it commits the block and the signatures into its ledger. These blocks, and the corresponding signatures are pulled by the peers via gRPC.  Since each block is signed by a quorum of nodes, every peer can determine whether a block is fake or genuine by validating the signatures on the block. 

While a malicious orderer cannot forge a block, it can deprive a peer from blocks. Hence, receiving the stream of blocks from a single orderer exposes the peer to a censorship attack.

To protect against such an attack, the SmartBFT ordering service implements a block delivery client that is resilient to censorship attacks:
The approach takes advantage of the Fabric block structure, in which the orderer signs the header which contains the hash over the block data, allowing the peer to verify the signatures on a received block even if the block’s data is omitted. Thus, the peer can ask the orderer for block headers and their corresponding signatures, and avoid pulling the same block from different orderers but instead pull a block from a single orderer, and only headers and signatures from the rest of the orderers which minimizes the network bandwidth. The peer monitors all such connections, and if a block hasn't been received from an orderer for a threshold period of time, but its header has been received (and verified) from `f` or more orderers, then the peer suspects the orderer is censoring the peer, and selects a different orderer to fetch blocks from.


## Changes to client SDK
Since orderers can be malicious and thus either drop transactions or never include them in a block when assembling proposals, any client that interacts with the BFT ordering service needs to submit the transaction to all orderers.
We consider such a change as trivial and minimal and argue that it should be made in every SDK with ease.

## Configuration
The nice thing about SmartBFT, is that the operation and configuration is equivalent to Raft, so if an organization has infrastructure that administers Raft orderers, it can be used to administer BFT orderers without any additional overhead.

The [local configuration](https://github.com/SmartBFT-Go/fabric/blob/release-1.4-BFT-2/sampleconfig/orderer.yaml) of the BFT orderer inherits from the local configuration of the Raft orderer but only uses the [WAL directory](https://github.com/SmartBFT-Go/fabric/blob/release-1.4-BFT-2/sampleconfig/orderer.yaml#L371) configuration key. 

The [channel configuration](https://github.com/SmartBFT-Go/fabric-samples/blob/release-1.4-BFT/first-network/configtx.yaml#L445) of the BFT ordering service is very similar to Raft. In addition to the TLS certificates of each consenter, there exist its MSP ID, enrollment certificate and unique identifier:

````
        Orderer:
            <<: *OrdererDefaults
            OrdererType: smartbft
            SmartBFT:
                Consenters:
                - Host: orderer.example.com
                  MSPID: OrdererMSP
                  Port: 7050
                  ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/server.crt
                  ConsenterId: 1
                  Identity: crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp/signcerts/orderer.example.com-cert.pem
                -  ...
                -  ...
````

## Integration tests
As appropriate for a production grade BFT ordering service, there are numerous intergration tests that test various corner cases such as:
- Crash fault tolerance
- Nodes detecting they are behind due to consensus messages and catching up with the rest of the nodes
- Nodes detecting they are behind based on heartbeats and catching up with the rest of the nodes
- Addition and removal of nodes dynamically without downtime
- A complete removel and replacement of all nodes without downtime
- Leader failover due to request censorship

## Performance evaluation

A performance evaluation of the SmartBFT ordering service was conducted.
The evaluation examined different cluster sizes and different batch sizes, as well as various geographical locations around the globe.
The full performance evaluation can be found in the [paper](https://smartbft-go.github.io/paper.pdf).

|                              LAN                              |                                                                                                                             WAN                                                                                                                            |
|:-------------------------------------------------------------:|:----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------:|
|  ![alt text](https://raw.githubusercontent.com/SmartBFT-Go/SmartBFT-Go.github.io/master/lan.png)                            |                                                                                                                           ![alt text](https://raw.githubusercontent.com/SmartBFT-Go/SmartBFT-Go.github.io/master/wan.png)                                                                                                                          |
| In LAN, the throughput is above 2000 transactions per second. It can be observed that even smaller batch sizes do not significantly degrede the performance of the system. Furthermore, the size of the cluster also didn't affect the throughput significantly.| In WAN, nodes were deployed all over the world: Dallas, London, Washington, San Jose, Toronto, Sydney, Milan, Chennai, Hong Kong and Tokyo and with a decent batch size (1000 transactions per block) a throughput of 1,000 transactions per block was achieved. |

# Possible questions and their answers

-  **Q**: Why do we even need a Byzantine Fault Tolerant ordering service in Fabric? If it would be really needed then Fabric would not have been so widely adopted by companies.

   **A**: The market share that uses blockchain technology is far from its full potential. As technology evolves, commerce and industry evolve with it. 
          Introducing Byzantine Fault Tolerance to Fabric will only widen the use cases it will be used for, and new commercial platforms would materialize.

-  **Q**: You say the BFT has already been integrated into a fork of Fabric, however it has been integrated into a fork of Fabric 1.4.x while Fabric introduces new features to the 2.x branch series.

   **A**: Integrating a consensus library into Fabric has two aspects: (a) The library itself, and (b) the changes to be made in Fabric. The library itself is agnostic of the Fabric version, and the changes to Fabric mostly revolve around the consensus plugin whose code base is the same in 1.4.x and in 2.x. In fact the only real difference in the code base that is relevant is the deliver client in the peer, which is a couple of days of work to somone that is well versed in the code base.
   
   
- **Q**: How do we know that the implementation you propose is secure and sound?
  
  **A**: The design and implementation of the library and its integration to Fabric [was made](https://github.com/orgs/SmartBFT-Go/people) by [key contributors to the Fabric core](https://github.com/hyperledger/fabric/graphs/contributors). 
  In fact, two of them are maintainers of the Fabric core, and two of them (combined) have contributed the bulk of the Raft ordering service code to this day.
  Despite having such a strong evidence to the correctness and security of the implementation, we expect all code submitted to the Fabric repository or referenced by it to 
  undergo the standard code review process, which will only amplify the guarantee that the BFT orderer is safe to use.
  
- **Q**: I think that using a different BFT consensus algorithm is preferable over using BFT-SMaRT as it is (more performant / more fair / can brew coffee). 
         If we introduce a BFT ordering service to Fabric, it should use the best possible BFT algorithm in the world.
  
  **A**: The SmartBFT consensus library implemented BFT-SMaRT whose paper was accepted into several academic conferences, and its safety and correctness draws upon its simplicity and similarity to PBFT. In fact, the algorithm is a non pipelined PBFT with a more efficient view change protocol.
         While a different protocol may be better in some aspects, a protocol that drifts too far from what is proposed here, will require massive changes that will ripple throughout the entire Fabric architecture and code base and will require a massive amount of work.
         In contrast, the design and implementation of the BFT ordering service that is proposed here, preserves Fabric semantics and structure and re-uses existing Fabric infrastructure to its full extent, so it is very cheap to integrate since the development and testing has already been done by a team of people that have
         both experience in distributed systems and are also key contributors to the Fabric core. Lastly, if we are thinking about the long term, nothing prevents the open source community from implementing additional BFT types in the future, and any work done in the future will surely benefit from the process of integrating 
         a BFT ordering service type in the present.
         
- **Q**: The proposed algorithm only changes the leader in case it crashes or misbehaves. This impairs fairness and makes the nodes utilize an asymmetrical amount of network bandwidth.
  
  **A**: Good point. Indeed the current protocol doesn't rotate the leader, however enabling leader rotation support is [being worked on](https://github.com/SmartBFT-Go/consensus/tree/leader_rotation) these very days, and judging from the integration tests, [it already works in the development branch](https://github.com/SmartBFT-Go/fabric/blob/release-1.4-BFT-development/integration/smartbft/smartbft_test.go).
  
- **Q**: What about scalability? I have a need to run hundreds of ordering nodes, but it doesn't seem this protocol will scale with so many nodes.

  **A**: Indeed, classical consensus protocols scale very poorly in huge networks. At some point, broadcasting a message from a node over the internet to too many nodes becomes slow and expensive.
         Fortunately, we have contacted the developers of the SmartBFT project and they claim they have a design that allows one to deploy thousands of Fabric BFT orderer nodes over the internet without impairing throughput.
         They said the design is not intrusive to Fabric and in fact doesn't require any change in the Fabric code at all. They plan on starting implementing this early next year.

- **Q**: What about fairness? It doesn't seem like the protocol ensures fairness.

  **A**: While it doesn't ensure fairness in the strong sense of the term, no malicious leader can censor a transaction because that would make the rest of the followers overthrow the leader with a view change, once the timeout expires. 
         Additonally, once leader rotation is implemented, you will have amortized fairness because honest leaders dequeue transactions in FIFO order, so while there is no strict quality of service, it is good enough for most applications.
         
- **Q**: What if a client goes rogue and bombards the orderer with too many transactions in order to deny the service to everyone? 

  **A**: Since Fabric is a permissioned blockchain, such behavior can be easily monitored and the majority of the administrators could team up and issue a configuration transaction that revokes the right of the client (or its organization) from interacting with the ordering service.
   