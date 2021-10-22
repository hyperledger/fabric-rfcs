---
layout: default
title: Orderer ledger snapshots
nav_order: 3
---

- Feature Name: OSN_snapshots
- Start Date: 2021-03-23
- RFC PR:
- Fabric Component: orderer, ledger
- Fabric Issue:

# Summary
[summary]: #summary

The RFC introduces ledger snapshot feature at orderers, similar to the same feature at peers.

# Motivation
[motivation]: #motivation

Real world use cases show the necessity of controlling the disk space consumed by block ledger, and this RFC is mainly motivated by the need to reduce space consumption at orderer nodes.
Additionally this RFC presents a better way to join a channel without pulling the huge number of all historical blocks, which also may take significant time and may pose a problem for scaling of an existing Fabric network.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC introduces a notion of a **ledger snapshot for OSNs** (ordering service nodes). With this notion two new logical operations appear at the orderer REST API and in the CLI tool *osnadmin*:
- Create a snapshot,
- Join channel from a snapshot.

Ledger snapshot is a set of files containing all the necessary information for OSN to join a channel from an arbitrary block in the chain without loss of any functionality except the ability to deliver an earlier block.

A **local orderer administrator** is a role that was introduced in RFC-0004 "Channel participation API without system channel" and is currently implemented with the help of "Admin Configuration" section in the OSN YAML config file (<Config.Admin.TLS>). Only the identity associated with this role is able to invoke the API defined in this RFC, that is existing **channel participation API** and newly defined **ledger management API**.

## Guide: Create snapshot

To create a ledger snapshot use the following REST API (optional block_number should be supplied as a parameter in the request body):

    POST /ledger/v1/{channelID}/snapshot

 or the following CLI command:

    $ osnadmin ledger snapshot -c <channelID> [-b <block_number>]

If the optional parameter <block_number> is omitted, snapshot will be created from the latest block, i.e. from the block number equal to ledgerHeight-1.

A process of snapshot creation will start immediately if the block number is omitted, or if the block number is specified and is less than current ledger height. If the block number is specified and is in future (>= ledger height) then the operation will immediately fail with a reasonable error message.

The process of snapshot creation is a synchronous one; upon the successful completion of the "ledger snapshot" API call the snapshot itself is ready and can be accessed in the following directory in the file system:
```
<Config.FileLedger.snapshots.rootDir>/completed/<channelID>/<block_number>
```

## Guide: Join channel from a snapshot

To join a channel from a snapshot use the following REST API (files from the snapshot must be placed in the body, using multipart/form-data content-type):

    POST /participation/v1/joinbysnapshot

 or the following CLI command:

    $ osnadmin channel joinbysnapshot -s <snapshot_dir>

There is no need to specify {channelID} when using the above API, as it will be extracted from the snapshot content.

To check the status of the "join by snapshot" operation use the following REST API:

    GET /participation/v1/channels/{channelID}

 or the following CLI command:

    $ osnadmin channel list -c <channelID>

## Guide: Prune block ledger of an existing channel
[guide-prune-block-ledger-of-an-existing-channel]: #guide-prune-block-ledger-of-an-existing-channel

To prune an existing channel ledger do the following:
 - create a new snapshot at the desired block number (using "ledger snapshot" API),
 - remove the channel from the OSN (using "channel remove" API),
 - join the channel from the snapshot (using "channel joinbysnapshot" API).

This approach has some drawbacks, see [Drawbacks section](#drawbacks).

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Changes to the existing API

This RFC adds a new endpoint to the existing **channel participation API** to allow joining a channel by snapshot. Additionally this RFC defines new **ledger management API** and introduces a new endpoint of this API to allow snapshot creation. In future we plan to add at least one more endpoint to this API which will enable ledger pruning "on the fly" without interruption in OSN service to resolve the known [drawbacks](#drawbacks).

Technically we achieve the above by following these steps:
1. Introduce new REST API endpoint **/participation/v1/joinbysnapshot** while keeping all existing functionality of the channel participation API, that is keeping "join by join-block", "remove channel", and "list channel(s)" API endpoints.
2. Add new REST API handler for the ledger management API with prefix **/ledger/v1/**. Define a single endpoint of this API **/ledger/v1/{channelID}/snapshot** to allow snapshot creation.
3. Add "Ledger Management API Configuration" section in the OSN YAML config file.
4. Add support of two new snapshot operations to the CLI tool *osnadmin*, specifically the commands *osnadmin channel joinbysnapshot* and *osnadmin ledger snapshot*.

## Snapshot content

We propose the OSN snapshot to follow the same logical structure as peer snapshot, that is to comprise a directory with a number of files in it, one of the files being a metadata JSON file. But here the similarity ends, as the content of the files will be drastically different.
Here is the content of the OSN snapshot:
1. Snapshot version - to allow for the future extensions. Current version is 1.
2. Channel ID.
3. lastBlockNum - the number of the last pruned block.
4. last config block before lastBlockNum - this is required to verify the newly pulled blocks. The sequence number of the last config block must be lower or equal to lastBlockNum.
5. lastBlockHash - the hash of the last pruned block.
6. prevBlockHash - technically not required, but this will make internal structure BlockchainInfo complete.
7. ConsenterMetadata of the last pruned block (lastBlockNum)

Version 1 of the OSN snapshot described in this RFC defines three files as the content of the snapshot directory:
1. Last config block for the channel before lastBlockNum. Content: The block data with block metadata removed completely, in protobuf binary serialization format. Name: "lastConfig.block"
2. Value of OrdererBlockMetadata.ConsenterMetadata of the last pruned block (note: this is coming not from the last config block, but rather from the last pruned block, that is from the block with sequence number lastBlockNum). Content: ConsenterMetadata value in protobuf binary serialization format. Name: "ConsenterMetadata.pb"
3. Metadata file containing all the rest of the snapshot content. Name: "metadata.json"

Version 1 of the OSN snapshot defines the following JSON fields in the "metadata.json" file:
```json
{
  "version": 1,
  "channelID": string,
  "lastBlockNum": number,
  "lastBlockHash": string,
  "prevBlockHash": string | null, // null is valid only when lastBlockNum == 0
  "parts": [ // array of all the files in the snapshot
    "lastConfig.block",
    "ConsenterMetadata.pb"
  ]
}
```

Unlike peer snapshot the OSN snapshot does not contain state DB, private data configs, the list of TxIDs and commit hash - as the OSN is purposefully designed to be unaware of these. So these two kinds of snapshots (peer snapshot and OSN snapshot) bear resemblance only on the conceptual level, being completely different in the content.

## Future-compatibility of the "join by snapshot" API

1. REST API future-compatibility.
To maintain compatibility with the future modifications of the snapshot format, this RFC mandates the following:

    1.1. The POST request to **/participation/v1/joinbysnapshot** endpoint must use multipart/form-data content-type.

    1.2. First part of the multipart body must have the name "metadata.json" and must always contain "metadata.json" file.

    1.3. API handler must read the "metadata.json" part first, confirm the validity of the JSON format and extract the fields defined for the version 1. Additional fields may be present in future snapshot versions; the presence of the additional fields not defined in version 1 format should not prevent the process from successful completion.

    1.4. API handler checks the validity of all the metadata fields, ignoring the fields not defined in version 1. In case the validity check fails, the API handler must fail immediately with an appropriate error.

    1.5. After the successful check of the metadata, the handler searches for two more parts with the names "lastConfig.block" and "ConsenterMetadata.pb" containing the respective files. Additional body parts may be present in future snapshot versions; the presence of the additional body parts not defined in version 1 format should not prevent the process from successful completion.

    1.6. API handler checks the validity of the found "lastConfig.block" and "ConsenterMetadata.pb" files. In case the validity check fails, the API handler must fail immediately with an appropriate error. Otherwise it proceeds with the "join by snapshot" process.

2. CLI tool *osnadmin* future-compatibility.
When processing *osnadmin channel joinbysnapshot* command, the following checks must be performed:

    2.1. Firstly, the CLI tool should find the file "metadata.json" in the specified directory <snapshot_dir>.

    2.2. Next it must confirm the validity of the JSON format and the presence of all metadata fields defined in version 1. Additional fields may be present in future snapshot versions; the presence of the additional fields not defined in version 1 format should not prevent the process from successful completion.

    2.3. Next, the CLI tool checks the validity of all the metadata fields, ignoring the fields not defined in version 1. In particular all the files defined in "parts" object must be present. In case the validity check fails, the process must fail immediately with an appropriate error.

    2.4. Lastly, the CLI tool creates POST request to **/participation/v1/joinbysnapshot** endpoint and places all the files defined in the "parts" object of the metadata JSON into the body of the request according to the REST API specification. Name of each part must be the name of the file as defined in "parts" object of the metadata JSON. First part must be the "metadata.json" itself.

## Implementation considerations

When a channel is joined from a snapshot by "channel joinbysnapshot" command, we have to save the following info on the block storage:
1. Last config block from snapshot
2. BootstrappingSnapshotInfo

For BootstrappingSnapshotInfo persistence the implementation is already there, but for the last config block a new persistent unit of storage (a file in case of file-based ledger) should be created.
A special attention should be paid to the use case of **Deliver** service being asked to deliver a single block with the number equal to the block number of the last config block from a snapshot - in this case the delivery request must succeed by returning the last config block from the new persistent unit of storage (from a separate file) and not from the continuous block ledger. In general, the ledger storage interface should be aware of a possibility that aside from a continuous block ledger there is a special case of a single block representing a last config block from a snapshot which should also be successfully returned when requested by block number.
Additionally all the procedures inside the OSN codebase looking up the last config block in the OSN ledger storage should be modified to return the last config block from the snapshot in case the last config block and/or a last block is not found in the block ledger itself.

The channel bootstrapping from snapshot must be crash-consistent, that is if a crash takes place while bootstrapping a channel, upon restart the orderer, either the bootstrapped channel exists in the usable state or not at all. In particular this means that the creation of the above two pieces of information (last config block and BootstrappingSnapshotInfo) must be an atomic transaction, and there should be no possibility after a crash to have one of them but not another.
Possible implementations of such an atomic transaction may be keeping both pieces of information in the same file and creating the file atomically, or keeping them in separate files in the same directory and creating the directory atomically.

## System channel co-existence

The following API can work both with system channel and without it:
- "channel list" - existing API, keep existing behaviour

The following APIs can work only without system channel, will fail in presence of system channel:
- "channel joinbysnapshot"
- "channel remove" - existing API, keep existing behaviour
- "channel join" - existing API, keep existing behaviour
- "ledger snapshot"

## New channel creation ("genesis" snapshot)

A new channel can be created using the existing API "channel join" with a genesis block as a join-block.

Theoretically a "genesis" snapshot also can be constructed. For this a genesis block should be used as a last config block and lastBlockNum should be set to 0. This RFC does not define any specific tool capable of creation of a "genesis" snapshot, instead this RFC mandates that such a snapshot should be accepted as a valid one. In particular a snapshot should be considered as valid if prevBlockHash is omitted from the metadata JSON file or is set to "null", but only when lastBlockNum is equal to 0.

## A **follower** mode

Exactly as when using "channel join" API the orderer bootstrapping from a snapshot should start in the **follower** mode pulling the blocks without processing them until it finds a config block confirming its membership in the channel, unless it can be established that it is a member of the channel already (a member of the consenter set in case of RAFT).

# Drawbacks
[drawbacks]: #drawbacks

There are three main drawbacks in the proposed ledger pruning method (see [Guide: Prune block ledger of an existing channel](#guide-prune-block-ledger-of-an-existing-channel)):
- an interruption in the service of the orderer is required,
- this works only in the absence of system channel, because "channel joinbysnapshot" works only in the absence of system channel.

To resolve the above problems a new RFC will be issued introducing a separate **Prune** endpoint to the **ledger management API** which will allow to prune the ledger without stopping the service and will work both in presence and in absence of system channel.

## Admin risks

Allowing ledger pruning in OSNs introduces a new risk at the network level. It is possible to end up in a situation when all orderers are pruned at some point and the peer snapshot above that point is not accessible. In this situation it is impossible to bootstrap a new peer.
We believe that the benefits of the ability to manage disk space outweigh the possible risk; nevertheless the admins should apply extreme caution when pruning OSNs.
Possible solutions and the network admin level may be:
 - always keep at least one orderer un-pruned;
 - ensure the existence of a peer snapshot above the lowest pruning point of all orderers.

# Rationale and alternatives
[alternatives]: #alternatives

## Alternatives considered

- Implement only pruning without the snapshot functionality.
  This still requires storing some information about the history of the ledger, as a bare minimum we need the last config block and the hash of the last pruned block. So why should we invent a new mechanism to support pruning, we can as well use the snapshot format to store the required info.
- Store the information about the first known block in a new field (firstBlockNum) somewhere in the blockStore or fileLedger structure.
  Again, we already have a code implemented that stores this info in BootstrappingSnapshotInfo structure. It is better to rely on the already tested code to achieve the same functionality.

# Rationale

Strictly speaking to achieve the main goal - orderer ledger pruning - we don't need the snapshot functionality. But due to the implementation reasons the notion of snapshot arises in some or another way. For example we need to store somewhere the last config block and the number of the last pruned block (or the number of the first block existing in the pruned ledger). So we decided that it is better to rely on the already existing code supporting snapshots and slightly modify it to suit the needs of the OSN snapshot.

In addition to the above, joining a channel from a snapshot solves many problems with the existing join-block implementation from RFC-0004 "Channel participation API without system channel", namely:
- the need to pull potentially huge number of the historical blocks
- the fact that the config in the join-block may be outdated and not representing the latest config

# Prior art
[prior-art]: #prior-art

RFC-0000 "Ledger Checkpointing"

RFC-0004 "Channel participation API without system channel"

# Testing
[testing]: #testing

Integration tests will cover the following scenarios:

1. Snapshot generation
  - start with a set of OSNs, with a channel, with blocks
  - create a snapshot from the same non-genesis block on all the OSNs
  - verify that the snapshot generated across OSNs is exactly same
2. Bootstrap:
  - start with a set of empty OSNs
  - construct a "genesis" snapshot
  - join all to the application channel from the "genesis" snapshot
3. Add new channel:
  - start with a set of OSNs with a channel
  - construct a "genesis" snapshot for the second app channel
  - join a subset of OSNs to the second channel from the "genesis" snapshot
4. Add new OSN to the channel:
  - start with a set of OSNs, with a channel, with blocks
  - create a snapshot from some non-genesis block
  - add the new OSN to the channel from the snapshot:
    - as a follower (not in consensus set), adding it after catch-up
    - as a member (in consensus set)
5. Add new channel without genesis block:
  - start with a set of OSNs, with a channel, with blocks, with a non-genesis config block among them
  - create a snapshot from some block after the last config block
  - remove the OSN from channel
  - start a set of additional OSNs
  - add all OSNs to the channel from the created snapshot
6. Snapshot creation after bootstrapping:
  - start with a set of OSNs, with a channel, with blocks, with a non-genesis config block among them
  - create a snapshot from some block after the last config block
  - start a new OSN, no channels
  - new OSN joins the channel from the snapshot
  - add some blocks to the channel without a config block
  - trigger snapshot creation
  - check the correctness of the created snapshot
  - add a config block and some more blocks to the channel
  - trigger snapshot creation
  - check the correctness of the created snapshot
7. System channel restrictions:
  - start with a set of OSNs with the system channel
  - make sure that the following operations fail:
    - "channel joinbysnapshot"
    - "ledger snapshot"
  - create an app channel with blocks
  - make sure that the following operation succeeds:
    - "channel list"
  - remove system channel
  - make sure that two operations that previously failed now succeed
8. Failure during bootstrapping the OSN from a snapshot
  - start "channel joinbysnapshot" operation
  - kill the OSN during the bootstrap
  - restart the OSN
  - verify that the OSN can again join the channel with the same snapshot
  - verify that the OSN can again join the channel with a different snapshot

# Dependencies
[dependencies]: #dependencies

**RFC-0004 "Channel participation API without system channel"**

This RFC relies on the **channel participation API** introduced in RFC-0004 and extends it for the use case of joining a channel from a snapshot without breaking compatibility with the existing functionality of the channel participation API.

# Unresolved questions
[unresolved]: #unresolved-questions

## Questions to be resolved during the implementation

- Should we introduce a new return code to **Deliver** call when a block before the snapshot start is requested?

It appears that the current codebase already returns a reasonable error code “NOT_FOUND“ with the log record “Error reading from channel, cause was: NOT_FOUND“ which indicates that the channel is there, but the block was not found. Because of this we do not include a new error code for this particular use case in the RFC.

## Questions to be resolved in future RFCs

- The problem of channel divergence that was pointed out in RFC-0004 "Channel participation API without system channel" is still unresolved.

It is possible to join the channel with the same channelID by using the snapshots created from different config blocks with the same block number.

- How to resolve a situation when all the orderers are pruned and a peer tries to fetch a block below the pruning point?

The peer will receive an error code “NOT_FOUND“, but the problem is that it will not be able to proceed further and will be basically stuck at this point. This is something that has to be resolved on the network admin level and the exact resolution is unclear at this point.
