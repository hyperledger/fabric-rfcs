---
layout: default
title: Fabric-X - A Microservice-based Hyperledger Fabric for High-Throughput Systems
nav_order: 3
---

- Feature Name: Fabric-X -- Next generation Fabric
- Start Date: 2025-05-14
- RFC PR: (leave this empty)
- Fabric Component: <u>peer (decomposed)</u>, <u>orderer (decomposed)</u>
- Fabric Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC proposes "Fabric-X," a fundamental re-architecture of Hyperledger Fabric designed to achieve significant performance gains, targeting over 100,000 transactions per second (TPS). Fabric-X adheres to Hyperledger Fabric's established **execute-order-validate** transaction flow paradigm; the primary innovations lie in the architectural changes to implement this flow via a highly distributed and scalable microservice-based system. This is in response to the limitations of the current Hyperledger Fabric (HLF) which, with a throughput of around 2,000 TPS, falls short for demanding use cases in financial services such as Central Bank Digital Currency (CBDC) systems, managing digital assets, etc. Fabric-X introduces a microservice-based peer and orderer architecture, a sharded distributed database, threshold-based signatures for reduced transaction overhead (alongside traditional endorsement policies with pre-stored identities), an enhanced programming model, parallel transaction validation with a dependency graph, and a highly scalable BFT ordering service named Arma. To avoid breaking backward compatibility with existing Fabric deployments and to accommodate substantial changes like a flattened transaction structure, Fabric-X will be developed in separate repositories (e.g., `fabric-x-endorser`, `fabric-x-orderer`, `fabric-x-committer`, `fabric-x-common`). Fabric-X will initially focus on core performance and will not support all features of the original Fabric, such as Private Data Collections (PDC), Channels.

# Motivation
[motivation]: #motivation

The primary motivation for Fabric-X is the critical need for significantly higher transaction throughput and scalability in Hyperledger Fabric to support emerging large-scale applications in digital assets. The current monolithic architecture of HLF faces several limitations:
1. **CPU Resource Contention:** Peers act as both endorsers and validators/committers on a single node, leading to CPU contention.
2. **Exclusive Ledger Access:** Endorsers and committers compete for exclusive access to the ledger state due to limited concurrency control in the state database.
3. **Disk Write Bandwidth Constraint:** Committer performance is bottlenecked by single-node disk write bandwidth and storage capacity.
4. **Large Transaction Size:** Transactions with multiple signatures and X.509 certificates create network and storage overhead, impacting ordering service performance and increasing storage consumption.
5. **Endorsement Policy Verification Overhead:** Verifying multiple signatures against complex policies is computationally intensive.
6. **Non-pipelined and Sequential Execution:** Sequential execution of all the phases (verification, validation, and commit), limits resource utilization and increses latency.
7. **Low Consensus Throughput:** Traditional BFT protocols and leader-based consensus in HLF have inherent throughput limitations.
8. **Inflexible Programming Model:** The inflexible programming model makes it hard to allow the chaincode to query the external world, consult other systems/blockchains, or different endorsers to run different logic for the same purpose, typically required in regulatory-relevant use-cases with multiple types of regulators involved.

The optimizations proposed in the following key research papers have not achieved the desired >20,000 TPS:
1. [A Transactional Perspective on Execute-order-validate Blockchains](https://dl.acm.org/doi/10.1145/3318464.3389693)
2. [Blurring the Lines between Blockchains and Database Systems: the Case of Hyperledger Fabric](https://dl.acm.org/doi/10.1145/3299869.3319883)
3. [FastFabric: Scaling Hyperledger Fabric to 20,000 Transactions per Second](https://arxiv.org/abs/1901.00910)
4. [Scaling Blockchains Using Pipelined Execution and Sparse Peers](https://dl.acm.org/doi/10.1145/3472883.3486975)

This necessitates a complete architectural redesign. Fabric-X aims to systematically address the performance issue.

A core consideration is **backward compatibility**. The proposed changes, such as splitting the `peer` binary into microservices (endorser, sidecar, coordinator, signature-verifier, vcservice, query-service) and the `orderer` binary into (router, batcher, consenter, assembler), along with a shift from a nested to a flat transaction structure, are fundamental. Implementing these in the existing Fabric repository would introduce significant breaking changes, disrupting all current deployments and operational scripts. Therefore, Fabric-X will be developed in new, dedicated repositories to allow innovation without destabilizing the existing Fabric ecosystem.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Fabric-X represents a paradigm shift in how Fabric's **execute-order-validate** protocol is implemented, moving from a monolithic structure to a collection of specialized, independently scalable microservices. **We have an implementation of Fabric-X that provides 100,000 TPS**. While the core transaction flow remains familiar (transactions are first executed by endorsers, then ordered, then validated and committed), the underlying architecture that facilitates this flow is significantly different, enabling higher performance. For a Fabric developer, this means understanding a more distributed system.

**Key Concepts:**

* **Decomposed Peer:** Instead of a single `peer` process, you'll interact with distinct services that collectively handle the `execute` and `validate` phase:
    * **Endorser Service:** Solely focuses on the `execute` phase of transaction proposals.
    * **Sidecar Service:** Acts as a dedicated communication bridge. It sits between the ordering service (Arma) and the Coordinator Service. Its responsibilities include fetching blocks from Arma, forwarding them to the Coordinator, managing the local block store, receiving consolidated transaction statuses from the Coordinator, and notifying registered clients about transaction events.
    * **Coordinator Service:** Orchestrates the validation and commit phases. It receives blocks from the Sidecar, builds the dependency graph, and dispatches transactions to Signature Verifiers and then to Validator-Committer services.
    * **Signature Verifier Service:** Dedicated to verifying endorsement signatures against policies.
    * **Validator-Committer (VC) Service:** Responsible for ledger state validation (read-set validation) and committing valid transactions to the distributed database.
    * **Query Service:** Handles read-only ledger queries efficiently.
* **Decomposed Orderer (Arma):** The "order" phase is managed by Arma, a microservice-based ordering service:
    * **Router:** Validates incoming transactions and routes them to appropriate shards.
    * **Batcher:** Bundles transactions into batches within shards.
    * **Consenter:** Participates in the BFT consensus protocol on batch digests.
    * **Assembler:** Reconstructs full blocks from ordered batch digests and batches.
* **Sharded Distributed Database:** The ledger state is managed by a sharded, strongly consistent distributed database, allowing for greater concurrency and overcoming single-node storage/IO limitations. Developers need to be aware that data is distributed, though the interaction model via chaincode during endorsement should remain largely consistent for basic operations.
* **Programming Model:** Fabric-X enhances the programming model by allowing application development beyond traditional chaincode-centric architectures while still benefiting from the security through endorsement policies.
* **Threshold Signatures & Traditional Policies with Pre-stored Identities:** Fabric-X will support threshold signatures for compact, efficient validation. Additionally, traditional endorsement policies will be supported, with an optimization to pre-store identities (e.g., X.509 certificate details) in a configuration block, allowing transactions to reference these identities rather than including full certificates with each signature, thus reducing transaction size.
* **Dependency Analysis Enables Parallel Verification and Validation:** The Coordinator constructs a dependency graph for transactions. This allows transactions without conflicts to be verified and validated in parallel by multiple signature-verifier and VC instances, significantly speeding up the commit phase.
* **Flattened Transaction Structure:** To reduce processing overhead associated with serialization/deserialization of deeply nested structures, Fabric-X will adopt a flatter transaction model.
* **Sequantial Versioning:** Fabric-X adopt sequantial versioning of key/values. This allows endorsers to predict the next version without querying the database.

**Impact on Fabric Users:**

* **Deployment:** Deployment scripts and tools will need to be adapted for a microservice environment (e.g., using Kubernetes or similar orchestration platforms).
* **Configuration:** Configuration will be more granular, specific to each microservice.
* **Debugging:** Troubleshooting will involve analyzing logs and interactions across multiple services.
* **Performance:** Expect significantly higher throughput and lower latency for transaction processing due to the architectural overhaul.
* **Feature Set:** Be aware that Fabric-X will not initially support all HLF v2.x/v3.x features (e.g., Private Data Collections, Channels). The focus is on core transactional performance of the execute-order-validate flow.
* **Security:** While individual components are more isolated, the overall security of the distributed system, including inter-service communication, must be carefully managed. Threshold signatures offer enhanced privacy by obscuring individual endorser identities.

**Migration:** There is no direct migration path from an existing Fabric network to Fabric-X due to the architectural differences. Fabric-X is a new platform.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This section details the technical design of Fabric-X. Fabric-X implements the execute-order-validate flow using the following microservice architecture.

## 1. Microservice Architecture and Distributed Database (Peer-Side)

The monolithic Fabric Peer is decomposed into the independently scalable microservices as shown in the below figure.
![Proposed Fabric Peer Architecture](../images/fabx/peer-arch.png)
* **Endorser Service:** Receives transaction proposals from clients. It executes the specified application endorsement logic against a snapshot read from the sharded distributed database through a query service and generates a read-write set along with its signature. In contrast to HLF, the endorser service may be part of the application stack. Multiple instances can be deployed to scale the endorsement capacity.
* **Query Service:** Dedicated to handling read-only chaincode queries submitted by clients. It leverages snapshot reads from the distributed database for efficient data retrieval without impacting transactional workload.
* **Sidecar Service:** This service acts as an intermediary and manager for block handling and eventing. Its key functions are:
    * **Block Ingestion:** Connects to the ordering service (Arma) to pull newly ordered blocks.
    * **Block Forwarding:** Forwards received blocks to the Coordinator Service for processing.
    * **Status Aggregation & Block Storage:** Receives final transaction statuses for blocks from the Coordinator. Once all transactions in a block are processed (committed or invalidated), the Sidecar writes the complete block to the persistent block store (local or distributed).
    * **Event Notification:** Manages and sends transaction commitment events to clients registered for notifications.
* **Coordinator Service:** The central orchestration point for the verification, validation, and commit phases on the peer-side. It:
    * Receives blocks from the Sidecar Service.
    * Constructs the transaction dependency graph for transactions within and across blocks (see Section 3).
    * Dispatches transactions (dependency-free ones first) to available Signature Verifier Services for endorsement policy verification.
    * Receives policy-verified transactions from Signature Verifiers and dispatches them to available Validator-Committer (VC) Services for read-set validation and ledger commit.
    * Collects transaction statuses (valid/invalid) from VC Services and forwards statuses to the Sidecar Service.
* **Signature Verifier Service:** Responsible for verifying that the endorsements on a transaction satisfy the specified endorsement policy (either traditional or threshold-based). This service can be scaled horizontally to handle high verification loads.
* **Validator-Committer (VC) Service:**
    * Performs read-set validation for each transaction against the current state in the sharded distributed database, using Multi-Version Concurrency Control (MVCC) to ensure serializability.
    * For valid transactions, applies the write-set to the distributed ledger.
    * Records their status (valid/invalid) appropriately.
    * The sharded distributed database supports strong consistency and concurrency control mechanisms (e.g., Snapshot Isolation) to eliminate exclusive access contention and distribute disk write load effectively.

## 2. Flexible Programming Model with Decoupled Endorser

The transaction execution phase in Fabric-X goes beyond the classic chaincode-centric programming model. By extracting the endorser component from the peer, we allow a flexible deployment of the endorser service. In HLF, application developer write their business logic as chaincode implementing the shim interface and install the chaincode at the endorsing peers. In Fabric-X, we allow application developers to integrate the endorsing logic directly in their applications by using an endorser SDK. Like in HLF, transactions are associated to a namespace which is protected by an endorsement policy which specifies the set of endorsing parties that must endorse the transaction contents to be considered valid. The endorser SDK allows to build the transaction read-write set and the endorsement signature.
This enhanced programming model allows application developers to implement interactive protocols tailored to their business logic, execution business related actions and collaboratively produce transactions for Fabric-X.

## 3. Threshold-Based Signatures and Optimized Traditional Policies

To address limitations 4 (Large Transaction Size) and 5 (Endorsement Policy Verification Overhead):
* Fabric-X allows endorsement policies to be defined by a single public key, thereby requiring only one signature. For multi-org endorsed transactions, Fabric-X will introduce support for threshold signature schemes, accompanied by a key distribution framework. This approach enables a transaction to carry a single, compact threshold signature instead of multiple individual signatures and certificates. Consequently, this significantly reduces both transaction size and the computational cost of verification. The verification key for the threshold signature can be directly linked to the endorsement policy.
* To retain expressiveness, Fabric-X will add support for traditional endorsement policies (e.g., AND/OR/OutOf). To mitigate their impact on transaction size, identities (such as X.509 certificate information) can be pre-registered on the blockchain, for example, via configuration blocks. Transactions can then reference these known, pre-registered identities along with the signature, eliminating the need to embed full certificates in every transaction.

## 4. Dependency Graph Facilitates Parallel Verification, Parallel Validation, and Pipelined Execution

To address limitations 6 (Non-pipelined and Sequential Execution):
* The Coordinator builds a **transaction dependency graph**. Nodes are transactions; edges represent dependencies. Let’s consider two transactions, $T_i$ and $T_j$ , where $T_i$ appears earlier in the block ordering than $T_j$ (either in the same block
or a preceding block). An edge from $T_j$ to $T_i$ signifies that $T_i$ must
be verified, validated, and committed/aborted before $T_j$ can be considered
for processing. We define the following three dependencies which
sufficient to represent any potential conflict that could arise from
concurrent access to shared state:
    * **Read-Write Dependency ($T_i \xleftarrow{rw(k)} T_j$):** $T_i$ writes key $k$, $T_j$ reads $k$'s previous version. If $T_i$ is valid, $T_j$ (which read stale data) must be marked invalid.
    * **Write-Read Dependency ($T_i \xleftarrow{wr(k)} T_j$):** $T_j$ writes $k$, $T_i$ reads $k$'s previous version. $T_i$ can be validated, but $T_j$ must not be committed before $T_i$ to ensure $T_i$ doesn't become invalid due to $T_j$'s commit. Enforces commit order.
    * **Write-Write Dependency ($T_i \xleftarrow{ww(k)} T_j$):** Both $T_i$ and $T_j$ write $k$. $T_j$ must not be committed before $T_i$ to prevent $T_i$'s write from being lost. Enforces commit order.
* Edges always point from a later transaction to an earlier one, making the graph a Directed Acyclic Graph (DAG).
* Local dependency graphs are built per block (potentially in parallel for graph construction itself if block data is available) and then merged into a global graph respecting block order.
* Transactions with an out-degree of zero are dependency-free and can be dispatched for signature verification and then MVCC validation concurrently by multiple Signature Verifier and VC services respectively.
* Successful commits resolve dependencies, enabling further parallel processing. This pipelined approach allows verification, validation, and commit phases to overlap.

## 5. Arma: A Scalable Ordering Service

Arma is a high-throughput BFT ordering service designed to replace the current HLF orderer, addressing its throughput limitations. Key design principles include:
1.  **Separation:** Ordering transaction digests, not full transactions.
2.  **Pipelining:** Decomposing ordering into parallel stages (Routing, Batching, Consensus, Assembly).
3.  **Sharding:** Dividing processing, especially batching, into parallel shards.

Each Arma "party" (participating organization) runs microservices for these stages
as shown in the below figure.
![Proposed Fabric Orderer Architecture](../images/fabx/orderer-arch.png)

The figure above presents the Arma party composition. An Arma party is equivalent to an HLF v3 OS Node. An Arma party is composed of 4 sub-components: a router (R), one or more batchers (B), a consenter (C), and an assembler (A). Endorsing clients (E) submit transactions to the router. The router spreads transactions across the shards, sending only to the batcher (B) of its own party (p) in a given shard. Batchers aggregate transactions in to batches (B1,B2), and compute a signed digest (batch attestation fragment) on the batches (BAF1,BAF2, resp.). The consensus cluster collects BAFs and emits a total order of batch attestations (BAs). The assembler consumes both totally ordered BAs from consensus, and batches from the shards. It then assembles HLF blocks according to the order induces by consensus and the content received from the shards. The HLF blocks are then ready for the scalable committer (SC) to pull.

The transaction flow is as follows:

* **Router Service:** Clients submit transactions, typically to multiple Arma parties for robustness. The Router service at each party:
    1.  Validates incoming transactions (format, ACLs, client signature).
    2.  Assigns each valid transaction to a specific shard using a deterministic hash function ($H_r(TX) \rightarrow {shard\_ID}$) for load balancing.
    3.  Forwards the transaction to its party's Batcher service instance for that assigned shard. Routers are horizontally scalable.

* **Batcher Service:** Within each shard, each party runs a Batcher instance; one is elected primary, others are secondaries for a given term.
    1.  The **primary Batcher** bundles transactions routed to it into batches, identified by $\langle \text{shard, primary, seqN, digest} \rangle$. It persists these batches locally and disseminates them to secondary Batchers in the same shard.
    2.  **Secondary Batchers** receive batches, verify transactions (to prevent junk injection by a faulty primary), and persist them. If a primary censors transactions, secondaries can forward them to the primary and initiate complaints if censorship persists.
    3.  All Batchers (primary and secondaries), after validating and persisting a batch, create and send a signed **Batch Attestation Fragment (BAF)** for that batch to the Consensus Nodes.

* **Consensus Node Service:** Consensus Nodes across all parties use a BFT protocol (e.g., SmartBFT library) for two main purposes:
    1.  **Order BAFs:** They agree on a total order of BAFs received from Batchers. Once $F+1$ distinct BAFs for the same batch are ordered, the consensus protocol outputs a signed **block header** containing the batch digest, sequence number, and a hash pointer to the previous header.
    2.  **Manage Primary Rotation:** They order complaint votes from Batchers against their primaries. If $F+1$ complaints for a primary in a shard are ordered, a primary rotation is triggered, and a new primary is selected deterministically (e.g., round-robin).

* **Assembler Service:** The Assembler service at each party:
    1.  Retrieves the ordered, signed block headers from the Consensus Nodes.
    2.  Fetches the corresponding full data batches from the Batcher services of the relevant shard (guaranteed to be available from at least one correct Batcher due to the $F+1$ BAFs).
    3.  Combines the header and batch to form a complete Arma block, which is then persisted and made available to the peer-side components (e.g., Sidecar service).

The below figure depicts the Arma party architecture and the transaction flow.
![Proposed Fabric Orderer Transaction Flow](../images/fabx/orderer-flow.png)

In the figure above a correct endorsing client (E) tries to submit a TX to all parties. Routers validate and dispatch the TX to a shard according to hash function $H_r$. The primary batcher in a shard (e.g. $B2_{s3}$ in shard 3) bundles TXs in a batch, persists it, and broadcasts it to the secondary batchers. Batchers that persist a batch send a BAF to the consensus cluster. Upon receiving enough BAFs, the consensus cluster emits a total order of BAs. Assembler nodes receive a stream of BAs from consensus and collate them with matching batches they pull from the shards. Assemblers then append HLF blocks to their ledger and make it available to the scalable committer (SC). Finally, a batcher that suspects misbehavior of the primary may complain to the consensus cluster (e.g. $B4_{s3}$ in shard 3), which given enough distinct complaints will exert control and change the primary of a shard.

**BFT Properties of Arma:**
Arma's BFT properties (safety and liveness) are derived from the underlying BFT consensus on headers and the mechanisms ensuring batch availability and primary rotation. $F+1$ BAFs ensure batch persistence before ordering, and the complaint mechanism handles faulty or censoring primaries.

# Drawbacks
[drawbacks]: #drawbacks

* **Increased Operational Complexity:** Managing a distributed system of microservices (Fabric-X peer and orderer components, distributed database) is inherently more complex than managing monolithic Fabric nodes. This requires more sophisticated deployment, monitoring, and orchestration tools.
* **Backward Incompatibility:** As stated, Fabric-X is not backward compatible with existing Hyperledger Fabric versions. This means current Fabric users cannot simply upgrade; they would need to adopt a new platform. This necessitates the separate repository approach. As a result, developers and administrators familiar with the current Fabric will face a learning curve adapting to the new microservice architecture, new components like Arma, and the distributed database.
* **Reduced Initial Feature Set:** To focus on performance and core functionality, Fabric-X will initially omit certain features present in HLF, such as Private Data Collections, Channels, and Chaincode API. Re-introducing such features later will require careful design to align with the new architecture.

# Rationale and alternatives
[alternatives]: #alternatives

* **Why is this design the best in the space of possible designs?**
    * **Microservices for Scalability & Isolation:** Decomposing the peer and orderer into microservices directly addresses CPU contention, allows independent scaling of specific functions (e.g., endorsement, signature validation, block handling via Sidecar), and improves fault isolation. This is a proven pattern for building scalable and resilient systems.
    * **Distributed Database for State Management:** A sharded distributed database tackles disk I/O bottlenecks, storage limitations, and improves concurrency for ledger access, which are critical pain points in the current HLF for high-throughput scenarios.
    * **Targeted Optimizations:** Threshold signatures, optimized traditional policies, parallel validation with dependency graphs, and Arma's design (digest ordering, pipelining, sharding) are specific solutions to identified performance limiters (transaction size, sequential validation, consensus overhead).
    * **Preservation of EOV Flow:** By maintaining the execute-order-validate paradigm, Fabric-X remains conceptually aligned with HLF's trusted model while radically improving its performance characteristics. The changes are architectural rather than a shift in the fundamental transaction lifecycle.
    * **Addressing All Key Limitations:** This holistic design attempts to address all eight major limitations identified, rather than providing incremental improvements.

* **What other designs have been considered and what is the rationale for not choosing them?**
    * **Incremental Improvements to Existing Fabric:** While some performance gains could be made by optimizing existing HLF components, these would not reach the target of >100,000 TPS. The inherent limitations of the monolithic architecture and current consensus would cap potential improvements well below the required threshold.

* **What is the impact of not doing this?**
    * Hyperledger Fabric would remain unsuitable for extremely high-throughput applications like many national-level CBDC implementations or other large-scale transactional systems.
    * The Fabric ecosystem might lose relevance for use cases demanding performance beyond its current capabilities.
    * The opportunity to leverage Fabric's robust permissioning model and enterprise features in these high-demand scenarios would be missed.

# Prior art
[prior-art]: #prior-art

* **Microservice Architectures:** The decomposition of monolithic applications into microservices is a widely adopted architectural pattern (e.g., Netflix, Amazon). Fabric-X applies this to the blockchain peer and orderer.
* **Narwal and Tusk:** Arma's design for ordering is inspired by [Narwhal and Tusk: A DAG-based Mempool and Efficient BFT Consensus](https://arxiv.org/abs/2105.11827), separating dissemination from consensus and using BFT.
* **Sharded Databases:** (e.g., YugabyteDB, CockroachDB, Google Spanner, Vitess) for scaling data storage and throughput.
* **Threshold Signatures:** Active research area applied in blockchain for efficiency and privacy.

Fabric-X combines these established patterns and innovations.

# Testing
[testing]: #testing

Validating this proposal will require a comprehensive testing strategy beyond standard unit tests:

* **Unit Tests:** Each microservice (Endorser, Sidecar, Coordinator, Signature Verifier, VC, Query Service, Router, Batcher, Consenter, Assembler) and major internal modules.
* **Integration Tests:**
    * **Intra-Peer:** Interactions between peer microservices (e.g., Sidecar to Coordinator, Coordinator to Signature Verifier and VC).
    * **Intra-Arma:** Interactions between Arma microservices.
    * **Peer-Orderer:** End-to-end flow including Sidecar's role in block fetching.
    * **Distributed Database Integration:** All ledger read/write operations.
* **Performance and Scalability Testing:** Targeting >100,000 TPS, measuring latencies, testing horizontal scaling.
* **BFT Testing (Arma):** Resilience, safety, liveness, primary rotation.
* **End-to-End (E2E) Scenario Testing:** Simulating high-volume use cases, concurrent submissions, data consistency.
* **Security Testing:** Penetration testing, secure communication, threshold signature implementation.
* **Long-Running Stability Tests:** Identifying memory leaks, resource exhaustion.

**Integration Test Scenarios (Examples):**

1.  **Successful Transaction Flow:** Client -> Endorser -> Arma Orderer -> Sidecar -> Coordinator -> Signature Verifier -> VC -> Ledger. Verify state update and Sidecar event notification.
2.  **Transaction with Invalid Endorsement:** Rejection at Signature Verifier or VC.
3.  **Conflicting Transactions in Parallel Validation:** Dependency graph correctly identifies conflicts.
4.  **Arma Primary Batcher Failure:** New primary elected, shard continues.
5.  **Sidecar Fails to Pull Blocks:** Test recovery mechanisms and impact on Coordinator.

# Dependencies
[dependencies]: #dependencies

* **External Libraries/Frameworks:**
    * **SmartBFT Library:** For Arma's consensus.
    * **Distributed Database System:** Specific choice TBD (e.g., CockroachDB, TiKV).
    * **gRPC/Messaging System:** For inter-microservice communication.
    * **Cryptography Libraries:** For threshold signatures and standard operations.
* **New Hyperledger Fabric-X Repositories:** This proposal necessitates new repositories:
    * `fabric-x-common`: Shared code, data structures (flattened transaction format), tools.
    * `fabric-x-endorser`: Endorser service implementation.
    * `fabric-x-orderer`: Orderer service implementation.
    * `fabric-x-committer`: Committer service implementation.

# Unresolved questions
[unresolved]: #unresolved-questions

* **Scope of Initial Feature Set:** Exhaustive list of HLF features not initially ported.
* **Error Reporting and Distributed Tracing:** Mechanisms across microservices.
