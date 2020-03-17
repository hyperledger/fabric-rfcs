---
layout: default
title: RFC Template
nav_order: 3
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
of a system channel, which improve the privacy and scalability of the ordering service and the network as a whole. 


# Motivation
[motivation]: #motivation

Creating channels by issuing a transaction to the system channel presents several disadvantages.

## Privacy problems
A system channel requires that all ordering service nodes (OSNs) be aware of all channels. As Fabric depricated the 
Kafka-based ordering service and moved to Raft (and later is planned to introduce BFT consensus), it is no longer 
desirable that an OSN that services a channel `A` with member organizations `X & Y` be aware of channel `B` with member 
organizations `Y & Z`. This creates a privacy problem, because the orderer organization (or organizations) now knows 
that `Y`, `X`'s partner to channel `A`, is also doing business with organization `Z`.  

There are workarounds that reduce the privacy problem, but not eliminate it. For example, using the system channel it 
is possible to create an application channel with only a single org, and then add additional orgs by updating the 
channel itself. However, these type of solutions are cumbersome and only partial.

## Scalability problems
All OSNs are members of the system channel, which creates a scalability problem in a large scale Fabric network. When 
the number of channels increases, Raft allows us do decouple the consenters sets of the different application channels, 
achieving linear horizontal scalability in the number of channels. That is, more resource can be added as the number of 
channels increase. However, the system channel is an exception: decoupling application channel consenters sets as 
described above will cause its number of members to increase, and hence its to performance to decrease. Joining and 
leaving channels without a system channel solves this problem.     
   
Moreover, in order start a new orderer and have it join an existing channel (perform 'on-boarding'), a new orderer 
starts by scanning the system channel in order to discover existing channels and their membership. As the number of 
channels increase, this scan prolongs the process of adding a new orderer.  
    
## Operational problems
There are several operational problems that affect large scale deployment and hinder efficient management of cloud 
environments using automatic tooling.
     
Channel creation (using the system channel) utilizes information contained in the system channel - e.g. MSP information 
of the organization - in order to apply it onto the newly created channel. This create the convoluted situation in 
which an organization that wants to update its details in the consortium must coordinate that action with third parties 
that have write privileges to the system channel.

There is no good way to list all the channels that an orderer is a member of. 

There is no good way to dispose of the resources associated with a channel, once an orderer is no longer a member of 
the consenters set of a that given channel.

In many cloud environments, it is necessary to restart an orderer in order for it to discover immediately that it has 
been added to a channel it was not a part of previously, because otherwise detection takes minutes. 

   
# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

We introduce a new mode of channel participation management that does not depend on the existence of a system channel.

We introduce the ability to start an OSN process without any channels. The OSN exposes a new **"channel participation" 
API** that allows the operator to join and leave channels on that specific OSN.   

A **"local orderer channel participation administrator"** is a role that is associated with the operation and 
administration of an OSN. The identity associated with this role is able to invoke a new **"channel participation" 
API**, and perform the following commands:

- **Join** this OSN to a new or an existing channel,
- **Remove** this OSN from an existing channel,
- **List** the channels this OSN is a member of.   

The new mode of channel participation management is meant to be incompatible with channel creation using a system 
channel, in the sense that if the system channel exists, the `Join` operation will not be supported. 
However, there is a transition path from a network operated with a system channel to a network that is operated without
one. In addition, it should be possible to start OSN processes without channels, create a system channel with the
channel participation API, and continue to operate the network in the usual way - creating channels through the system 
channel. "Mixed mode" management, that is, using the system channel to create channels on some OSNs and the 
participation API on other OSNs is not supported and highly discouraged.  

## Channel participation commands 
In this section channel participation commands are described in a high level. The commands are presented in REST style
but some details (i.e. exact formats, responses) are omitted for clarity.
   
### Joining a channel
A channel is joined providing the OSN with a config block:

  ```Join channel-name channel-config-block```
  
  or
  
  `POST /channels/channel-name` with body containing `channel-config-block`
  
The body contains a config block that contains all the details about the channel. 

If the OSN is joining a new channel, the config block is the genesis block of the channel, generated by the 
`configtxgen` tool, as described below. It looks like the first block of an 
application channel that is generated by the OSN, when a channel creation transaction is submitted to the system channel.

If the OSN is joining an existing channel, the config block is simply the last config block of the channel, retrieved 
from one of the other OSNs.

If the channel already exists, the call will fail.

If the system channel exists, the call will fail. 
          
The OSN receiving this call will inspect the block and conduct some sanity checks in order to make sure that the block 
is well-formed.

The OSN will try to find itself in the consenters set, and determine if it is a follower or a member of the cluster
executing consensus. 

If it is not in the set, it will continue as a **follower**, and start to pull blocks from other OSNs. It will continue
to do so until it catches up with the config block received in the invocation. From then on, it will continue to pull blocks, 
but will inspect each incoming config block and check whether it is included in the consenters set. If it is, it will 
start the runtime components that execute consensus, thus joining the cluster. 

If the OSN is in the consenters set, it will continue as **member**, first pulling blocks from other OSMs util it catches
up with the given config block, and then starting the runtime components that execute consensus, thus joining the cluster.

Note that joining an OSN as a member means that the channel was updated to include it in the consenters set before it
was joined by the participation API. Therefore, during the catchup process and until the new OSN starts executing 
consensus, the cluster is a state of reduced fault tolerance. For long chains this might be a problem, since catchup
can take a long time. The solution is to join the new OSN as a follower. After it catches up with the cluster the 
channel admin can update the channel config and include it in the consenters set.

The new OSN can pull blocks from other OSNs only if it has authorization to do so. A minimal requirement is that its 
owning org will be defined in the channel configuration, and that the new OSN will have a role of at least 
`/Channel/Member`.

#### Creating a channel genesis block
The way to create a channel genesis block is by using the `configtxgen`  tool, which already to supports the 
generation of an **application** channel genesis block.

```configtxgen -outputBlock channel_genesis_block.pb -profile SampleRaftChannelCreation -channelID my-app-channel```

Generating the channel genesis block follows the same rationale as generating the system channel genesis block, when 
bootstrapping a new Fabric network. 

The command refers to a profile `SampleRaftChannelCreation` in a `configtx.yaml` file, as usual. The YAML file 
will point to the file-system path of the MSP configuration of each org, for application as well as ordering orgs. 
In addition, it will point to the path of the certificates of the consenters. Generating a genesis block for an 
application channel means that:
- the `configtx.yaml` **does not** include a `Consortiums` config group
- the `configtx.yaml` **must include** a `Channel/Application` config group
 

The minimally required elements should include enough details for the OSN to achieve consensus and allow further 
configuration of all the config groups.           

#### Asynch operation 
Successfully completing the "Join" command means that the orderer will start a process of "on-boarding". 
The orderer will start a stage of state transfer in which it will fetch the ledger from other consenters, as described 
above. If it is member of the cluster, it will join it and participate in the consensus protocol on new blocks. 
Note that misconfiguration of the channel, networking problems, or failures of other nodes may mean that the 
"on-boarding" process might not be able to complete or be slow. However, that is not indicated by the response to the 
call but rather by invoking a "List" command on the channel, see below. 

### Removing a channel
In order to remove a channel from an OSN the local orderer channel participation admin should invoke:

```Delete channel-name  options ```

or

```DELETE /channels/channel-name``` with body containing ```options``` JSON object. 

If the operation succeeds, the runtime resources associated with this channel in the orderer will be terminated. The 
`options.renoveStorage` flag will determine whether the storage resources associated with channel will be removed 
or left intact.  

It is recommended that the channel administrators first update the channel configuration and remove the target OSN from 
the consenters set, and only then let the local orderer channel participation admin remove it from the OSN. However, it
should be possible to remove a channel from an OSN even though the target OSN was not removed from its channel config. 
It is the responsibility of the channel administrator and participation admin to make sure they do not deprive the 
channel's consensus protocol from the ability to reach a quorum.    

It should be possible to remove a channel from an OSN and join or create it again later.

When the system channel exists, it should be possible to delete the system-channel, but not any other channel.   

### Listing the channels
Listing the channels serviced by a given OSN is achieved by invoking

```List *``` 

or

```GET /channels```

The response is a list of pairs, where each pair contains the channel name and an informational object in JSON format. 

The informational object contains (at least):
- the on-boarding status of a channel: catching up, following (past catchup), or complete (running consensus).
- whether the node is a follower or a member of the cluster
- the height of the ledger
- ...

Discovering whether a channel exists, and what is its status is achieved by invoking 

```List channel-name```

or 

```GET /channels/channel-name```

The response will be successful if the channel exists, and fail otherwise. If successful, it will return the 
informational object described above.

- In case an OSN is joined as a follower, invoking "List channel-name"   
  conveys the information of whether this OSN had finished catching up with the cluster. 
- When the OSN is a member of the cluster, following the height can help detect connectivity and configuration problems. 

## Security
The channel participation API adopts the style of the "operations" API in the orderer. The local orderer channel 
participation admin is not identified by the MSP, but rather through mutual TLS. Like in the operations API that handles 
log-spec, the orderer.yaml file specifies:
- whether the channel participation API is enabled
- whether security is enabled for channel participation
- the client root CAs that are authorized to invoke channel participation commands

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
- These entities agree on a basic configuration for the channel, as expressed in the `configtx.yaml` file.
- A single entity generates the channel genesis block, and all the entities inspect and agree that the block is correct.
- All local channel participation admins (one for each ordering org) create the channel on all their respective OSNs by
  invoking the "Join" command with said genesis block.
- Channel admins (as defined by MSP config in said genesis block) can now join peers to the channel, and continue to 
  configure the channel as usual.  
- The mechanism for coordination and agreement on the genesis block is out of the scope of this RFC. It is the 
  responsibility of the channel admins and participation admins to supply the same genesis block to the "Join" command.

### Adding a new OSN to an existing channel with a long chain
When the number of blocks in the ledger is large, it is recommended to join the node as a follower, wait for ctach-up,
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
- If the options parameter indicates not to remove the channel storage, the orderer 

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
  participation admins. This is achieved by setting the respective fields in the orderer.yaml file of each OSN.
- These OSNs may belong to different organizations.
- Generate (using configtxgen, as usual), the genesis block of a system channel. (The system channel contains the 
  `Consortiums` config group, whereas application channels do not).
- Create the system channel by invoking the "Join" command as described above, but supply the genesis block of the 
  system channel, rather than that of an application channel.
- The channel participation command "Join" will no longer accept commands, and channel creation can only 
  be done using a config transaction on the system channel.
- The channel participation commands "List" and "Delete" (of system channel only) will continue to function.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient
detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and
explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this? Are there any risks that must be considered along with
this rfc. 

# Rationale and alternatives
[alternatives]: #alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For consensus, global state, transaction processors, and smart contracts
  implementation proposals: Does this feature exists in other distributed
  ledgers and what experience have their communities had?
- For community proposals: Is this done by some other community and what were
  their experiences with it?
- For other teams: What lessons can we learn from what other communities have
  done here?
- Papers: Are there any published papers or great posts that discuss this? If
  you have some relevant papers to refer to, this can serve as a more detailed
  theoretical background.

This section is intended to encourage you as an author to think about the
lessons from other distributed ledgers, provide readers of your RFC with
a fuller picture.  If there is no prior art, that is fine - your ideas are
interesting to us whether they are brand new or if it is an adaptation.

Note that while precedent set by other distributed ledgers is some motivation,
it does not on its own motivate an RFC.  Please also take into consideration
that Fabric sometimes intentionally diverges from common distributed
ledger/blockchain features.

# Testing
[testing]: #testing

- What kinds of test development and execution will be required in order
to validate this proposal, beyond the usual mandatory unit tests?
- List integration test scenarios which will outline correctness of proposed functionality.

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

- Describe all dependencies that this proposal might have on other RFCs, known JIRA issues,
Hyperledger Fabric components.  Dependencies upon RFCs or issues should be recorded as 
links in the proposals issue itself.

- List down related RFCs proposals that depend upon current RFC, and likewise make sure 
they are also linked to this RFC.

# Unresolved questions
[unresolved]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process
  before this gets merged?
- What parts of the design do you expect to resolve through the implementation
  of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be
  addressed in the future independently of the solution that comes out of this
  RFC?
