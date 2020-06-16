
- Feature Name: (Reshaping Hyperledger Fabric block dissemination)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Fabric Component: (Fabric peer)
- Fabric Issue: (TBD)

# Introduction
[Introduction]: #Introduction

In Fabric, the term “gossip” is a term often used when referring to the component that manages all peer to peer interactions.It is named after the technique that Fabric employs to disseminate blocks among peers, after reception from the ordering service.
Henceforth, when referring to the architecture or code of peer to peer communication, message dissemination, etc. in Fabric, the term “gossip” will be used.

During the development of Fabric v1.0, block dissemination was gossip’s sole purpose, however as Fabric matured it picked up more roles, such as private data pre-image dissemination and providing membership information to service discovery.

Undoubtedly, a peer to peer infrastructure that provides membership information, disseminates information according to arbitrary policies, and can even be used by applications as a secure authenticated point to point channel primitive is a game changing factor in every blockchain platform.

However, unfortunately there are shortcomings in the way the gossip component is integrated into Hyperledger Fabric, and the aim of this document is to propose how to address them.


# Background:
[Background]: #Background

The gossip component in the Fabric peer is built in layers. The bottom most layer is the [communication layer](https://github.com/hyperledger/fabric/tree/master/gossip/comm) which manages a pairwise connection between every two peers, receives requests to send messages to remote peers, and propagates messages from remote peers to layers above it. Information about peers in the network such as their endpoint and organizational association is maintained by the membership layer.

```
     ______________________________________________________________________
    |  Message Dissemination | Election | Private Data | Block Replication |
    |________________________|__________|______________|___________________|
    |                              Channel                                 |
    |______________________________________________________________________|
    |                           Membership                                 |
    |______________________________________________________________________|
    |                   communication            |      Identity           |
    |____________________________________________|_________________________|

```

While the [membership layer](https://github.com/hyperledger/fabric/tree/master/gossip/discovery) maintains information that is channel independent such as identity to endpoint mapping and whether certain peers are dead or alive, the [channel layer](https://github.com/hyperledger/fabric/tree/master/gossip/gossip/channel) maintains information that is channel-scoped, such as the ledger heights and chaincodes installed that peers publish to one another, as well as which peers participate in the various channels.

Even though a peer maintains long lasting connections to all peers known to it, it disseminates information about itself to all eligible peers via epidemic dissemination by having peers gossip with each other.
This is done via the [message dissemination layer](https://github.com/hyperledger/fabric/tree/master/gossip/gossip) which mainly includes routing rules based on information from the membership layer and the [identity layer](https://github.com/hyperledger/fabric/tree/master/gossip/identity).

When a peer broadcasts a message proclaiming some information about itself, it puts its unique identity identifier into the message and signs it. Whenever a peer receives such a message, it performs an in-memory lookup of the full identity of the peer according to the identity identifier, and validates the signature of the message using the identity layer.


Now, possessing the ability to communicate with peers, authenticate their messages and determine which peers are alive and what channels they belong to, gossip can finally be used for peer to peer block dissemination which is actually implemented as additional instances of routing rules and data structures within the message dissemination layer.

Recall that in Hyperledger Fabric, blocks are produced by the ordering service but are consumed by peers in order to build the world state which is essential for smart contract execution.


Let’s enumerate the various approaches to block dissemination from the ordering service to the network peers:

1. Have all peers connect to the ordering service and pull blocks consecutively.
2. Have a small subset of nodes connect to the ordering service and pull blocks on behalf of all peers in the network.
3. Have ordering service nodes participate in the gossip protocol and act as source nodes for block propagation.

Let us discuss the upsides and downsides of each approach:

1. This is obviously the simplest approach of all, however the more peers pulling blocks the less bandwidth is available for the orderer nodes to run the consensus protocol.
2. This eliminates the bandwidth penalty on the ordering service, but requires peers to autonomously and collaboratively determine the representatives to pull the blocks on behalf of the rest, as well as taking care of failover and high availability should these peers become unreachable.
3. This fundamentally contradicts the heterogeneous architecture of Hyperledger Fabric, and amplifies the difficulty in troubleshooting and diagnostics, as well as poses implementation and integration technical difficulties.

The approach eventually chosen was approach 2, and it is the central claim of this document that we should shift to approach 1.
Before discussing why or how we should tackle this, let us get familiar with how peers designate representatives to pull blocks on behalf of the rest.

**Selecting representative peers for block retrieval**: In every organization, peers run an instance of a leader election protocol (implemented in the [leader election layer](https://github.com/hyperledger/fabric/tree/master/gossip/election)) to select a single peer, termed the “leader” among the candidates. It is the responsibility of the leader peer to connect to the ordering service, pull blocks and disseminate them through epidemic dissemination (gossip) among the rest of the peers termed “followers”.

To address failover and high-availability, the leader peer periodically sends heartbeats to the followers, and whenever a follower suspects a loss of heartbeats over a too greater period of time, it starts its own leader election campaign where it either finishes as a leader, or as a follower to a different leader with a lower identity identifier. Additionally, leaders step down and become followers if they continuously fail to retrieve blocks from the ordering service.


Lastly, there are two additional layers worth mentioning:

- Peers are also able to replicate blocks from each other in a point to point manner based on a request-response protocol which is less efficient than the streaming one used in the deliver service that exists in the peer and in the orderer. This is implemented in the [block replication](https://github.com/hyperledger/fabric/tree/master/gossip/state) layer.
- Whenever a block is received from either a peer or an orderer and before it is committed, the peer scans the transactions in the block for private data writes in order to fetch them either locally from storage or from remote peers. This is implemented in the [private data layer](https://github.com/hyperledger/fabric/tree/master/gossip/privdata) that uses gossip as a dependency and is rather tightly coupled with Fabric.

# Drawbacks of leader based block dissemination
[drawbacks]: #drawbacks

Even though the leader based block dissemination approach seems like a good balance between scalability and simplicity, it suffers from drawbacks in many dimensions:

1. **Identifier based election might lead to block starvation**: Suppose a new peer joins the network with an identifier lower than every other existing peer, and that the rest of the peers in the peer’s organization have a ledger height of 1,000,000 blocks. Further assume that after some time a network problem occurs which causes all peers to elect themselves leaders.
When the network problem concludes, the lately added peer will make the rest of the peers in the organization to become followers due to leader election breaking symmetry with identifiers. From this point on and until the leader catches up with the followers, all blocks it receives and propagates are blocks the followers have already committed in the past, thus the newly added peer is effectively starving the rest of the peers in the organization.
2. **Inadequate when peers have diverged ledger heights**: When a new peer joins the network its ledger is significantly behind the rest, yet it still receives blocks from peers in its organization which only end up discarded because it has no short term use for them.
3. **Memory overhead**: While gossip is an ephemeral component (no way to persist data to stable storage), it often receives blocks out of order due to the random nature of the epidemic style block dissemination. Consequently, it stores blocks in an in-memory sliding window which evicts old blocks in favor of newer blocks. Even though blocks are eventually purged from memory, it is not immediate and therefore every channel the peer participates in, consumes memory throughout the runtime of the peer. This trivially limits the amount of channels a peer can be part of, and increases the memory consumption of the peer.
4. **Complications** of diagnostics: Making only a single peer per organization to retrieve blocks makes it that network administrators and site reliability engineers need to determine which is the leader peer when troubleshooting issues related to peers lagging behind.
5. **Non determinism**: The non deterministic and highly concurrent nature of epidemic based block replication makes it harder to build stable tests.

**Point to point block replication**: In case a peer is too far behind the rest, as in situations where it is just added to the network, it may choose a peer to initiate synchronization from in a point to point manner.
This entails a series of requests and responses where the peer behind asks for several consecutive blocks, and the peer ahead sends them back.
Due to the idle time spent in waiting for blocks to be sent or for requests to be sent.


This protocol is slower than one where the peer retrieves blocks from the orderer. In the latter, the orderer can just send as many blocks per second as the gRPC stream throttling allows, thus maximizing the block transfer rate.


# The proposed new block dissemination method
[Proposed]: #Proposed

An alternative block dissemination method is proposed to replace the current block dissemination.

**Note:** Whenever an "orderer" is mentioned, we mean some orderer among the available orderers via some selection policy that is orthogonal to this proposal.

The proposal is to be implemented in two phases:

**Phase I**: Gossip block replication among peers will cease to operate, and the relevant code will be removed. This means that both gossip block replication layer and leader election layers will be removed and each peer will always pull only from the ordering service.

**Phase II**: To address use cases where new peers join an organization and while pulling from the ordering service might be possible, it
              might be slower than pulling from peers within the organization in cases of across continent deployments.
              The block dissemination method will then utilize the block deliver service that exists on both peers and orderers, and the following shall hold:
- When a peer starts up, or disconnects from an orderer/peer replication session, it polls the orderer to see the difference between its ledger height and the orderer's ledger height.
If the peer's ledger is too far behind (further than a configurable threshold) then it may (if enabled in configuration) choose a peer from its own organization to replicate from.
- When choosing a peer to replicate from, that peer needs to be a peer that is not too far behind the orderer. In other words, the difference between the orderer's ledger height and the ledger height of the peer to pull blocks from needs to be less than a configurable threshold.

It might be that the peer is far behind the orderer and doesn't see any other peer in its membership view,
however subsequent sampling of the membership view will reveal a peer that is a good candidate to pull blocks from instead of the orderer.
Therefore, the process for block retrieval is an endless loop that its body constantly evaluates if it needs
to pull from the orderer or from the peer based on:
*  Whether we are permitted (configuration wise) to pull from peers at all
*  How far we are from the ordering service
*   How far are other peers in our organization from the ordering service


~~~~
myHeight = ledger.Height()
if ordererLedgerHeight == 0
	ordererLedgerHeight = RetrieveOrdererLedgerHeight()


// We can replicate from a peer of our org if:
// (1) It is permitted in the configuration
// (2) That peer's height is not too far from the ordering service ledger height
aheadPeerFromMyOrg = samplePeerFromMyOrgWithHighestLedgerHeight()
if ordererLedgerHeight - aheadPeerFromMyOrg.Height < Threshold && peerReplicationEnabled
	candidatePeer = aheadPeerFromMyOrg

// If there exists a canidate peer to be used for block retrieval,
// and we are currently pulling from the orderer, switch to use a peer.
if candidatePeer != nil && pullingFromOrderer
	blockSourceRetriever = initializeBlockSourceRetrieval(candidatePeer)
	pullingFromOrderer = false

// If we currently not have any block source,
// check to see which source we want to pull from (orderer/peer)
// based on how far we are from the orderer
if blockSourceRetriever == nil
	if ordererLedgerHeight - myHeight > Threshold && peerReplicationEnabled
		// We are rather far from the orderer so let's see if we have a peer in our org
		// that is not far from it
		aheadPeerFromMyOrg = samplePeerFromMyOrgWithHighestLedgerHeight()
		if ordererLedgerHeight - aheadPeerFromMyOrg.Height < Threshold
			// There is a peer from our org that is not far from the orderer,
			// so let's pull from it instead of the orderer
			blockSourceRetriever = initializeBlockSourceRetrieval(candidatePeer)
			pullingFromOrderer   = false
	else
		// We either have peer replication disabled, are not too far from the orderer,
		// or all peers we know of are also too far from the orderer,
		// therefore we pull from the ordering service
		blockSourceRetriever = initializeBlockSourceRetrieval(orderer)
		pullingFromOrderer   = true


if blockSourceRetriever != nil
	block, err := blockSourceRetriever.RetrieveBlock()
	if err != nil
		pullingFromOrderer = false
		blockSourceRetriever = nil
		continue
	verify and commit block...
~~~~

# Changes to the gossip code and extraction to a separate repository
The semantic changes proposed will be addressed with the following code changes:

- The block dissemination mechanisms from the **Message Dissemination** layer will be removed
- The **Block Replication** and **Election** layers will be removed entirely from the code base
- The remaining gossip code, modulo the **Private Data** layer will be extracted into a separate repository.

As a result, the new structure will be as following:

```
     __________________________________________________
    |  Message Dissemination |      Private Data       |
    |________________________|_________________________|
    |                     Channel                      |
    |__________________________________________________|
    |                   Membership                     |
    |__________________________________________________|
    |         communication            |    Identity   |
    |__________________________________|_______________|

```

## Extraction to a separate repository
Except to the private data layer, all of gossip is pretty self contained:
- It exposes a set of APIs to the peer, namely the ability to sample the membership or send messages to remote peers and receive messages from them.
- It consumes self defined APIs implemented by the peer, mostly being [cryptographic](https://github.com/hyperledger/fabric/blob/master/gossip/api/crypto.go) and [authentication](https://github.com/hyperledger/fabric/blob/master/gossip/api/channel.go) operations.


Hence, we shall extract all layers (but private data) into a separate repository, and the new layer structure in the peer will be thinned down to:
```
     __________________________________________________
    |                        |      Private Data       |
    |                        |_________________________|
    |                                                  |
    |              Vendor of                           |
    |              hyperledger/fabric-gossip           |
    |                                                  |
    |                                                  |
    |__________________________________________________|

```

This has several advantages:

- **Reduced unit test time and stability**: When building a pull request for the **fabric** repository, gossip tests will now not run and hence this would considerably reduce the time to complete a CI build. This would also remove the success amplification caused by false negatives (notoriously known as CI flakes).
- **Loose coupling**: This would enable to change or replace the gossip component with more ease, as the boundary between the fabric core is well defined and concrete. Additionally, when done correctly - future open source projects that needs a messaging layer fit for a permissioned and un-trusted environment might be able to re-use the component and then even to contribute to the upstream based on their various needs.