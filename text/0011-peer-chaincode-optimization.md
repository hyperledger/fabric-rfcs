
---
layout: default
title: Optimization of Communication Protocol between Hyperledger Fabric Endorsing Nodes and Chaincodes
nav_order: 3
---

- Feature Name: Peer to Chaincode Communication Optimization
- Start Date: (fill me in with today's date, 2022-03-01)
- RFC PR: (leave this empty)
- Fabric Component: peer, chaincode
- Fabric Issue: (leave this empty)


## Abstract
This RFC proposes an enhancement to the communication protocol between Hyperledger Fabric endorsing nodes and chaincodes by introducing a batching mechanism. The current protocol requires each state modification command, such as `PutState` or `DelState`, to be sent individually, creating unnecessary overhead. This optimization minimizes the frequency and volume of communication between the chaincode and peer nodes, improving performance without changing the transaction semantics or requiring modification.

---

## Motivation
In the current Hyperledger Fabric architecture, every state-changing operation triggers a distinct message from the chaincode to the peer node. This design causes several inefficiencies:

1. **Communication Overhead:** The need to send individual messages for each state change operation increases the total number of messages transmitted, especially for workloads involving many updates. Each message exchange introduces latency and network overhead.

2. **Performance Degradation:** Applications that perform multiple state modifications in loops experience high latency due to the large number of message exchanges. As a result, transaction throughput decreases, and system resources are inefficiently utilized.

### Objectives of the Optimization
The proposed batching mechanism aims to address these challenges by:
- **Reducing Message Volume:** Grouping multiple state changes into a single `ChangeStateBatch` message reduces the number of messages exchanged between chaincode and peers.
- **Improving Resource Utilization:** By transmitting fewer messages, the network load is reduced, and processing overhead on both the peer and chaincode is minimized.
- **Ensuring Backward Compatibility:** Existing applications can continue to function as before without modification. If the batching feature is not supported by a component, the system will gracefully fall back to the original communication model.

---

## Problem Illustration with Sample Chaincode
The following chaincode example demonstrates the inefficiencies caused by the current communication protocol. Each iteration of the loop sends an individual `PutState` message to the peer node, resulting in unnecessary overhead:

```go
type OnlyPut struct{}

// Init initializes the chaincode.
func (t *OnlyPut) Init(_ shim.ChaincodeStubInterface) *pb.Response {
    return shim.Success(nil)
}

// Invoke - Entry point for invocations.
func (t *OnlyPut) Invoke(stub shim.ChaincodeStubInterface) *pb.Response {
    function, args := stub.GetFunctionAndParameters()
    switch function {
    case "invoke":
        return t.put(stub, args)
    default:
        return shim.Error("Received unknown function invocation")
    }
}

// Both params should be marshaled JSON data and base64 encoded.
func (t *OnlyPut) put(stub shim.ChaincodeStubInterface, args []string) *pb.Response {
    if len(args) != 1 {
        return shim.Error("Incorrect number of arguments. Expecting 1")
    }
    num, _ := strconv.Atoi(args[0])
    for i := 0; i < num; i++ {
        key := "key" + strconv.Itoa(i)
        if err := stub.PutState(key, []byte(key)); err != nil {
            return shim.Error(err.Error())
        }
    }
    return shim.Success(nil)
}
```

---

## Proposed Solution

The proposed solution introduces a **batching mechanism** where multiple state changes are grouped into a single `ChangeStateBatch` message. This new message type will encapsulate all state changes and transmit them as a batch to the peer node. During the `Ready` message exchange, the peer and chaincode will negotiate the capability to use batching, ensuring backward compatibility.

### New Message Format
```proto
message ChangeStateBatch {
    repeated StateKV kvs = 1;
}

message StateKV {
    enum Type {
        UNDEFINED = 0;
        PUT_STATE = 9;
        DEL_STATE = 10;
        PUT_STATE_METADATA = 21;
        PURGE_PRIVATE_DATA = 23;
    }

    string key = 1;
    bytes value = 2;
    string collection = 3;
    StateMetadata metadata = 4;
    Type type = 5;
}
```

---

### Process Flow with ASCII Diagram
The diagram below illustrates the message exchange between the chaincode and peer, showcasing the integration of the batching mechanism:

```
+------------+                                            +-----------+
| Chaincode  |                                            |    Peer   |
+------------+                                            +-----------+
       |                                                        |
       |--- Ready (UsePutStateBatch, MaxSizePutStateBatch) -->  |
       |                                                        |
       |<---- Acknowledgement Ready --------------------------  |
       |                                                        |
       |  Batch multiple commands                               |
       |                                                        |
       |--- ChangeStateBatch (Batch of Commands) -------------> |
       |                                                        |
       |<--- Acknowledgement (Success/Failure) ---------------- |
       |                                                        |
```

---

## Benchmark Results

### Before Optimization
```
Only PUT State
Name                                | N   | Min     | Median  | Mean    | StdDev  | Max    
===========================================================================================
invoke N-5 cycle-1 [duration]       | 5   | 62.9ms  | 64.2ms  | 437.8ms | 458.4ms | 999.9ms
invoke N-200 cycle-1000 [duration]  | 200 | 297ms   | 649.5ms | 651ms   | 335.6ms | 1.0139s
invoke N-200 cycle-10000 [duration] | 200 | 2.3774s | 2.4011s | 2.4037s | 16.8ms  | 2.5557s
```

### After Optimization
```
Only PUT State
Name                                | N   | Min     | Median  | Mean    | StdDev  | Max    
===========================================================================================
invoke N-5 cycle-1 [duration]       | 5   | 62.2ms  | 62.7ms  | 437.9ms | 459.7ms | 1.0013s
invoke N-200 cycle-1000 [duration]  | 200 | 71.8ms  | 541.3ms | 539.4ms | 427.6ms | 1.0085s
invoke N-200 cycle-10000 [duration] | 200 | 176.2ms | 586.4ms | 588.9ms | 336ms   | 1.002s
```
The cycle of 10000 updates rounds clearly demonstrates the performance improvement of the batching mechanism.


## Order of Repository Changes

To implement the proposed batching mechanism, modifications need to be made across multiple repositories. Below is the order in which the repositories should be updated to ensure smooth integration:

	1.	github.com/hyperledger/fabric-protos
	    •	Define the new ChangeStateBatch message format.
	    •	Ensure the protobuf schema includes all relevant message fields such as StateKV and Type.
	2.	github.com/hyperledger/fabric-chaincode-go
	    •	Update the chaincode framework to support batching logic.
	    •	Add negotiation mechanisms for the Ready message exchange to determine if the batching feature is enabled.
	    •	Modify internal functions to collect state modifications and prepare ChangeStateBatch messages for transmission.
	3.	github.com/hyperledger/fabric
	    •	Implement the handler logic to process the new ChangeStateBatch messages.
	    •	Update the peer’s transaction context to support batched state changes.
	    •	Ensure that any failure within a batch rolls back all changes, preserving consistency with existing transactional behavior.

This sequence ensures that all dependencies are resolved correctly, avoiding integration issues when introducing the batching mechanism.

---

## Conclusion
The proposed batching mechanism offers a simple yet effective way to optimize communication between Hyperledger Fabric chaincodes and peers. By reducing the number of messages exchanged, the system achieves better performance and resource utilization while maintaining transactional integrity and backward compatibility.

## Amendment

### Concerns Raised

During the RFC discussion, the following concerns were raised about the proposed batching mechanism: maintaining the consistency of read-write order, managing existing read-write relationships, ensuring compatibility with current chaincodes, and addressing the impact on interleaved operations[1][2][3]. Altering the sequence of reads and writes during batching could lead to unexpected behavior or different error messages, particularly for applications that rely on the immediate execution of commands. Additionally, operations like SetState involve implicit reads or validations, such as metadata checks or collection name verification, which are currently handled interactively. Transitioning these operations to a batched model introduces complexity and risks compromising correctness. To mitigate these risks, a thorough review of the transaction simulator code is essential, as certain write operations, such as SetState, depend on reading the current state or metadata to function as expected.

[1]: https://github.com/hyperledger/fabric-rfcs/pull/58#issuecomment-2458605718
[2]: https://github.com/hyperledger/fabric-rfcs/pull/58#issuecomment-2460109295
[3]: https://github.com/hyperledger/fabric-rfcs/pull/58#issuecomment-2473626315


### Proposed Solutions

1. Maintaining Read-Write Order

	•	Solution: Ensure that the batching mechanism processes operations in the same order as they are invoked in the chaincode. This approach can be achieved by batching only consecutive write operations and sending read operations interactively.
	•	Implementation: The chaincode shim can collect PutState or DelState operations into a batch, but any GetState operation will immediately flush the current batch and process the read interactively. This guarantees consistency with the current behavior.

2. Handling Dependencies and Validation

	•	Solution: Retain validation logic (e.g., metadata reads and collection name checks) on the peer side to ensure correctness. Avoid shifting these responsibilities to the shim, as it increases complexity and risk.
	•	Implementation: The peer should process batched operations in a way that preserves existing checks and validations, ensuring no functional changes to current read-write dependencies.

3. Explicit Developer Control

	•	Solution: Introduce explicit APIs for developers to enable and manage batching in their chaincode. This gives developers control over when and how batching is applied, allowing for easier debugging and incremental adoption.
	•	Implementation: APIs like StartBatch and FinishBatch can provide clear demarcation points for batching operations, ensuring developers understand and control the batching process.

## Final Solution

After careful consideration of the various approaches, option #3 - Explicit Developer Control - was selected as the final solution. This choice was made for several key reasons:

1. Safety and Predictability: By giving developers explicit control over batching through dedicated APIs, we minimize the risk of unexpected behavior that could arise from implicit batching mechanisms. Developers can clearly understand and control when batching occurs.

2. Debugging and Observability: The explicit nature of the batching APIs makes it easier to debug issues by providing clear entry and exit points for batched operations. This visibility is crucial for troubleshooting in production environments.

3. Gradual Adoption Path: This approach allows for incremental adoption of the batching feature. Organizations can update their chaincodes at their own pace, testing and validating the performance benefits without forcing a system-wide change.

4. Preservation of Existing Semantics: The explicit APIs ensure that the current chaincode behavior remains unchanged unless specifically opted into batching. This maintains compatibility while still offering performance improvements where desired.

5. Simplified Implementation: Having clear batching boundaries reduces the complexity of managing state consistency and validation checks, as the system knows exactly when batching begins and ends.


### Extend Chaincode Stub API with Explicit Batching APIs

Introduce new APIs in the chaincode stub to explicitly control batching operations:

```go
// StartWriteBatch initializes a new batch for write operations.
func (stub *ChaincodeStub) StartWriteBatch() {
    // Implementation to begin batching write operations.
}

// FinishWriteBatch flushes the current batch and sends all collected write operations to the peer.
// Returns a serialized set of results for each operation with the corresponding status.
func (stub *ChaincodeStub) FinishWriteBatch() error {
    // Implementation to process the batched operations.
}
```

Advantages of Explicit Batching

	1.	Developer Awareness: Developers can explicitly enable batching, ensuring they understand the impact on transaction behavior.
	2.	Incremental Adoption: Existing chaincodes can operate without modification, while developers can selectively enable batching for specific chaincodes.
	3.	Ease of Debugging: The explicit demarcation of batched operations simplifies debugging, as developers can correlate logs and behavior directly with batching commands.
	4.	Backward Compatibility: Systems without batching support will continue to operate as before, ensuring seamless integration.

### Example Usage

```go
func (t *OnlyPut) put(stub shim.ChaincodeStubInterface, args []string) *pb.Response {
    if len(args) != 1 {
        return shim.Error("Incorrect number of arguments. Expecting 1")
    }
    num, _ := strconv.Atoi(args[0])

    // Start batching write operations
    stub.StartWriteBatch()
    for i := 0; i < num; i++ {
        key := "key" + strconv.Itoa(i)
        if err := stub.PutState(key, []byte(key)); err != nil {
            return shim.Error(err.Error())
        }
    }
    // Finish batching and send all operations to the peer
    err := stub.FinishWriteBatch()
    if err != nil {
        return shim.Error(err.Error())
    }

    return shim.Success(nil)
}
```

This approach balances performance optimization with consistency, developer control, and backward compatibility.