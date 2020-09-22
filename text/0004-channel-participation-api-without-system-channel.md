---
layout: default
title: channel participation api without system channel
parent: RFCs
---

- Feature Name: channel_participation_api_without_system_channel
- Start Date: 2020-03-01
- RFC PR: (leave this empty)
- Fabric Component: orderer
- Fabric Issue: (leave this empty)

# Summary
[summary]: #summary

Currently in order to create a channel one needs to issue a config transaction to the system channel. This has
privacy, scalability and operational disadvantages, since the system channel (and the orderers running it) is aware of
all channels and of all channel members (at creation time). In this feature we propose to expose a
"channel participation" API that would allow a local orderer administrator to join and leave a channel, as well as
to list all the channels that the local orderer is part of. This allows a Fabric network to be operated without the use
of a system channel, which improves the privacy and scalability of the ordering service and the network as a whole.


# Motivation
[motivation]: #motivation

Creating channels by issuing a transaction to the system channel presents several disadvantages.

## Privacy problems
A system channel requires that all ordering service nodes (OSNs) be aware of all channels. As Fabric deprecated the
Kafka-based ordering service and moved to Raft, it is no longer
desirable that an OSN that services a channel `A` with member organizations `X & Y` be aware of channel `B` with member
organizations `Y & Z`. This creates a privacy problem, because the orderer organization (or organizations) now knows
that `Y`, `X`'s partner to channel `A`, is also doing business with organization `Z`.  

There are workarounds that reduce the privacy problem, but not eliminate it. For example, using the system channel it
is possible to create an application channel with only a single org, and then add additional orgs by updating the
channel itself. However, these type of solutions are cumbersome and only partial.

## Scalability problems
All OSNs are members of the system channel, which creates a scalability problem in a large scale Fabric network. When
the number of channels increases, Raft allows us to decouple the consenters sets of the different application channels,
achieving linear horizontal scalability in the number of channels. That is, more resource can be added as the number of
channels increase. However, the system channel is an exception; decoupling application channel consenters sets as
described above will cause its number of members to increase, and hence its performance to decrease. Joining and
leaving channels without a system channel solves this problem.     

Moreover, in order to start a new orderer and have it join an existing channel (perform 'on-boarding'), a new orderer
starts by scanning the system channel in order to discover existing channels and their membership. As the number of
channels increase, this scan prolongs the process of adding a new orderer.  

## Operational problems
There are several operational problems that affect large scale deployment and hinder efficient management of cloud
environments using automatic tooling.

* Channel creation (using the system channel) utilizes information contained in the system channel - e.g. MSP information
of the organization - in order to apply it to the newly created channel. This creates the convoluted situation in
which an organization that wants to update its details in the consortium must coordinate that action with third parties
that have write privileges to the system channel.

* There is no good way to list all the channels that an orderer is a member of.

* There is no good way to dispose of the resources associated with a channel, once an orderer is no longer a member of
the consenters set of a that given channel.

* In many cloud environments, it is necessary to restart an orderer in order for it to discover immediately that it has
been added to a channel it was not a part of previously, because otherwise detection takes minutes.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

We introduce a new mode of channel management that does not depend on the existence of a system channel.

We introduce the ability to start an OSN process without any channels. The OSN exposes a new **"channel participation"
API** that allows the operator to join and leave channels on that specific OSN.   

A **"local orderer channel participation administrator"** is a role that is associated with the operation and
administration of an OSN. The identity associated with this role is able to invoke a new **"channel participation"
API**, and perform the following commands:

- **Join** this OSN to a new or an existing channel,
- **Remove** this OSN from an existing channel,
- **List** the channels this OSN is a member of.   

The new mode of channel participation management is meant to be incompatible with channel creation using a system
channel, in the sense that if the system channel exists, the `Join` and `Remove` operations will not be supported.

However, there are two transition paths:
 1. A transition path from a network operated with a system channel to a network that is operated without
one. That is achieved by removing the system channel from all OSNs.
 2. It should be possible to start OSN processes without channels, create a system channel with the
channel participation API, and continue to operate the network in the usual way - creating channels through the system
channel.

"Mixed mode" management, that is, using the system channel to create channels on some OSNs and the
participation API to create channels on other OSNs is not supported and highly discouraged.  

## Channel participation commands
In this section channel participation commands are described in a high level. The commands are presented in REST style
but some details (i.e. exact formats, responses) are omitted for clarity.

### Joining a channel
A channel is joined by providing the OSN with a config block:

  ```Join channel-name channel-config-block```

  or

  `POST /channels` with body containing `channel-config-block`

The body contains a config block that contains all the details about the channel. The `channel-name` is extracted from 
the block. 

If the OSN is joining a new channel, the config block is the genesis block of the channel. It looks like the first
block of an application channel that is generated by the OSN, when a channel creation transaction is submitted to the
system channel.

If the OSN is joining an existing channel, the config block is simply the last config block of the channel, retrieved
from one of the other OSNs.

If the channel already exists, the call will fail.

If the system channel exists, the call will fail.

The OSN receiving this call will inspect the block and conduct some sanity checks in order to make sure that the block
is well-formed.

The OSN will try to find itself in the consenters set, and determine if it is a follower or a member of the cluster
executing consensus (see [definition below](#definition-of-cluster-member-and-cluster-follower)).

If the call succeeds, the response includes a JSON informational object that describes the channel 
(see ["Listing the channels", below](#listing-the-channels)). 

#### Definition of cluster member and cluster follower
* A cluster **follower** is an OSN that is not defined in the channel's consenters set and does not take part in
  consensus. It just pulls blocks from other OSNs (which are members of the cluster).  
* A cluster **member** OSN is defined in the channel's consenters set and takes part in consensus.

Configuration changes can cause an OSN to switch between the follower and member roles.

* A follower evaluates incoming config blocks and checks whether it was added to the consenters set. If it was added,
  it starts the runtime components that execute consensus, thus becoming a member the cluster.
* A member evaluates incoming config blocks and checks whether it was removed from the consenters set. If it was
  removed, it stops the runtime components that execute consensus, but continues to operate as a cluster follower.

An OSN operating as a cluster follower is a new feature introduced by this work.

#### Two ways to join     
When an OSN joins a channel as a **follower**, it start to pull blocks from other OSNs. It will continue
to do so until it **catches up** with the config block received in the invocation. From then on, it will continue to pull
blocks, but will inspect each incoming config block and check whether it is included in the consenters set.
When channel admins add it to the consenters set, it will detect that and start the runtime components that execute
consensus, thus becoming a member the cluster.

When an OSN joins a channel as a **member**, it first pulls blocks from other OSNs until it **catches
up** with the given config block, and then starts the runtime components that execute consensus, thus joining the cluster.

Note that joining an OSN as a member means that the channel was updated to include that OSN in the consenters set before it
was joined by the participation API. Therefore, during the catch-up process and until the new OSN starts executing
consensus, the cluster is in a state of reduced fault tolerance. For long chains this might be a problem, since catch-up
can take a long time. The solution is to join the new OSN as a follower. After it catches up with the cluster the
channel admin can update the channel config and include it in the consenters set.

The new OSN can pull blocks from other OSNs only if it has authorization to do so. A minimal requirement is that its
owning organization will be defined in the channel configuration, and that the signature of the new OSN on the `Deliver`
request satisfies the `/Channel/Readers` policy.

#### Creating an application channel genesis block
There are several ways to create an application channel genesis block. This could be done programmatically or using
tools (e.g. `configtxgen`). The application channel genesis block is simply a configuration block, with the
configuration following this structure:
* The `Consortiums` config group must not be present;
* The `Orderer` config group must be included, and should contain a full definition of the ordering service;
* The `Application` config group must be included, and should contain the definitions of the organizations making up
the the channel.

In contrast to the config update transaction that creates a channel using the system channel, here there is no
template for the `Orderer` group; thus the ordering service definition must be included in full. Moreover, there is no
reference to a `Consortium` definition that resides in the `Consortiums` map; therefore the definition of each
organization must be included in full in the `Application` group.

For a full description of what needs to be included in each config group see:
[Channel Configuration (configtx)](https://hyperledger-fabric.readthedocs.io/en/release-2.0/configtx.html).

##### Example: using `configtxgen`
One way to create a channel genesis block is by using the `configtxgen` tool, which already supports the
generation of an **application** channel genesis block.

```configtxgen -outputBlock channel_genesis_block.pb -profile SampleRaftChannelCreation -channelID my-app-channel```

The command refers to a profile `SampleRaftChannelCreation` in a `configtx.yaml` file, as usual. The YAML file
will point to the file-system path of the MSP configuration of each org, for application as well as ordering orgs.
In addition, it will point to the path of the certificates of the consenters. Generating a genesis block for an
application channel means that:
- the `configtx.yaml` **does not** include a `Channel/Consortiums` config group
- the `configtx.yaml` **must include** a `Channel/Application` config group

#### Asynch operation
Successfully completing the "Join" command means that the orderer will start a process of "on-boarding".
The orderer will start a stage of state transfer in which it will fetch the ledger from other consenters, as described
above. If it is a member of the cluster, it will join it and participate in the consensus protocol on new blocks.
Note that misconfiguration of the channel, networking problems, or failures of other nodes may mean that the
"on-boarding" process might not be able to complete or be slow. However, that is not indicated by the response to the
call but rather by invoking a "List" command on the channel, see below.

### Removing a channel
In order to remove a channel from an OSN the local orderer channel participation admin should invoke:

```Delete channel-name```

or

```DELETE /channels/channel-name``` with an empty body.

If the operation succeeds, the runtime resources associated with this channel in the orderer will be terminated, and 
the storage resources associated with the channel will be removed.

It is recommended that the channel administrators first update the channel configuration and remove the target OSN from
the consenters set, and only then let the local orderer channel participation admin remove it from the OSN. However, it
should be possible to remove a channel from an OSN even though the target OSN was not removed from its channel config.
It is the responsibility of the channel administrator and participation admin to make sure they do not deprive the
channel's consensus protocol from the ability to reach a quorum.    

It should be possible to remove a channel from an OSN and join it again later.

When the system channel exists, it should be possible to delete the system-channel, but not any other channel.   

### Listing the channels
Listing the channels serviced by a given OSN is achieved by invoking

```List *```

or

```GET /channels```

The response contains a JSON structure with two fields: one contains an object that identifies the system channel, and 
is null if it does not exist, and the other is an array of objects, one for each application channel. The short 
descriptive objects contain a channel name and URL.


Retrieving further information about the channel is achieved by invoking

```List channel-name```

or

```GET /channels/channel-name```

The response will be successful if the channel exists, and fail otherwise. If successful, it will return a more detailed
JSON informational object that includes:
- the channel name 
- a relative resource URL that identifies the channel
- the cluster-relation of the channel: whether it is a `member` or a `follower`
- status of the channel: whether it is `onboarding` (catching-up towards the join-block) or `active` (past the join-block)
- ledger height

In case an OSN is joined as a follower, invoking "List channel-name" conveys the information of whether this OSN had 
finished catching up with the cluster. This can be the sign for the channel admin to add the OSN to the channel's 
consenters set. When the OSN is a member of the cluster, following the height can help detect connectivity and 
configuration problems. 

This informational object is included in the response to a successful "Join".  

The `List` command works also _when the system channel exists_, and on any consensus type. Therefore, to support that:
- In an `etcdraft` OSN:
  - If the channel is serviced by this OSN, cluster-relation is `member` and status is `active`; and 
  - If the channel is not serviced by this OSN, cluster-relation is `config-tracker` and status is `inactive`.
- In a `kafka` or `solo` OSN, cluster-relation is `none` and the status is `active`.

## Security
The channel participation API adopts the style of the
["Operations" API]((https://hyperledger-fabric.readthedocs.io/en/release-2.0/operations_service.html#operations-security))
in the orderer. The _local orderer channel participation admin_ is not identified by the MSP, but rather through mutual TLS.
Like in the operations API that handles log-spec, the `orderer.yaml` file specifies:
- whether the channel participation API is enabled
- whether security is enabled for channel participation
- a listening address (host:port)
- the server certificate and private key
- the client root CAs that are authorized to invoke channel participation commands

In order to separate the roles of "operations admin" and "local orderer channel participation admin" we intend to expose a 
new endpoint (host:port) that is dedicated to Fabric administrative tasks, like the channel participation API.
This means that the orderer configuration, as expressed in the `orederer.yaml` file, will identify endpoint 
configuration parameters that are unique to the channel participation API.    

It is recognized that the mutual-TLS mechanism is less than ideal and presents difficulties when web gRPC and SSL
termination are involved. In anticipation of an upgrade to this mechanism, the channel participation API will be
implemented is such a way that allows an easy upgrade to another mechanism of authentication and authorization.     

## Common operational flows

### Bootstrapping a new Fabric network without a system channel
- Start one or more OSNs without any channels.
- These OSNs should be configured to accept channel participation commands from their respective local orderer channel
  participation admins. This is achieved by setting the respective fields in the orderer.yaml file of each OSN.
- These OSNs may belong to different organizations.
- Create a new channel as described below.

### Adding a new channel to existing OSNs
- The admins of the entities that take part in the channel cooperate to collect cryptographic material. Those are the
  organizations that compose the application consortium, and the organizations that run the ordering service.
- These entities agree on a basic configuration for the channel.
- A single entity generates the channel genesis block, and all the entities inspect and agree that the block is correct.
- All local channel participation admins (one for each ordering org) create the channel on all their respective OSNs by
  invoking the "Join" command with said genesis block.
- Channel admins (as defined by MSP config in said genesis block) can now join peers to the channel, and continue to
  configure the channel as usual.  
- The mechanism for coordination and agreement on the genesis block is out of the scope of this RFC. It is the
  responsibility of the channel admins and participation admins to supply the same genesis block to the "Join" command.

### Adding a new OSN to an existing channel with a long chain
When the number of blocks in the ledger is large, it is recommended to join the node as a follower, wait for catch-up,
and only then add it to the consenters set.

- The channel admins get the last config block of the channel, and provide it to the channel participation admin of the
  new OSN. (These may be the same entity).
- The channel participation admin of the new OSN invokes the "Join" command with said config block.
- The channel participation admin checks that the new OSN had caught up with the cluster using the "List" command.
- The channel admins issue a config update transaction that adds the new OSN to the consenters set of the channel, and
  to the orderer endpoints. This introduces the new OSN to the existing OSNs and the peers.

### Adding a new OSN to an existing channel with a short chain
When the number of blocks in the ledger is small, it is possible to reverse the order described above, and join the
node as a member of the cluster.

- The channel admins issue a config update transaction that adds the new OSN to the consenters set of the channel, and
  to the orderer endpoints. This introduces the new OSN to the existing OSNs and the peers.
- The channel admins get the last config block of the channel, and provide it to the channel participation admin of the
  new OSN. (These may be the same entity).
- The channel participation admin of the new OSN invokes the "Join" command with said config block.

### Removing an OSN from an existing channel
- The channel admins issue a config update transaction that removes the target OSN from the consenters set of the
  channel, as well as from the orderer endpoints. Care must be taken to ensure that the remaining OSNs can still
  function and reach consensus.
- After removal from the consenters set the target OSN switches to "follower" mode and continues to pull blocks from
  the other orderers.
- The channel participation admin of the target OSN invokes the "Delete" command with target channel name.

### Transitioning to local channel participation management from system-channel-based channel creation
- Switch the system channel to maintenance mode, in order to stop channel creation TXs from coming in.
- Configure all OSNs to accept channel participation commands from their respective local orderer channel participation
  admins. This is done by changing the `orderer.yaml` and rebooting the OSN. This operation can be staggered, such
  that, depending on the fault tolerance setup of respective channels, no channel down-time is experienced.  
- This will allow OSNs to accept "List" commands and "Delete" of the system channel only.
- Delete the system channel from all OSNs.
- OSNs will now accept the full set of channel participation commands.
- It is the responsibility of all local orderer channel participation admins to delete the system channel from all OSNs.

### Creating a system channel using the channel participation API
- Start one or more OSNs without any channels.
- These OSNs should be configured to accept channel participation commands from their respective local orderer channel
  participation admins. This is achieved by setting the respective fields in the `orderer.yaml` file of each OSN.
- These OSNs may belong to different organizations.
- Generate the **genesis block** of a system channel. (The system channel contains the `Consortiums` config group, whereas
  application channels do not).
- Create the system channel by invoking the "Join" command as described above, but supply the genesis block of the
  system channel, rather than that of an application channel.
- The OSN must be restarted.
- After restart, the channel participation API will no longer accept "Join" commands, and channel creation can only
  be done using a config transaction on the system channel.
- The channel participation commands "List" and "Delete" (of system channel only) will continue to function.

A variant of the flow described above can be used to add an OSN without any channels to a network that already has a 
system channel and application channels.
- Start an OSN without any channels.
- This OSNs should be configured to accept channel participation commands from their respective local orderer channel 
  participation admins.
- Get the last config block of the system channel from an existing OSN.
- Create the system channel by invoking the "Join" command as described above, but supply the last config block of the
  system channel, rather than the genesis block.
- The OSN must be restarted.
- After restart the OSN will on-board the system channel and existing application channels, just as if it was supplied 
  with a system channel bootstrap block of the same height. 

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The implementation of the channel participation API can be split in two. One part deals with the "API layer" and
deals with exposing the REST API via an HTTP server. The second part receives abstract channel participation commands
via a Go interface, and executed the operations described above.  

## The API layer
The peer and the orderer host an HTTP server that offers a RESTful “operations” API (see:
[The Operations Service](https://hyperledger-fabric.readthedocs.io/en/release-2.0/operations_service.html)).

We intend to provide an additional HTTP server that is hosted only the orderer, and expose a new capability
`/participation/v1`, which has a single resource  `/channels` as described above.
The channel participation commands will only be enabled if specified explicitly in the `orderer.yaml`
configuration file. These commands will only be available in the orderer.

The _local orderer participation admin_  role will have separate authentication and authorization credentials 
than the _operations admin_ (i.e. one can provide different TLS certificates for channel participation and operations). 
We choose this option in order to clearly separate said roles, since the _local orderer participation admin_ 
is capable of very intrusive and potentially destructive actions.

## Participation management
We intend to implement the channel participation management functionality in a new package
(`orderer/common/channelparticipation`). In this package the handlers for the respective REST calls will be defined.
The existing `orderer/common/multichannel/registrar.go` will keep a map of channel names and their status,
and will be in charge of coordinating the operations of following or joining a channel as a member.
Every channel that needs to follow or catch-up on the cluster will have a dedicated goroutine. The functionality of 
"following" a channel will be implemented in a new package `orderer/common/follower`. When the channel reaches a state 
where it can join the cluster, the respective methods in `orderer/common/multichannel/registrar.go` will be invoked in 
order to start the chain runtime.

When a channel is first joined with a `join-config-block`, the config block is atomically saved into a dedicated file 
repository. The existence of this file is a signal that the join process is
active, and should be resumed after restart, even if the orderer crashes or is terminated. 
This block will be used as a "bootstrap" for pulling blocks from other OSNs, and its height
an indication for when catch-up is complete. If the OSN is a member of the cluster when catch-up
is complete, the node will start the chain, otherwise it will continue to follow and check
incoming config blocks. When the node joins the cluster and starts the respective chain run-time component,
it will delete the `join-config-block` from the file repository.

The process of a join includes fetching blocks from other OSNs. For that purpose we will reuse code from
`orderer/common/cluster`, in particular the method that pulls a single named channel
`Replicator.PullChannel(channel string)`.

Deleting a channel is achieved by first terminating the runtime components of the channel and then removing the 
channel's storage resources. The removal of storage resources should be resistant to crashes, and continue across 
restarts. 

Listing the channel or channels is achieved by querying the channel map in the `.../multichannel/registrar.go` 
implementation.

The interface of executing channel participation commands is expected to stay stable even if the authentication and
authorization mechanisms change, or even if this API is separated on to a different service endpoint.

## Boot sequence of Orderer
In the boot sequence of the orderer (`orderer/server/main.go`) we will have to check whether the orderer is working
with or without the system channel, and then start either the current on-boarding code (eventually calling
`orderer/common/cluster/Replicator`) or the new channel participation management sub-component that implements  
the joining and following go routines for each respective channel.

In the initialization sequence of the `.../multichannel/registrar.go` we will have to reconcile the existence of 
ledger folders with the existence of the `join-config-block` in the file repository. 

# Drawbacks
[drawbacks]: #drawbacks

The main drawback is that, in the absence of a central synchronization point (system-channel), channel creation now
requires the coordinated effort of multiple local orderer participation admins. This exposes several risks.

## Genesis block divergence
In the proposed design the onus of providing identical channel genesis blocks to the OSNs is upon the participation
admins. In a Raft based orderer there are no explicit mechanisms to protect against different orderers on the same
channel starting with different genesis blocks, and depending on the nature of the difference between these blocks,
various failure scenarios are possible.

We propose to deal with this risk by developing mechanisms to detect and
prevent this situation in a separate RFC (see below).

## Mixed mode operation
Given a network operated with a system channel, an operator starts an empty orderer, gets the last config block from
an existing channel, and joins the orderer to it. There is no existing mechanism that prevents that. Mixed mode
operation can break existing on-boarding code in unpredictable ways.

We propose to deal with this risk by clearly documenting that this is not supported and highly discouraged.


# Rationale and alternatives
[alternatives]: #alternatives

## Rationale
The proposed design completely decouples application channels from the system channel and thus, breaks the scalability
barrier. Each channel can hence be deployed on a different set of orderers, achieving linear scalability in the number
of channels. This design also solves the privacy problem inherent in a shared system channel, and simplifies
admin operations.

## Alternatives
An alternative design is to keep the system channel running on small cluster, and drop the requirement of app-channel
OSNs to be a member of that cluster. OSNs belonging to app-channels follow the system channel and fetch blocks like
a peer does.

This solves the scalability of channels, but not the privacy problem. Some of the operational problems also remain,
e.g. slow on-boarding time of new OSNs existing channels.

# Prior art
[prior-art]: #prior-art

N/A

# Testing
[testing]: #testing

## Unit tests
Every new class of the new participation API will be covered by new unit tests.

## Integration tests
Integration tests will cover the following scenarios:
1. Bootstrap:
   - starting a set of empty OSNs
   - joining all to an app-channel
2. Add new channel:
   - starting with a set of OSNs with a channel
   - add another channel to a subset of OSNs (Join with genesis-block)
3. Add new OSN:
   - start with a set of OSNs, with a channel, with blocks
   - add the new OSN to the channel (join with last-config-block)
   - as a follower (not in consensus set), adding it after catch-up
   - as a member (in consensus set)
4. Remove an OSN from a channel:
   - start with a set of OSNs, with a channel, with blocks
   - add the new OSN to the channel
        - as a follower
        - as a member
5. Bootstrap to system channel:
   - starting a set of empty OSNs
   - joining all to a system-channel
   - create a channel
   - make sure Join and Delete (app-channel) are disabled
5. Transition from system-channel to local participation:
   - start from a network with a system channel and app channel
   - remove system-channel
   - add channel with participation API


# Dependencies
[dependencies]: #dependencies

## Existing items
This Jira epic governs the development of the features defined in this RFC:

- https://jira.hyperledger.org/browse/FAB-17712

## Authorization and Authentication
There is an ongoing effort to improve the AA mechanisms of the "operations" API.
When that work is completed, and implementation starts, the participation API would be impacted as well.
However, the bulk of the implementation, which resides "below" the API layer, is not going to be affected.   

# Unresolved questions
[unresolved]: #unresolved-questions

## Protection against genesis block divergence

In the proposed design the onus of providing identical channel genesis blocks to the OSNs is upon the participation
admins. In a Raft based orderer there are no explicit mechanisms to protect against different orderers on the same
channel starting with different genesis blocks, and depending on the nature of the difference between these blocks,
various failure scenarios are possible. For example:

* Genesis blocks that are identical in the sections that allow a cluster to form, but diverge in a
section that is not essential for the ordering service. The Raft cluster would form, a leader would be selected. The
next block coming from whoever happens to be the leader will have a hash that is inconsistent with the previous block
on all nodes that have a different genesis block. This would not be detected at the ordering service, but would crash
the peers that pull blocks from any OSN with a different genesis block than the leader.  
* Genesis blocks in which the certificate of a single OSN is corrupt in some blocks. This would lead to asymmetric
connectivity patterns in which some nodes can connect with that OSN and some do not, while being able to communicate
with each other. This can lead to reduced fault tolerance or complete loss of quorum that is hard to detect.  

One consequence of introducing the participation API is that some **mechanisms for protection against genesis block
divergence should be introduced**. Below are a few possible directions.

* Intercepting the Raft leader "AppendEntries" messages at the followers, if and only if the height of the Raft node
is 1 (only has a genesis block). This is very cheap since it will only happen once and never again.

* Have the cluster services use some derived unique name instead of the channel ID for routing of messages -- that way
two nodes only ever talk if they are on the same channel, and have the same genesis block. for example, the unique name
can be the channelName-hex.EncodeString(Hash(GenesisBlock))

Further investigation is needed in order to choose and design the right solution. This part is expected to be handled
in a different RFC.
