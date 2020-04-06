---
layout: default
title: Ledger Checkpointing
nav_order: 3
---

* Feature Name: ledger_checkpointing
* Start Date: 2020-04-22
* RFC PR: (leave this empty)
* Fabric Component: core, ledger
* Fabric Issue: (leave this empty)

# Summary

Currently, for a peer to join a channel and reach a certain height, it requires the peer to fetch and process all the blocks in the channel since the genesis block. This requires a network to store all the blocks in a channel forever. In this RFC, we propose a new feature namely, ledger checkpointing. In this feature, we allow for the construction of a snapshot of a channel at a particular block height. We also allow for a peer to join a channel by using a snapshot as a starting state, instead of starting from the genesis block.

# Motivation

The monotonically growing size of blockchains is a well-recognized problem.
Hyperledger Fabric also faces most of the issues caused by this growth in size. These issues include the following

* Continuously increasing storage demand for persisting blocks.
* Time and resource inefficiencies in transferring and processing the blocks for bootstrapping a new peer node.
* It is not possible to modify the validation and commit code without introducing a `capability`. Still, it requires us to maintain all the variations of validation and commit code. Historically, we managed the variations by introducing if-else statements. Later, we adopted another approach of maintaining all the previous versions of the validation code as-is and routing the control to the appropriate validation code based on the capability check at one place. We do not find this approach as well extendible or manageable in the long term. So, it is desired to have a mechanism that allows for removing the legacy code for validation and commit entirely and write it a fresh that may perform validation and commit in a different way without a guarantee of maintaining previous behavior
  
Ledger checkpointing feature introduced in this RFC directly solves the second problem mentioned above and paves the way for solving the other two problems.

In addition, ledger checkpointing provides the following benefits
* Ability to bootstrap a peer from latest channel configuration rather than the genesis block which may contain outdated configuration
* Ability to verify that the state of a channel across peers is the same

# Guide-level explanation

In this section, we first introduce a set of remote APIs for the management of snapshots. Follwoing this, we present a scenario to show the usage of these APIs.

* **SubmitSnapshotRequest (channelName string, height uint64) error**
  * Authorized User - An admin of the organization that owns the peer 
  * Accepts a request for the generation of a snapshot when the channel reaches a specified *height*
  * The request is persisted and survives a peer crash
  * Returns an error if
    * No such channel
    * The height of the channel is higher than the height specified in the API

* **PendingSnapshotRequests () (map[channelName][]height, error)**
  * Authorized User - An admin of the organization that owns the peer 
  * Returns the pending (or under processing) requests for snapshots

* **CancelSnapshotRequest (channelName string, height uint64) error**
  * Authorized User - An admin of the organization that owns the peer 
  * Cancels the previously submitted request
  * Returns an error if no such pending request
  
* **DeleteSnapshot (channelName string, height uint64) error**
  * Authorized User - An admin of the organization that owns the peer 
  * Deletes one of the previously constructed snapshots
  * This operation does not delete the metadata of the snapshot. This gives a peer the ability to sign the snapshot metadata for other orgs, even after the admin deletes the corrsponding snapshot (most likely, the historical snapshots) to reclaim the storage space.
  * Returns an error if no such snapshot
  
* **ListSnapshots (channelName string) ([]\*SnapshotInfo, error)**
  * Authorized User - An admin of any organization that participates in the channel
  * Returns the information about available snapshots or their metadata. `SnapshotInfo` is a protobuf message that includes the following information
    * `SnapshotMetadata` (a proto message, details in next API)
    * `OnlyMetadataAvailable` (bool)

* **SignSnapshotMetadata (channelName string, height uint64) (snapshotMetadataBytes []byte, signature []byte, err error)**
  * Authorized User - An admin of any organization that participates in the channel
  * Returns the bytes of the `SnapshotMetadata` protobuf and the peer signatures on these bytes. `SnapshotMetadata` contains following fields.
    * ChannelName
    * Height of the snapshot
    * Hash of the snapshot
    * BlockHash for last block in the snapshot
    * BlockCommitHash of the last block in the snapshot (if available, more details about availability in the section *Unresolved questions*)
  * Returns an error if no such snapshot
  * An admin can use this API in order to collect the signatures from peers of different orgs for a specified snapshot. This helps the admin to meet the org's requirement about the trust model before using a snapshot.

* **JoinChannel(snapshotPath string)** error
  * Authorized User - An admin of the organization that owns the peer 
  * Peer joins the channel via a snapshot
  * Returns an error if
    * The snapshot format version not recognized
    * Snapshot hash does not match

In addition, like other APIs, these APIs would return errors related to unauthorized access or internal processing errors.

Now, we describe one usecase as an example of how this feature is intended to be used for adding a new organization to an existing channel

Two orgs, Org1 and Org2, participate in a channel. At some later time, the admins of these two existing orgs need to add a new org, Org3, to the channel. The admins of Org1 and Org2 coordinate and submit an updated channel config transaction to add Org3 to the channel. After the commit of the new channel config transaction, the admins of the Org1 and Org2 coordinate for deciding a height for constructing a snapshot. Assume that when the admins coordinate, the channel height on the peers of Org1 and Org2 are 10,400 and 10,500 respectively. Both the admins decide to construct a snapshot as of block 11,000 for Org3 to bootstrap its peer and join the `channel`. Note that deciding a common height may be tricky, as it needs to take into account the expected block commit rate to the channel and the buffer time needed for the admins to submit a request for the snapshot to their respective peers. See more discussion on this aspect, and an alternate mechanism in the section *Drawbacks*

Next, both the admins invoke the API `SubmitSnapshotRequest` on their respective peers. When the peers of Org1 and Org2 reach the height 11,000, the peers construct the snapshot for the channel. Org1 admin finds the snapshot folder on its peer at the location `{snapshotsFolder}/{channelName}/{blockHeight}` and transfers this folder to the admin of Org3. The snapshot folder contains five files. The four files contains the snapshot data and the fifth file contains the metadata that also includes the hashes of the four data files.

The admin of Org3 verifies the sanity of the data files by independently computing the hashes of the snapshot files and by matching these with the hashes present in the metadata file. Also, the admin of Org3 invokes the `SignSnapshotMetadata` API on the peers of Org1 and Org2 and receives the signed metadata. The admin of Org3 verifies the signatures of peers and matches the metadata with the metadata in the snapshot. The Org3 admin, thus, confirms that the peers of Org1 and Org2 both produced the same snapshot and preserves the singed metadata for the record.

Finally, the Org3 admin invokes the `JoinChannel` API on its peer with the snapshot and the peer joins the channel with the state as of block height 11,000.

# Reference-level explanation

In this section, we first describe the data items that a point-in-time snapshot consists of.
We then describe how we construct a point-in-time snapshot and how a peer joins a channel using a snapshot.

## Definition: Point-in-time snapshot
A point-in-time snapshot of a channel at height `H` is defined as the state of the channel that
consists of the data items that derive the peer behavior for the two functions namely `Transaction simulation` and `Transaction validation`.
Further, a snapshot does not include the data items that are *only* needed for serving the queries related to historical data such as blocks, transaction envelopes, and previous versions of chaincode data below height `H`. In other words, a snapshot for a channel contains the data items sufficient enough to bootstrap a peer that can process and generate *future* transactions above height `H` but is not capable of answering queries on historical details about the channel.

In the following table, we list the data items that a snapshot at height `H` consists of. We briefly mention, without going into details, the usage of these data items in Transaction Simulation or Transaction Validation functions.
| Data Item | Usage | Additional Info |
| --------- | ----- |---------------- |
| Public state | <ul> <li> Tx simulation <li> Tx validation | <ul> <li>Latest versions as of Height `H` <li> Include <key, value, version, metadata>|
| Private data hashes | same as above | same as above|
| Private data state\*| Tx simulation | <ul> <li> Latest versions as of Height `H` <li>  Include <key, value, version> <li> Some data may be missing |
| All TxIDs between block 0 and block `H-1` | Tx validation | For detecting duplicate TxIDs |
| Collection Configs History between block 0 and block `H-1` | Tx Simulation | This controls the Tx simulation indirectly, via the following: <ul> <li>Requesting Reconciliation <li>Serving the Reconciliation requests|

The data items that are not included in the snapshot are the `blocks` and the historical versions of `private data` (recap, we maintain all versions of private data in the private data store corresponding to each block).

## Creating a point-in-time snapshot

As mentioned in the section *Guide-level explanation*, a peer generates a snapshot for a channel, at a specified height, in a folder `{snapshotsFolder}/{channelName}/{blockHeight}`. The snapshot folder would contain one file for each of the data items in the snapshot that is mentioned in the previous section except for the Private data state\*. In concrete terms, the snapshot folder would contain one file for each of the following items.
  - Public state
  - Private data hashes
  - TxIDs
  - Collection Config History
  
In addition, this snapshot folder would contain an additional file that would contain metadata for the snapshot in a JSON format. This metadata file will contain the following information.
- ChannelName
- Height
- BlockHash for the last block in the snapshot
- Hash of Public state
- Hash of Private data hashes
- Hash of TxIDs
- Hash of Collection Config History
- A representative hash of the snapshot, computed by combining all of the above
- BlockCommitHash for the last block in the snapshot
- Compression algorithm used for the data files

\*Additional notes on Private data state

- We do not include private data state in the hash of the snapshot. This is because, the private data may differ on each peer. Moreover, the private data hashes has been included in the hash of the snapshot, which are available on each peer and is a union of all private data state across peers.

- Though, the Private data state is one of the components of the snapshot, still we do not export this to a file in the snapshot folder. Instead, we use the existing reconciler on the bootstrapping peer for retrieving the applicable private data directly from other peers. As a recap, the private data reconciler is triggered by a list of missing data for a channel. The missing data information is fed to the reconciler as `<BlockNum, TxNum, ChaincodeName, CollectionName>`. The reconciler, then consults the point-in-time collection configuration in order to compute a list of potential peers that may have the required private data. In the snapshot, the private data hashes and the collection configuration history are sufficient to generate the required triggers for the reconciler. As an alternate, we considered exporting the private data state as well, like other data items, to a file. However, discussing the pro and cons of file-based approach v/s reconcilier-based approach here would shift the focus and hence, we discuss this thoroughly in the section *Rationale and alternatives*.

Below, we describe how we export each of the data items to the corresponding files in the snapshot.
 - TxIDs <br/>
    Starting with fabric version 2.0, we maintain entries in the block store index in the form of `TxID_BlkNum_TxNum`. We intend to scan this index for exporting all the TxIDs in their sort order. This index is an append-only index and hence and we can filter the TxIDs that are added after the snapshot height, by evaluating `BlkNum` portion of the index entry.

- Collection Config History <br/>
    Like TxIDs, the collection configuration history is maintained in an append-only store with the key for an entry being of the form `ChaincodeName_BlkNum`. Hence, a simple range scan over this store, and filtering the entries by evaluating the `BlkNum` portion, gives the desired collection config history, i.e., the history below the snapshot height. This store is expected to be relatively much smaller as the chaincode upgrade is not a very frequent transaction.
    
- Public state and Private data hashes state <br/>
    Typically, we refer to these two data items collectively as the world-state. We maintain this world-state in a dedicated database `statedb`. Fabric offers two options for the statedb i.e., leveldb and couch database. Unlike TxIDs and Collection Config History, in statedb, we maintain only latest versions of the keys from the world-state. In other words, the versions of the keys in the statedb keep changing by the commits of ongoing transactions. Also, these databases do not offer a built-in support for creating a point-in-time persisted snapshot. In the presence of the above two limitations, in order to retrieve a consistent snapshot as of specified height, we would *pause* the further block commits on the channel at the specified height until we export the Public state and Private data hashes to their corresponding files. For couch database, we will export the values in [canonical JSON format](http://gibson042.github.io/canonicaljson-spec/) so that the exported data is same across different peers. We considered alternate mechanisms for avoiding this pause in the block commit. We discuss these alternatives briefly in the section *Rationale and alternatives*

Each of the data files that we generate would contain the file format version in the first byte, followed by a deterministic byte stream representing the data. We do not list the full details of file formats for each of the data items here. However, for illustration, the simplest file format would be for the TxIDs, which would contain file_format_version followed by all the TxIDs in their sort order. 
<p/>
While exporting the data to the file, we would also compute the hash of the stream. We write this hash in the metadata file. Thus, the hash of the data item would be the same as the hash of the file. This property could be leveraged to verify the hash using OS commands i.e., outside the peer code. Also, note that we would provide options to write the files in the compressed format. However, if the compression is used, the hashes of the compressed files would be different from the hashes specified in the metadata file.

## Bootstrapping a peer with a point-in-time snapshot

As mentioned in the section *Guide-level explanation*, an admin can join a peer to a channel by using the *JoinChannel* API, supplying this with a snapshot folder. 

Before we list the steps that a peer would perform to join a channel, we like to highlight that at the implementation level, peer maintains two other data stores that we have not mentioned so far. These are, the `PurgeManager bookkeeper db` and the `Metadata hint db`. The former is used to keep track of expiring private data and expiring private data hashes. The latter is used for an optimization so that the chaincodes that do not use key-level endorsement does not pay the additional performance penalty. 

We did not mention these data stores before, because, they do not play a role in the generation of a snapshot. However, we mention these here, because, we add entries into these store during bootstrapping. Note that, the data entries for these stores are derived from one or more data items that are included in the snapshot.

Now, we list the steps that a peer performs during bootstrapping.

1. Read the TxIDs files and populate the block store index with TxIDs entries. The values of these entries will be a `nil`. In a regular block store index, the TxIDs entries point to file offset in one of the block files. This offset is expected to be the first byte of the corresponding transaction envelope (in order to serve query RetrieveTxByID) in the block file. However, this new peer will not have these previous blocks and would not serve such queries for previous blocks or transactions.

2. Load the snapshot metadata from the metadata file and maintain in the blockstore. Particularly, the BlockHash and the BlockCommitHash (if present) for the last block. These are used to verify and compute the commit hash respectively, when processing the the next block

3. Read the Collection Configuration History file and populate collection configuration history db

4. Read the Public state file and and perform the following
   1. Populate Public state portion of the statedb
   2. If Metadata is present in any of the key, set flag in the MetadataHint db for the corresponding chaincode

5. Read the Private data hashes file and perform the following
   1. Populate the Private data hashes portion of the statedb
   2. Populate the PurgeManager bookkeeping db for the KeyHash entries
   3. Populate the Private data store with entries for missing data for both eligible and ineligible collections as applicable. At this stage, we would populate all the entries for the private data as missing. These entries act as a trigger for the reconciler in a later step
   4. If Metadata is present in any of the keyHash, set flag in the MetadataHint db for the corresponding chaincode
   
6. Invoke the private data reconciliation function explicitly, in order to retrieve the missing private data. We would modify the reconciler code so that it can be invoked directly without any pause that we do during the normal operations. As a recap, the reconciler also populates the private state portion in the statedb and adds the appropriate entries in the PurgeManager bookkeeping db for the private data keys

7. If couch database is used as a statedb, create indexes for the chaincodes (and collections) that are already installed on the peer
   
8. Add an entry for the newly joined channel in the ledgermgmt db, open the new ledger, and construct the corresponding new channel object

### Additional Notes

1. The peer computes the hash of the data while loading the data into appropriate stores. After having read an entire file, the peer matches the hash of the file with the corresponding hash present in the metadata files. To make it simple, we can perform the hash check before loading the data. However, that cuases double the work as far as uncomression and file I/O is concerned. We list this in the section *Unresolved questions*.
   
2. The first four steps can be executed in parallel and the remaining steps are to be executed sequentially, because, they require the information processed in the previous steps. 

3. If a failure happens anytime during step 1 through 5, the peer would clean up (or madate the admin to do so) all the data that was partially loaded for the channel. Also, if the peer crashes during this period, upon restart, the peer would clean up the data. However, any crash after step 5, the peer would resume to finish the remaining work. We would maintain a simple progress marker to identify this transition point.

# Drawbacks

Following are the added drawbacks or risks that come with this feature that should be considered while evaluating this RFC

## Admins' responsibilities 

1. The scope of this RFC is to enable the peer to generates a channel snapshot and to bootstrap from a snapshot. However, many scenarios would require the generation of a snapshot across orgs at a same height of the channel. For these types of scenarios, we leave the coordination of agreeing on a common *future* height to the orgs' admins. After agreeing on a common height, admins are expected to invoke the `SubmitSnapshotRequest` API on their respective peers before their peers reach that height. A practical way of achieving this would be to pick the height of the peer that has maximum height among all and add a safety buffer number to it. This limitation can be overcome later, by introducing a special purpose transaction that can simplify this coordination up to some extent.
  
2. We generate the snapshot in the form of a set of files. We leave this to the orgs' admins to transfer the snapshot folder from one org to another, if needed - e.g., to a new org for joining the channel. Since the snapshot contains the application specific data, the admin would need to be cautious that they are sharing the data with the channel members only. If required in the future, this behavior can be changed by introducing the snapshot transfer mechanism within the peer and in that case the `JoinChannel` API will take the address of the source peer instead of the file path of the snapshot. Note that an admin need not be aware of private data collections and its memberships, as we still transfer the private data via peer's built-in reconciler.

3. Another element that we leave to orgs' admins is the trust model for the snapshot data. An admin of an org who receives a snapshot from a different org is expected to verify the content of the data files by computing the hashes afresh and then matching these with the hashes present in the metadata file. Also, the admin is expected to cross-check the hash of the snapshot with the sufficient numbers of other organization based on application data involved and the trust model.

## Performance impact

We pause the commit of blocks on the channel while exporting the snapshot data (in particular, the world-state data items). Also, the exporting the channel's data to files and computing the hashes is expected to generate significant burst in CPU and I/O usage, this is also expected to degrade the performance of other channels on the peer. So, we assume that in most of the networks,
 - The snapshot generation  is an infrequent activity and
 - The above mentioned performance behavior during the snapshot generation is acceptable

However, if the above two assumptions are not true in a network, we expect a dedicated peer would be hosted for snapshot generation.

## Bootstrapping peer from an much older snapshot

When the majority of the peers in a channel are bootstrapped from a snapshot, a new peer should not be bootstrapped from a snapshot that had been generated at a lower height. The risk in this environment relates to private data, which is that the peer bootstrapping form a lower-height snapshot may ask for private data that may not be available on the peers that were bootstrapped from a higher snapshot. In this scenario, though the peer would start with a large amount of missing private data, but the background reconciliation process may stuck asking for the same data over and over. Though, this may not be a usual scenario but we will document this limitation. This is a current limitation of the reconciler that we need to fix anyways.

# Rationale and alternatives

The rationale for the presented design in this RFC is to choose simplicity over efficiency. This is because, the checkpointing is expected to be an infrequent operation. Following are the alternatives that we considered, but eventually dropped, in the favor of a simpler design.

## Private data transfer
As mentioned in the section *Reference-level explanation*, we do not export the `Private data state` to a file in the snapshot folder. Instead we transfer private data directly from one of the existing peers to the bootstrapping peer, by leveraging existing reconciler. Like for the other data items, we considered exporting the private data to files. However, this causes additional complexity, primarily because unlike other data items, private data may differ on different peers. Further, the private data requires selective distribution to different orgs. Below, we highlight some of these issues in the context of snapshots. We divide the description in the two broad categories - (1) Sharing a snapshot across organization and (2) Sharing a snapshot within organization.

### Sharing a snapshot across organization
When the snapshot is shared across orgs, it presents challenges in distributing data via exported files. We list these challenges, in the increasing order of complexity, in the following scenarios.

#### When the newly bootstrapping org is not eligible for the private data
Typically, the new org would not be eligible for the existing private data as of snapshot generation. Because, most often, the existing orgs would generate a snapshot for the new org soon after the commit of the updated channel config transaction. The peer of the new org would learn about its eligibility for one or more collections, if any, when it would process further transactions. In this scenario, at the time of bootstrapping the peer, we would anyway be required to add the missing data entries for the entire private data, marking as an ineligible missing data. This scenario is no different from when an existing org is added to an existing collection.

#### When the newly bootstrapping org is eligible for some of the private data
Though, this may be a remote scenario, but the possibility cannot be ruled out that the existing orgs upgrade the collection configuration(s) before creating the snapshot. However, if we export entire private data into a single file, the file cannot be shared with the new org. Alternatively, we could export the private data in separate files for each combination of `<chaincode, collection>` along with the names of the eligible orgs (per latest collection config). However, this may require the new org admin to gather individual collection specific files from the multiple organizations. This requires the admins to understand collection configuration which are application (chaincode) level detail.

 #### General Scenario
This is not a practical usecase, still we mention this briefly, because the code complexity will eventually be of this order. In this most general scenario, an existing org bootstraps its peer from a snapshot that is generated by another existing org. This adds additional complexity, in addition to the ones that are already mentioned. Further, this additional complexity is not easy to solve outside the peer. This is caused by the fact that the bootstrapping peer may be eligible only for the part of the collection data. This is possible, if the org was a member of the collection in the past but was not a member at the time of snapshot generation.

### Sharing a snapshot within organization
Unlike the scenarios for snapshot sharing across orgs, in this scenario, there is no complexity of deciding which collection private data can be shared. Also, the exported private data files could be useful from efficiency point of view. However, it does not solve the code complexity at the bootstrapping peer in any manner. This is because
 - Anyways, we would be required to detect and add the missing data entries for ineligible collection
 - We would also need to do the same for the eligible collections because, some of the private data may be missing on the snapshot generating peer

In a nutshell, the private data in exported files fit naturally for sharing only within an org. In addition, the exported private data provides data loading efficiency. However, in all the scenarios, we still need to execute the code for detection of potential missing data (for both, the eligible and the ineligible collections) and populate with applicable missing data entries. In order to keep it simple and a single and uniform way to generate and share snapshots across scenarios, we chose to use the existing reconciler.

Having said that, still, exporting the private data to files may be important for backup and restore usecase but that is outside the scope of this RFC. We may provide explicit API/options to export private data corresponding to a snapshot later when we deal with this usecase - see more details in the section *Future anticipated related RFCs*

## Exporting the world-state
As mentioned in the section *Reference-level explanation*, the proposed approach for exporting the world-state (i.e., the public state and the private data hashes) directly from statedb causes a pause in the block commits to the channel. We considered alternative approaches to solve this problem. Below, we mention briefly the two approaches that we considered

### Maintain multiple versions of data items in the statedb
  In this approach, we maintain multiple versions of data items in the statedb. This includes the latest versions and the versions that belong to snapshots. Though, this approach works when leveldb is being used for the statedb, for the couchdb, this is not easy to achieve. Because, we do not have control over couchdb query engine. In order to achieve a common solution that works for both couchdb and leveldb, this approach requires us to maintain a separate leveldb instance that maintains multiple versions for the snapshot generation. The additional benefit of this approach would have been that exporting the snapshot data would be much efficient when couchdb is used for the statedb.

### Maintain Undo-logs during snapshot generation
In this approach, we maintain a separate leveldb instance for maintaining the data, akin to the undo-logs, for the duration of generation of the snapshot only.

Both of the above approaches require maintaining an additional data store. This additional data store adds complexity in the peer code that may raise the challenge for guaranteeing that the exported snapshot contents are same as in the statedb at the specified snapshot height. So, in order to keep the code simple, we chose for the approach presented in this RFC.

## Incremental snapshots
The approach presented in the RFC for generating the snapshot and computing the corresponding hash of the snapshot is very straight forward. Because, we always generate a complete snapshot. We considered some complex approaches of maintaining the state in a data structure that would allow for exporting the incremental snapshots and computing the hashes incrementally, such as bucket-hash Merkle tree from Fabric 0.6. However, since the frequency of snapshot generation is expected to be low, despite the complexity, these approaches may not benefit the general workload. In addition, it would not be possible for an org admin to verify the hash outside peer code.

# Prior art
We divide this section in three broad categories
## Checkpointing in public blockchains
Checkpointing has become significantly important in public blockchains, as typically they allow forks. In the absence of a checkpointing, in these networks, it is not easy to provide some basic guarantees on transactions' finality, and by extension, on double spending. A rudimentary approach for a checkpointing was introduced in the bitcoin network by hard-coding with every release, the choice of one of the blocks to treat as a checkpointed block. See [initial release message](https://bitcointalk.org/index.php?topic=437.msg3807) for more details. This feature was removed later, perhaps because, as such this did not add value to the basic security model and mandated the upgrade to new releases. Since then, there has been studies that analyze the checkpointing properties more formally and propose new methods for a federated agreement on a common checkpoint. These studies include [Securing Proof-of-Work Ledgers via Checkpointing](https://eprint.iacr.org/2020/173.pdf), [Transaction Finality through Ledger Checkpoints](https://ieeexplore.ieee.org/document/8975825), and [Casper the Friendly Finality Gadget](https://arxiv.org/pdf/1710.09437.pdf).

These studies at a broad level does not make much sense in the context of Hyperledger Fabric, as the Fabric does not support forks by design. Still, one subtle similarity is that similar to the public networks that allow state management via smart contract, such as Ethereum, Fabric also needs to compute the hash of the state at the checkpointed state. Ethereum stores hash of the state in each block and hence a checkpointing block is no special from other blocks in this regard. However, Fabric, from version 1.0 onward does not compute the state hash at each block commit. Hence, we would need to compute state hash at the checkpointing block. This aspect has already been factored in the design presented in this RFC.

## Snapshots in databases
Snapshots is a well studied domain in the context of regular databases. The snapshots in regular databases are used for both the transaction isolation and for taking backups. The databases maintain multiple versions for maintaining snapshots that allows concurrent operations. We have already discussed the reasons for not adopting this approach in the section *Rationale and alternatives*.

### Hash computation methods
There are different data structure that are suitable for different workload when the hash computation requirements are very frequent. This includes the structures used in Ethereum and Fabric 0.6. [Analysis of Indexing Structures for Immutable Data](https://arxiv.org/pdf/2003.02090.pdf) studies the performance characteristic of three persisted data structures in the context of frequent hash computation, at the granularity of a block.
We considered some of these approaches and have discussed the reasons for not choosing, in the section *Rationale and alternatives*

In addition to the above three categories, a small section in the paper [Scaling Hyperledger Fabric Using Pipelined Execution and Sparse Peers](https://arxiv.org/pdf/2003.05113.pdf) is relevant. Though, the focus of this paper is somewhat different, section 4.1 mentions about bootstrapping a peer by making a copy of statedb as is, as compared to copying the entire blocks.

# Testing

We divide the tests in three categories - integration tests, system tests, and performance measurements.
Under each category below, we mention the scenario that we intend to cover. Some of the failure scenarios mentioned in the integration tests may not be easy to test with guarantee from outside, still we mention them here.

## Integration Tests

1. Snapshot generation
   - Create a network and submit transactions that include private data
   - Invoke `SubmitSnapshotRequest` API on multiple peers
   - Verify that the snapshot generated across peers is exactly same
   - Verify that the snapshot is generated atomically

2. Join a new Org using a snapshot
    - Create a network with two orgs, Org1 and Org2
    - Commit some transactions, including private data for two collections - collection-1 and collection-2
    - Update channel config to include Org3
    - Upgrade chaincode definition to include Org3 to collection-1
    - Generate a snapshot for the channel
    - Create a new peer for Org3 and Invoke `JoinChannel` API with the previously generated snapshot
    - Verify that the peer of Org3 behaves for the transaction simulation and commit as expected
    - Verify that the private data for collection-1 is available on the bootstrapped peer
    - Verify that the private data for collection-2 is not available on the bootstrapped peer
    - Verify that query on historical transactions and blocks return appropriate errors
    - Verify that the peer returns an appropriate error when supplied with snapshot files with unexpected version info in the header byte
    - Verify that the peer returns an appropriate error when supplied with corrupted files and leaves the peer in a cleaned up state
    - Verify that at the peer returns an appropriate error when the hashes in the metadata files does not match with hash of the data files
    - Upgrade chaincode definition to include Org3 to collection-2
    - Verify that the peer for Org3 reconciles the private data for collection-2

3. Join an existing peer to the channel
    - Create a network with two orgs, Org1 and Org2 and two channels, channel-1, and channel-2
    - Org1.Peer1 joins only in channel-1, Org1.Peer2 joins both in channel-1 and channel-2
    - Commit some transactions to channel-2, including private data
    - Generate a snapshot for the channel-2 on Org1.Peer2
    - Join Org1.Peer1 to channel-2 using the previously generated snapshot
    - Verify the behavior as mentioned in the test 2 above

4. Failure during bootstrapping the peer, before peer loads the full data
    - Bootstrap a peer using a snapshot
    - Kill the peer during the bootstrap, before the peer loads the full data
    - Restart the peer
    - Verify that the peer can again join the channel with the same snapshot
    - Verify that the peer can again join the channel with a different snapshot
    - Verify that the peer can again join the channel using the genesis block

5. Failure during bootstrapping the peer, after peer loads the full data
    - Bootstrap a peer using a snapshot
    - Kill the peer during the bootstrap, after the peer loads the full data
    - Restart the peer
    - Verify that the peer resumes and finishes the channel joining process automatically

6. A peer bootstrapped via a snapshot can serve the private data when another peer is bootstrapped from the a snapshot
    - Test when the new peer is bootstrapped from the same snapshot
    - Test when the new peer is bootstrapped from a more recent snapshot

7. Behavior of remaining basic APIs for snapshot management 
   - Verify the behavior of APIs `PendingSnapshots, CancelSnapshotRequest, DeleteSnapshot, ListSnapshots, and SignSnapshotMetadata`
   - After deleting a snapshot, the SignSnapshotMetadata API should still work for the snapshot
   
8. Extend Ledger reset and rollback related tests to include a channel that is bootstrapped from a snapshot

## System Tests

Verify that Snapshot generation and Bootstrapping a peer functionally works as expected when there is a significant large data committed on the channel. Particularly, verify that the peer or couch database should not be crashing because of excessive resource usages over a large number of trials.

## Performance measurement

When there is significant large data committed on the channel, measure the time taken in generating the snapshot and bootstrapping a peer. Also, measure the performance degradation on the other channel during this period

# Dependencies

The existing Jira for the feature discussed in this RFC:
https://jira.hyperledger.org/browse/FAB-106

# Unresolved questions

## Resolve during implementation via appropriate Jira Items

1. Behavior of ledger rollback and reset in the presence of a channel that is bootstrapped from a snapshot. The tentative plans include the following
     - Instead of dropping the entire statedb, drop the channel specific data. Because, some channels may have been bootstrapped from a snapshot and we do not intend to preserve a copy of the snapshot on the peer only for this purpose.
     - If channel bootstrapped from a genesis block, the channel can be reset rolled back, similar to current behavior but the reset would become channel scoped.
     - Regardless of how the channel bootstrapped, from genesis block or from a snapshot, always allow bootstrap from a new snapshot.

2. Decide whether we should mandate that the peer maintains the BlockCommitHash for the channel. This require resetting the ledger on the snapshot creating peer

3. Decide how to expose the remote APIs - Perhaps via command line

4. Decide whether during bootsrap, the peer would match the hash of the snapshot before starting with populating the data or does this alongside

5. Fix other historical queries that we may have missed would be discussed to return appropriate errors



## Future anticipated related RFCs

### Ledger Archiving or Pruning
Ledger checkpointing would enable pruning of blocks forever or archiving them on a cheaper storage medium for record only. This may be useful if the participating orgs agree on some snapshot as a starting point of the channel. The relation to this RFC is that we would expect the pruning operation to leave a channel on the peer in the state that would be same as if the channel had been bootstrapped from a snapshot of same height.

### Backup and Restore
The generated snapshots can be backed up for a disaster recovery scenarios. However, this would require a peer to include the private data as well in the snapshot. As the primary usecase here is to consume the snapshot within an org, the admin would not worry about exporting the private data along with the snapshot. However, to restore a peer from such a snapshot, the bootstrapping process needs to be extended to utilize the private data from the backed-up copy. 

Also, since the backup is expected to a more frequent operation, an incremental back up and corresponding restore may be desired. However, that would diverge from the proposed checkpoint design significantly. See, related discussion in the sections *Rationale and alternatives* and *Prior art*.