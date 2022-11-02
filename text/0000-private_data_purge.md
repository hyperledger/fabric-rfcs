---
layout: default
title: Private Data Purge
nav_order: 3
---

- Feature Name: private_data_purge)
- Start Date: 2021-10-01)
- RFC PR: (leave this empty)
- Fabric Component: chaincode, ledger
- Fabric Issue: (leave this empty)

# Summary
[summary]: #summary

The private data collections feature in Fabric allows the dissemination of the sensitive data to a subset of organizations in a channel. A peer that is eligible to receive the private data of a collection within a namespace, maintains all the versions of the private data in its private data store forever. This RFC proposes a new operation *purge* on private data via chaincode shim that allows the deletion of one or more private data keys from the current state (statedb), and in addition, purging of all the historical versions of the supplied keys from the private data store permanently. In addition, it lays down a basic groundwork for enabling history-related queries on private data.

# Motivation
[motivation]: #motivation

The primary motivation of this feature is to enable users to implement right to forget in the context of private data in a Fabric application. This may help users to meet certain business compliance about data privacy. At present, there exists two ways to retrieve historical versions of private data. An end user can retrieve historical versions of private data from a peer using the block Deliver service (`DeliverWithPrivateData` API). Additionally, other peers, potentially from other organizations, can access to the historical versions via the private data reconciliation process. Further, in future, we may expose an API for users to access the history of a specific private key in a collection, similar to the existing API for accessing the history of a key in the chaincode namespace. By enabling the purging of desired private data, the proposed feature makes peer prevent sending private data out from these APIs and eventually deletes it permanently from its persistent stores. An added advantage of this feature is that it enables users to free up some storage space by purging the historical versions of private data.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation
To enable private data purge feature, we introduce a new API in chaincode shim. Below is the function signature of this new API.

**`PurgePrivateData(collection, key string) error`**

As far as the latest state of a channel, maintained in statedb, is concerned, this new operation, `PurgePrivateData`, is no different from existing operation, `DeletePrivateData`. Hence, an application or chaincode would not notice any difference in behavior. The additional semantics associated with `PurgePrivateData` primarily impact private data store and reconciliation of private data. Both of these are not exposed to end users as of now (beyond the APIs mentioned above).


For understanding the usage of this API, lets assume a scenario where a chaincode defines a private data collection and the collection contains personally identifiable information for customers such that the chaincode maintains PII info of a customer in the form of three keys, namely {CustomerID}_PII_Name, {CustomerID}_PII_DOB, and {CustomerID}_PII_Address. While name and date-of-birth may not change, a customer may have updated his address a few times and hence peers of eligible orgs would have persisted all the historical values for the address. Now, the customer may want the Fabric network to purge any of its PII data.

In order to achieve this, the application may invoke a chaincode transaction in which the chaincode in turn invokes the *PurgePrivateData* API three times - once for each of the above mentioned PII keys. Finally, the application would get enough endorsements to satisfy the endorsement policies for each of the above keys and submits the transaction to the orderers. Once the transaction is committed on a peer, the peer stops sending out any version of any of these private data keys and eventually deletes from the persistent stores.

Note that we will gate this new API behind a capability tag so that this new operation is interpreted on all the peers in a channel and assures the user that the data will be purged if the transaction is committed as a valid transaction. In the absence of the capability, the endorser will return error during the invocation of `PurgePrivateData`. As this is a newly added function, it is sufficient to gate it at the endorsing time that ensures that no transaction with purge operation gets committed before enabling the capability.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Before diving into the proposed design, we describe some background, challenges, and broad requirements for the desired behavior for implementing private data purge.

## Background, challenges, and Requirements
We divide this section into two subsections.

### Deletion of private data from statedb and private data store
In regard of actual deletion of private data keys specified in a purge transaction, following is the desired behavior of a peer.
- Deletion of the private data key from the state should be done atomically with the other state updates caused by the transaction. In other words, as far as state updates are concerned from the perspective of future transaction simulation, the *purge* operation is no different from *delete* and merely deleting from hashed state is sufficient at commit time.
- While the hashed key must be deleted from state atomically, the private key itself could be deleted in the background (not atomically with commit). To keep the implementation simple however, as a first implementation the private key will also be deleted atomically at commit time.
- The deletion of all the historical versions from the private data store is expected to be time consuming. Moreover, it does not have any bearing on the transaction execution. So, in the interest of not directly impacting the transaction throughput, it is desired to perform this operation in the background
- Despite performing the deletion from private data store in the background, the peer should behave for all external communication as if the data is actually purged, i.e., not send out the data in API responses after the purge transaction has been committed.
- In some scenarios, when an org is removed from a collection, the peer of the org stops receiving future private data operations for the collection. This very well includes the private write-set that includes the purge operation itself. However, such peers may already have the historical versions of private data and latest of those versions in its private state. So, it poses a requirement for performing the deletion of private key versions both from the private data store and from the private key, while working with the hashed write-set, which is delivered to all the peers, as part of transaction inside the block.

### Impact and desired changes on private data reconciliation
Most impacted function from this feature would be [private data reconciliation](https://hyperledger-fabric.readthedocs.io/en/latest/private-data-arch.html#private-data-reconciliation). As a recap, the private data reconciliation enables an eligible peer to retrieve latest and historical versions of private data from other eligible peers. The reconciliation is triggered by the requesting peer when the peer has some versions of private data missing in its private data store. This missing data situation is caused by one of the following situations.

1) Private data could not be delivered to the peer along with the block at the block commit time
2) A peer (org) is added to an existing collection, so peer catches up by pulling all historical versions of private data

In the current implementation of reconciliation, the basic unit of data transfer in the reconciliation is a tuple <blockNumber, transactionNumber, chaincodename, collection> i.e., the entire private write-set of a collection within a transaction is what the requesting peer asks for from another eligible peer. The requesting peer, before accepting the private write-set from the replying peer, verifies the contents by matching the hash of the entire private write-set with the expected hash for this collection that is included in the block. However, when we implement the private data purge feature, the replying peer may send a subset of the private write-set as some of the keys may have been purged. This change in behavior motivates the following changes in the reconciliation process.

- The replying peer filters the already purged keys before sending out the response to reconciliation request
  - For example, if a private key purge transaction is committed at block number 100, the replying peer will remove the private key from the response if the version of the key being sent was committed in a block prior to block number 100.

- The requesting peer filters the already purged keys
  - In some scenarios, the replying peer of private data historical version may be behind the requesting peer in the terms of block chain height. For example, assume that a private key purge transaction is committed at block number 100. The requesting peer is at height 101 and seek to retrieve private data corresponding to block 50. The replying peer is at height 60, i.e., it can deliver the requested private data but has not yet seen the purge transaction. In this case, the requesting peer needs to trim the received write-set by removing the purged keys that it is aware of before persisting the write-set.

- The requesting peer accepts and verifies subset of the collection
  - Contrary to the above situation, the requesting peer may be behind and may not be aware of the purge transaction yet. While, the replying peer sends the subset of the collection write-set. In this scenario, the requesting peer needs to verify the subset by comparing the hashes of the individual keys (as opposed to the hash of the entire collection write-set). Moreover, the requesting peer cannot remove the collection from its list of still missing data, until the requesting peer itself sees the purge transaction, as the replying peer may be playing malicious by deliberately sending a selective subset.


Keeping the above requirements and desired behavior in consideration, below is the the proposed design. First, we describe the new key types that we would maintain in the private data store and then, we discuss various operations in Fabric that would create, update, read, or delete these keys to implement this feature.

## Proposed Design
We divide the proposed design in four subsections. First, we describe the changes in the Write-set Proto message. Second, we introduce two new key types that we would maintain in the private data store. Third, we describe the proposed changes in the existing functions and finally, we describe the new functions that we would add.

### Changes in Write-set Proto
We capture the Hashed write-set for a particular key in the proto message [`KVWriteHash`](https://github.com/hyperledger/fabric-protos/blob/9c69228417592158899cb0dd4a77a3eedf25225f/ledger/rwset/kvrwset/kv_rwset.proto#L57). In the proposed design, We would extend this proto to capture the purge operation. We do this by adding an additional field `is_purge`.
- Note that unlike existing fields in the hashed-write-set, this field will not have any corresponding field in the raw private-write-set.
- Also, when we set `is_purge` to true, as a result of invocation of the newly introduced chaincode shim API, we also set the existing field `is_delete` set to true both in the hashed-write-set as well as in the private-write-set. In the terms of semantics, we interpret `is_delete` as the instruction for deleting the key from the latest state while `is_purge` indicates purging from the private data store.

### New Keys in Private data store
 In this design, we maintain two types of additional keys in the private data store. We refer to these new types of keys as *PurgeMarker* and *HashedIndex*. Below, is the format and intended use of these keys.
1) PurgeMarker
   A PurgeMarker key is intended to record a purge operation for a key. The design of this marker key would be `Ns_Coll_Hash(PrivateKey)`, while the marker value would contain `CommittingBlkNum_CommittingTxNum`. For illustration, assume that a purge transaction appears in block number 100, at position 20 that specifies for purging the keys `key1` and `key2` under collection `myColl` for chaincode `myCC`. When processing this transaction, we insert the two keys into private data store as `myCC_myColl_hash(key1)` with value `100_20` and `myCC_myColl_hash(Key2)` with value `100_20`. Note that the separation of different components of a key and value shown by an underscore is just for the illustration. For the reasons discussed later, we intend to keep these marker keys forever in the initial implementation. The assumption is that the purge operation is expected to be relatively less frequent, and hence, it should not be a concern from storage space point of view. We may revisit this assumption however after an initial implementation and assessment.

   - Although not required, we can also create a PurgeMarker record at the collection level to serve as an optimization in the implementation to efficiently identify whether any key has ever been purged for a given collection.

2) HashedIndex
   A HashedIndex key is intended to act as an index for the data keys in private data store, so that when a key is eventually purged we know from where to purge it in the private data store. The format of this key would be `Ns_Coll_Hash(PrivateKey)_blkNum_txNum`. For each version of a private key in the private data store, we would maintain such an index key. For illustration, assume that a key `key1` under collection `myColl` in chaincode `myCC` is inserted at block 100, transaction 10. Later this key is updated in block 200, transaction 20 and finally, the key is deleted in block 300, transaction 30. In this case, today, we maintain in the private data store all three private write-sets that contain, potentially along with other, the above three mentioned operations. In this example, for `key1`, we would maintain three HashedIndex keys as `myCC_myColl_hash(key1)_100_10`, `myCC_myColl_hash(key1)_200_20`, and `myCC_myColl_hash(key1)_300_30`. The value for the HashedIndex key will be the private key pre-image itself, to serve as a reference for which private keys can be purged from the peer for a given hashed key in a transaction write-set. For instance, in the above two examples, the values would be `key1` and `key2` respectively.

   - Note that the format of this key look same as the format of the PurgeMarker key - however, we always prepend a unique byte for the key type so, no two keys of different type ever clash.

   - Unlike the PurgeMarker key, we delete an HashedIndex key when we delete the corresponding data, either because of newly introduced explicit purge operation in this RFC or because of an implicit purge operation caused by [expiry of private data](https://hyperledger-fabric.readthedocs.io/en/release-2.2/private_data_tutorial.html#purge-private-data).

   - An important point to note here is that we would need to create these index keys retroactively - i.e., when a peer is upgraded to a version that would introduce private data purge feature, as a part of the upgrade, we would create these index keys.

### Changes in existing operations

#### Block commit
In the current code, when computing the state updates for a block, for each updated key, we take the latest value (i.e., the value set by the highest valid transaction in the block for the key). With the purge private data feature, while we still compute the state updates as is but, in addition, we perform following two data-set for committing to the private data store.
- Compute a set of PurgeMarker keys such that the set contains one entry for each hashed-key that is marked for purge by at least one transaction in the block. Further, if there exist more than one transactions that purge the same key, we ignore the lower transaction for the key. For example, assume block 100 contains transaction 10 that purges the key `key1` and `key2` and transaction 15 that purges the key `key1` and `key3` for collection `myColl` in chaincode `myCC`. In this case the set of PurgeMarker keys would look like `myCC_myColl_hash(Key1)` (with value `100_15`), `myCC_myColl_hash(Key2)` (with value `100_10`), and `myCC_myColl_hash(Key3)` (with value `100_15`).
- Compute HashedIndex keys for each private key-value present in the private write-set.


#### Private data commit via reconciliation
As mentioned previously, private data reconciliation is going to be most affected of the existing functions. Below is a list of changes that we would make in the reconciliation.

1) When we commit private data via reconciliation, the requesting peer may receive the keys that have already been purged. This would be the case when the replying peer has not yet seen the purge transaction. However, the requesting peer may have seen the purge transaction. In order to not save the purged keys, we would first compute the expected private write-set by removing the already purged keys on the requesting peer from the hashed write-set from block, by intersecting with PurgeMarker keys. This implicitly means that we need to maintain PurgeMarker keys forever (theoretically, at least, until no more missing data, including the missing data for ineligible collections, exists below the committing block of PurgeMarker key).

2) In the current code, during reconciliation, a requesting peer verifies the data by computing the hash of the private write-set received against the hash of the collection present in the corresponding transaction in the block. On a side note, if a peer has been bootstrapped via a snapshot, this is true only for the private data that has been added after the height of the bootstrapping snapshot. Now, in the presence of the private data purge feature at a key level, a peer may receive a partial private write-set, because one or more keys in the private write-set may have been purged by the replying peer. Hence, it is required for the requesting peer to perform the verification of hashes at the level of individual key-value pairs.

3) During reconciliation, a requesting peer may receive a subset of the expected write-set, as computed in (1) above, including an empty write-set, because the requesting peer may yet not have seen the purge transaction while the replying peer has seen it. To handle this, the requesting peer accepts the subset of the expected write-set and puts the missing data request into the deprioritized list and is attempted again at a low frequency interval. Eventually, the requesting peer catches up to block height and executes the purge transaction and gets aware of the purge transaction. Finally, when the expected keys, as computed in (1) above, becomes the empty-set, the peer updates the missing keys info as fully reconciled.

#### Queries on Private Data Store
The queries need to filter the private write-sets that it sends out as a response to a query. This is intended to be achieved with the help of PurgeMarker key, if the version of the key is lower than the version present in one of the corresponding PurgeMarkerKey keys. This is required because the actual data purge may take some time after having created the marker keys.


### New Operations

#### Purging from Private Data Store and private state
As mentioned in section [Background, challenges, and Requirements](#background-challenges-and-requirements), we intend to perform the private data store deletions of the historic keys in the background. Therefore, at the time of processing a private data purge transaction, we insert the marker key at the time of block commit, as described in the section [Block commit](#block-commit). In order to perform the actual purge of the data from private data store, we run a background goroutine that reads the PurgeMarker keys and scans all the relevant HashedIndex keys such that the index keys contain lower blkNum_TxNum than what is present in the PurgeMarker key. Finally, we would use the index key to locate the actual private write-set and we would trim the write-set by purging the intended key (recall that the HashedIndex value contains the actual private key as a reference for deletions). In the implementation, we intend to add an info field in the Marker key that would indicate whether the particular marker key has been processed. Also, if another purge operation is specified on the same key in the future, we delete the existing PurgeMarker for the key

#### Creating the HashedIndex keys retroactively
As mentioned in the section [New Keys in Private data store](#new-keys-in-private-data-store), when starting the peer version that contains purge support for the first time, we would need to retroactively create the HashedIndex keys for already committed private data in the private data store. We intend to do this automatically at the start of peer using the framework we introduced in data format upgrade in version 2.0.

[drawbacks]: #drawbacks
# Drawbacks
If historical versions of private data are purged, it will not be possible to rollback the peers to a previous height without losing the purged private keys forever. However, this is not a big drawback as reset/rollback is not a strategic direction for Fabric.

# Rationale and alternatives
[alternatives]: #alternatives
- Maintaining the PurgeMarker keys forever may monotonically increase the storage demand. An alternate design choice is to reformat the value for the missing keys in the private data store. In the current code, we maintain the missing data information just at the collection level, not at the level of individual key level - because, so far purging the individual keys were not possible. In the changed format, we can maintain the hashes of individual keys (or key-values both) as they appear in the block. We already do this in a specific scenario - when a peer is bootstrapped from a snapshot, the private data hashes cannot be fetched from the corresponding block and hence we load them in the private data store during bootstrapping a channel from a snapshot and maintain until the peer receives the private data via reconciliation.

   In this scheme, we would simply keep deleting the key hashes from the missing data entry and hence the peer knows exactly what keys are purged. Though, this scheme is expected to increase the performance for the reconciliation, but this would come at its own storage cost of maintaining the hashes for individual keys. However, this is expected to induce much bigger storage demand than maintaining the PurgeMarker keys because, in the case of Purge Marker we maintain at most one marker for a given key unlike, storing hashes repeatedly for each version of the key. As reconciliation is not a very frequent scenario, we prefer low storage cost over reconciliation efficiency and hence chose the PurgeMarker based approach as compared to the alternate design mentioned here.

- Maintaining HashedIndex keys adds additional storage demand. The alternate choice is not to maintain these indexes, however, in absence of these, we would be required to scan and unmarshal entire data in the private data store each time we process a PurgeMarker. In the proposed design we maintain HashedIndex such that for a PurgeMarker all the relevant HashedIndex are scanned sequentially followed by random access for relevant items only. Moreover, as the purge operation is supposed to free up space, the effect of additional index keys may not be high. Also, these HashedIndex keys maintain the raw private key as well (as the value of the index key) otherwise, in addition to scan and unmarshal, we would need to compute the hashes of all the keys as well.

- We always desire to avoid the need of introducing a new capability as it requires coordination between admins of different orgs. In the context of private data purge feature, as far as the latest state and transaction processing is concerned, we can introduce this feature without gating it behind a new capability tag -- Because, as mentioned above, we would always set `isDelete` to true when we set `is_purge` to true and `isDelete` would be interpreted the same way by the existing and the new version. Still, without introducing a new capability, in a channel, peers different versions would behave differently for purging of historical versions. Which raises some concerns as the purpose of introducing this feature via a transaction is to ensure that the requested private data is purged fully. Otherwise, this feature could have very well been exposed as an admin API on peer that a user can invoke on selective peers. Moreover, this in turns makes reconciliation process more challenging as a peer that missed processing a purge transaction, would not learn this fact and would keep seeking for the missed private data. As an alternate of this, we considered to provide an offline utility that would scan the block store for the purge-transactions and create the PurgeMarker keys in the private data store. The admin can use this utility on their peer either before or after upgrading their peers without coordinating with others. Though this avoids the need to set the capability but it comes with its own challenges which would be newer as compared to the known procedure of setting the capability.

# Testing
[testing]: #testing
In addition to regular unit tests and component level integration tests, following behavior should be verified by the end-to-end integration tests. For testing the purged data, we intend to use deliver service APIs to fetch a particular version of private data.

1) User is able to submit a purge transaction that involves more than one keys

2) The endorsement policy is evaluated correctly for a purge transaction under different endorsement policy settings (e.g., collection level/ key-hash based)

3) Data is purged on an eligible peer
   - Add a few keys into a collection
   - Issue a purge transaction for some of the keys
   - Verify that all the versions of the intended keys are purged while the remaining keys still exist
   - Repeat above to purge all keys to test the corner case

4) Data is purged on previously eligible but now ineligible peer
   - Add a few keys into a collection
   - Submit a collection config update to remove an org
   - Issue a purge transaction to delete few keys
   - The removed orgs peer should have purged the historical versions of intended key

5) A new peer able to reconcile from a purged peer
   - Stop one of the peers of an eligible org
   - Add a few keys into a collection
   - Issue a purge transaction for some of the keys
   - Start the stopped peer and the peer should reconcile the partial available data

6) The purge transaction takes effect only if the corresponding capability is set
   - Add a few keys into a collection
   - Issue a purge transaction without setting the new capability
   - Verify that the purge behaves exactly as `delete` - i.e., deletes from state but does not purge the historical versions
   - Issue a config transaction to set the new capability
   - Now the purge transact should delete the historical versions as well


# Dependencies
[dependencies]: #dependencies

In the future, we if decide to provide another API that allows access to history of private data keys, the HashedIndex keys introduced in this RFC can be leveraged as is.

# Unresolved questions
[unresolved]: #unresolved-questions
