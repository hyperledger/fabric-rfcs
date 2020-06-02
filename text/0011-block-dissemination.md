---
layout: default
title: RFC Template
nav_order: 3
---

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

The gossip component in the Fabric peer is built in layers. The bottom most layer is the communication layer which manages a pairwise connection between every two peers, receives requests to send messages to remote peers, and propagates messages from remote peers to layers above it. Information about peers in the network such as their endpoint and organizational association is maintained by the membership layer.

While the membership layer maintains information that is channel independent such as identity to endpoint mapping and whether certain peers are dead or alive, the channel layer maintains information that is channel-scoped, such as the ledger heights and chaincodes installed that peers publish to one another, as well as which peers participate in the various channels.

Even though a peer maintains long lasting connections to all peers known to it, it disseminates information about itself to all eligible peers via epidemic dissemination by having peers gossip with each other.

When a peer broadcasts a message proclaiming some information about itself, it puts its unique identity identifier into the message and signs it. Whenever a peer receives such a message, it performs an in-memory of the full identity of the peer according to the identity identifier, and validates the signature of the message. This calls for the need of a component that replicates known identities among peers, and is termed the identity store.
Now, possessing the ability to communicate with peers, authenticate their messages and determine which peers are alive and what channels they belong to, gossip can finally be used for peer to peer block dissemination.

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

**Selecting representative peers for block retrieval**: In every organization, peers run an instance of a leader election protocol to select a single peer, termed the “leader” among the candidates. It is the responsibility of the leader peer to connect to the ordering service, pull blocks and disseminate them through epidemic dissemination (gossip) among the rest of the peers termed “followers”.

To address failover and high-availability, the leader peer periodically sends heartbeats to the followers, and whenever a follower suspects a loss of heartbeats over a too greater period of time, it starts its own leader election campaign where it either finishes as a leader, or as a follower to a different leader with a lower identity identifier. Additionally, leaders step down and become followers if they continuously fail to retrieve blocks from the ordering service.

# Drawbacks of leader based block dissemination
[drawbacks]: #drawbacks

Even though the leader based block dissemination approach seems like a good balance between scalability and simplicity, it suffers from drawbacks in many dimensions:

1. Identifier based election might lead to block starvation: Suppose a new peer joins the network with an identifier lower than every other existing peer, and that the rest of the peers in the peer’s organization have a ledger height of 1,000,000 blocks. Further assume that after some time a network problem occurs which causes all peers to elect themselves leaders.
When the network problem concludes, the lately added peer will make the rest of the peers in the organization to become followers due to leader election breaking symmetry with identifiers. From this point on and until the leader catches up with the followers, all blocks it receives and propagates are blocks the followers have already committed in the past, thus the newly added peer is effectively starving the rest of the peers in the organization.
2. Inadequate when peers have diverged ledger heights: When a new peer joins the network its ledger is significantly behind the rest, yet it still receives blocks from peers in its organization which only end up discarded because it has no short term use for them.
3. Memory overhead: While gossip is an ephemeral component (no way to persist data to stable storage), it often receives blocks out of order due to the random nature of the epidemic style block dissemination. Consequently, it stores blocks in an in-memory sliding window which evicts old blocks in favor of newer blocks. Even though blocks are eventually purged from memory, it is not immediate and therefore every channel the peer participates in, consumes memory throughout the runtime of the peer. This trivially limits the amount of channels a peer can be part of, and increases the memory consumption of the peer.
4. Complications of diagnostics: Making only a single peer per organization to retrieve blocks makes it that network administrators and site reliability engineers need to determine which is the leader peer when troubleshooting issues related to peers lagging behind.
5. Non determinism: The non deterministic and highly concurrent nature of epidemic based block replication makes it harder to build stable tests.

**Point to point block replication**: In case a peer is too far behind the rest, as in situations where it is just added to the network, it may choose a peer to initiate synchronization from in a point to point manner.
This entails a series of requests and responses where the peer behind asks for several consecutive blocks, and the peer ahead sends them back.
Due to the idle time spent in waiting for blocks to be sent or for requests to be sent.


This protocol is slower than one where the peer retrieves blocks from the orderer. In the latter, the orderer can just send as many blocks per second as the gRPC stream throttling allows, thus maximizing the block transfer rate.


# The proposed new block dissemination method
[Proposed]: #Proposed

An alternative block dissemination method is proposed to replace the current block dissemination.

The block dissemination method utilizes the block deliver service that exists on both peers and orderers, and the following shall hold:
- At each point in time, every peer is connected to a block source for block retrieval,
be it either the ordering service or some peer of its choice.
- In a periodic manner, the peer re-assesses whether it should switch block sources.
Generally, if a peer is extremely behind the rest, it would prefer peers, however if it is not behind the rest, or is only slightly behind, it would prefer the ordering service.
- If applicable, a peer would prefer to retrieve blocks from peers of its own organization over peers from remote organizations.
- A configuration option will be put in place to fine tune the ledger height threshold that determines whether pulling blocks from peers is preferred over the ordering service.
Furthermore, a configuration option that determines the frequency of reassessment of the block source will also be put in place.

Informally, the process for block retrieval is an endless loop that its body performs:
~~~~
If blockSource == nil
    aheadPeerForeignOrgs, aheadPeerFromMyOrg := sampleMembershipView()
If aheadPeerFromMyOrg.Height - myHeight > Threshold
	blockSource = aheadPeerFromMyOrg
If aheadPeerForeignOrgs.Height - myHeight > Threshold && blockSource == nil
	blockSource = aheadPeerForeignOrgs
if blockSource == nil
	blockSource = orderingService
if blockSourceRetriever == nil
	blockSourceRetriever = initializeBlockSourceRetrieval(blockSource)
	lastSelectionTime := time.Now()
block, err := blockSourceRetriever.RetrieveBlock()
If err != nil || lastSelectionTime.Add(maybeSwitchSourceInterval).After(time.Now())
	blockSource = nil
	blockSourceRetriever = nil
	continue
~~~~