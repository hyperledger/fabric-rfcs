---
layout: default
title: CoAP Gateway for Hyperledger Fabric
nav_order: 3
---

* Feature Name: CoAP Gateway for Hyperledger Fabric
* Start Date: 18/08/2025
* RFC PR: (leave this empty)
* Fabric Component: Ecosystem Architecture, Peer , Gateway.
* Fabric Issue: (leave this empty)

# Summary

This RFC proposes the addition of a Constrained Application Protocol (CoAP) gateway to Hyperledger Fabric. The goal is to enable IoT and resource-constrained devices to interact with Fabric networks using a lightweight, UDP-based protocol. The CoAP gateway will provide a simplified interface for submitting transactions, evaluating chaincode, and receiving events, maintaining security through DTLS encryption and certificate-based authentication.

# Motivation

The current Hyperledger Fabric gRPC gateway has limitations for IoT and resource-constrained devices due to:

* IoT devices often have limited memory, processing power, and network bandwidth, making gRPC/HTTP2 impractical.

* Many IoT deployments use UDP-based networks or have unreliable connectivity.

* gRPC requires TCP and HTTP/2 connections, which may not be suitable for constrained environments.

* IoT devices require lightweight yet secure communication protocols.

The CoAP gateway addresses these limitations by providing:

* Lightweight, UDP-based communication.

* DTLS security for encrypted and authenticated connections.

* Support for resource-constrained devices.

* Event streaming capabilities.

# Guide-level explanation

This section explains the proposed Hyperledger Fabric CoAP Gateway. It's designed to teach a Fabric developer how to understand, use, and think about integrating IoT devices with Fabric using this new capability.

For developers and users, the CoAP gateway will be perceived as a new option for integrating IoT devices into Hyperledger Fabric.

The main difference is the use of CoAP (UDP/DTLS) instead of gRPC (TCP/HTTP2), resulting in lower protocol overhead and better suitability for constrained environments. APIs will be ***similar*** to the current gRPC gateway, but using CoAP instead, with specific endpoints for transaction endorsement, submission, and evaluation, as well as event subscription.

Comprehensive documentation, configuration guides, security, performance, and troubleshooting guides will be provided to ensure users understand the capabilities and use of the CoAP gateway.

The CoAP gateway introduces a DTLS security layer and uses X.509 certificates for client authentication, mapping them to Fabric identities. Authorization will be integrated with existing Fabric ACL mechanisms.

## Connecting the Physical World to Fabric

Imagine you're building a supply chain network to track refrigerated goods. You have thousands of IoT sensors that monitor temperature and location. How do you get that data onto the blockchain in a secure, efficient, and standardized way?
Historically, this required a complex middle layer: an IoT platform or custom application server that would receive data from devices (perhaps over MQTT or CoAP), validate it, and then use a Fabric SDK to submit a transaction. This adds latency, a central point of failure, development overhead, ***and requires implicit trust on that middle layer***.
The Fabric CoAP Gateway solves this problem by creating a direct, lightweight, and secure bridge between CoAP-enabled IoT devices and the Fabric network. It allows resource-constrained devices to interact with chaincode by sending simple CoAP messages, extending the reach of Fabric to the very edge of your network.

## New Concepts

To understand the gateway, let's introduce a few new concepts:

* This is a new, optional peer service (or standalone component) that exposes a CoAP endpoint. It listens for incoming CoAP requests from registered IoT devices, translates them into Fabric transaction proposals, and returns the result to the device. Think of it as a specialized application client, like one built with the Fabric SDK, but one that speaks CoAP instead of gRPC and is optimized for IoT workloads.

* The gateway acts as a protocol converter, translating CoAP requests into the appropriate gRPC calls. This API is designed to be a direct reflection of the existing Hyperledger Fabric gRPC gateway's interface, providing a consistent experience for developers.

* Instead of mapping to dynamic URI-paths, the gateway exposes four core functions as fixed endpoints. The client is responsible for packaging all necessary information, such as the channel, chaincode name, and function arguments, within the Protobuf payload, just as they would when using the gRPC gateway directly.

Specifically, it uses:

* `/evaluate` for evaluation requests

* `/endorse` for managing endorsements

* `/submit` for processing transaction submissions

* `/events` for listening to events

* `/commit-status` for checking transaction commit status

This design simplifies the gateway's logic by reusing the existing gRPC message structures and avoids the complexities of a dynamic mapping system. This ensures the CoAP gateway is a new entry point to existing functionalities without needing format or logic transformations.

A key difference from typical REST endpoints is that these endpoints don't receive JSON strings. Instead, they accept a binary stream encoded with Protobuf, just like the gRPC gateway. This simplifies the resource-to-chaincode mapping by allowing the CoAP gateway to act as a new entry point to existing functionalities without needing format transformations.

Security is paramount. An IoT device cannot simply send data to the network. Each device must have a verifiable identity. The CoAP Gateway integrates directly with the Member Service Provider (MSP). Devices are enrolled through a Fabric CA and issued X.509 certificates, just like users or apps. These certificates are then used to establish a secure DTLS (Datagram TLS) session with the gateway, which authenticates the device and uses its identity and attributes for authorization.

## An Example: The Smart Cold Chain

Let's start with the cold chain example. A temperature sensor on a truck needs to report its reading. The device is already enrolled and has a certificate.

* The sensor wakes up, reads the temperature (e.g., -18°C). It must then create a transaction proposal, endorse it and finally submit it. For such, it will encode a protobuf transaction proposal the same way the sdk would for a gRPC endorse call and sends it to the node it is connected to:
  * Target: coap://fabric-gateway.mycorp.com:5683/endorse
  * Method: POST
  * Payload: protobuf-encoded transaction

* The gateway receives this request over a DTLS-secured connection.
  * It authenticates the device using its client certificate, identifying it as sensor-345 belonging to transporter-org.
  * It extracts the standard Fabric transaction proposal, signed with the device's identity.
  * It passes that signed proposal to the standard (gRPC) gateway for further processing.
  * With the response calculated by the standard gateway, it encapsulates it into a CoAP response, and sends it back to the device.

* Once the transaction is endorsed, the gateway sends a CoAP response back to the device.
  * Success: A 2.04 Changed response, confirming the data was recorded.
  * Failure: A 4.03 Forbidden if the endorsement policy failed, or a 5.00 Internal Server Error if a system-level problem occurred.

After the endorsement is completed, the device starts once again with the submission phase, where the process is similar to the endorsement.

## Teaching This to Different Developers

* For Established Fabric Developers: You already understand the transaction lifecycle, endorsement policies, and the MSP. The CoAP Gateway is simply a new adapter that plugs into this flow. You'll find that all your existing knowledge about chaincode development and policy definition applies directly. The main new tasks are learning the protobuf data encoding, which can be left to the sdk, and the best practices for managing IoT device identities within the MSP.

* For New Fabric Developers: Fabric is a platform for building decentralized applications with trust. The CoAP Gateway is one of the most powerful ways to get trusted data into that platform from IoT platforms. It allows you to build end-to-end solutions that securely connect a physical device to a digital ledger without needing to become an expert in complex network protocols. Start with an example like our cold chain, and see how device requests can trigger powerful, trusted logic on the blockchain.

# Reference-level explanation

This section provides the technical details of the CoAP gateway implementation. The gateway exposes a CoAP interface that mirrors the functionality of the existing gRPC gateway, allowing clients in constrained environments to interact with a Hyperledger Fabric network. The core design principle is to reuse the existing Fabric gateway logic and protocol message structures, with the CoAP server acting as a thin network wrapper.

## Protocol and Security

The gateway utilizes the Constrained Application Protocol (CoAP) over Datagram Transport Layer Security (DTLS) to ensure secure, low-overhead communication.

* **Transport Security**: All communication between the client and the CoAP gateway is secured using DTLS (CoAP-over-UDP with TLS). This provides authentication, confidentiality, and integrity for all data in transit.

* **Client Authentication**: Client identity is established via mutual DTLS, using the standard X.509 certificates issued by a Fabric Certificate Authority (CA). The gateway validates the client's certificate against the channel's MSP configuration. This means that existing client identities and certificates can be used without modification, and no separate identity management system is required for the CoAP gateway.

* **Transaction Security**: Since the gateway forwards standard Fabric protobuf messages, the transaction-level security model remains unchanged. Transaction proposals must be signed by the client, and the gateway will collect signed endorsements from peers. All security and policy enforcement (like endorsement policies and ACLs) are handled by the underlying Fabric infrastructure.

## API Endpoints

The CoAP gateway exposes five main endpoints, each corresponding to a primary function of the Fabric gateway service. Requests and responses use the same protobuf message types as the gRPC service, marshaled into a binary byte array for the CoAP payload.

### 1. Evaluate Transaction

Used to invoke a transaction function on a peer to query the ledger state without submitting a transaction for ordering.

* **Endpoint**: `POST /evaluate`
* **Request Payload**: The body of the CoAP request must contain a marshaled `gateway.EvaluateRequest` protobuf message.
* **Success Response**:
  * **Code**: `2.05 Content`
  * **Payload**: The response body contains a marshaled `gateway.EvaluateResponse` protobuf message.

### 2. Endorse Transaction

Used to collect endorsements for a transaction from peers sufficient to satisfy the chaincode's endorsement policy.

* **Endpoint**: `POST /endorse`
* **Request Payload**: The body of the CoAP request must contain a marshaled `gateway.EndorseRequest` protobuf message.
* **Success Response**:
  * **Code**: `2.05 Content`
  * **Payload**: The response body contains a marshaled `gateway.EndorseResponse` protobuf message, which includes the prepared transaction envelope ready for signing by the client.

### 3. Submit Transaction

Used to submit a previously endorsed and client-signed transaction to the ordering service.

* **Endpoint**: `POST /submit`
* **Request Payload**: The body of the CoAP request must contain a marshaled `gateway.SubmitRequest` protobuf message.
* **Success Response**:
  * **Code**: `2.04 Changed`
  * **Payload**: The response body contains a marshaled `gateway.SubmitResponse` protobuf message.

**Note**: The `/submit` endpoint waits for the transaction to be submitted to the ordering service before returning. To check whether the transaction has been committed to the ledger, clients must use the `/commit-status` endpoint.

### 4. Receive Ledger Events

Used to subscribe to a stream of chaincode events from the ledger. This endpoint uses the CoAP Observe option (RFC 7641) to deliver a stream of notifications to the client.

* **Endpoint**: `GET /events` (with the `Observe` option set to `0`)
* **Request Payload**: The body of the initial CoAP GET request must contain a marshaled `gateway.ChaincodeEventsRequest` protobuf message.
* **Response Stream**:
  * The server will register the client as an observer and send a success response (`2.05 Content`).
  * Subsequently, as new blocks containing relevant chaincode events are committed to the ledger, the server will push notifications to the client. Each notification will have a `2.05 Content` response code and a payload containing a marshaled `gateway.ChaincodeEventsResponse` message.

### 5. Commit Status

Used to check the status of a previously submitted transaction.

* **Endpoint**: `POST /commit-status`
* **Request Payload**: The body of the CoAP request must contain a marshaled `gateway.SignedCommitStatusRequest` protobuf message.
* **Success Response**:
  * **Code**: `2.05 Content`
  * **Payload**: The response body contains a marshaled `gateway.CommitStatusResponse` protobuf message.

## Error Handling

When a request fails, the CoAP gateway maps the underlying Fabric error to an appropriate CoAP response code from the `4.xx` (Client Error) or `5.xx` (Server Error) class. The payload of the error response contains a plain-text UTF-8 string describing the error.

The following table details the mapping from common Fabric errors to CoAP response codes:

| Fabric Error Condition     | CoAP Code                    | Description                                                                |
|----------------------------|------------------------------|----------------------------------------------------------------------------|
| Malformed request payload  | `4.00 Bad Request`           | The protobuf message in the request body could not be unmarshalled.        |
| Endorsement policy failure | `4.03 Forbidden`             | The transaction failed to meet the chaincode's endorsement policy.         |
| MVCC read conflict         | `4.12 Precondition Failed`   | A read-write conflict (MVCC) was detected when validating the transaction. |
| No peers available         | `5.03 Service Unavailable`   | The gateway could not connect to any endorsing peers for the request.      |
| Unspecified internal error | `5.00 Internal Server Error` | A generic or unexpected error occurred within Fabric or the gateway.       |

## Detailed Design

### Architecture Overview

The CoAP gateway will be implemented as an additional service within the Fabric peer, alongside the existing gRPC gateway. It will function as a CoAP server (UDP port 5683 or 5684 for DTLS), support DTLS 1.2/1.3 for secure communication, utilize X.509 certificates for client authentication, integrate with existing gateway services for transaction processing, and support CoAP observe for real-time event delivery.

### Configuration

Configured via the peer's `core.yaml` file. Includes options to enable/disable the gateway, address, port, DTLS settings (certFile, keyFile, clientCACertFile, requireClientCert, minVersion, maxVersion), and performance tuning (maxConcurrentRequests, requestTimeout, observeTimeout).

### API Design

The CoAP gateway will expose REST-like endpoints:

* `/endorse (POST)`: To submit transactions for endorsement.

* `/submit (POST)`: To submit an endorsed transaction for ordering.

* `/evaluate (POST)`: To evaluate chaincode without submitting to the ledger.

* `/events/{channel}` (GET with Observe option): To subscribe to chaincode events.

* `/commit-status (POST)`: To check the status of previously submitted transactions.

### Security Model

* Authentication: Based on X.509 certificates, with CA validation and identity mapping to Fabric identities.

* Authorization: Leverages existing Fabric ACL mechanisms, checking channel access and chaincode invocation permissions.

* Encryption: DTLS 1.2/1.3 for end-to-end encryption, support for Perfect Forward Secrecy, and optional certificate pinning.

### Implementation Details

**Core Components:**

* **Server** (server.go): Main CoAP server that handles incoming requests on UDP port 5683/5684
* **Authenticator** (auth.go): Validates X.509 device certificates and checks Fabric ACL permissions
* **EndorseHandler** (endorse.go): Creates and submits transaction proposals to endorsing peers
* **SubmitHandler** (submit.go): Submits endorsed transactions to the orderer
* **CommitStatusHandler** (commit_status.go): Checks the status of previously submitted transactions
* **EvaluateHandler** (evaluate.go): Executes chaincode queries without submitting to ledger
* **EventStreamer** (events.go): Manages CoAP observe subscriptions for chaincode events
* **ResourceMapper** (mapper.go): Maps CoAP URI paths to specific channel/chaincode/function calls

**Integration Points:**

* **Gateway Service**: Reuses existing gateway logic for transaction proposal creation and endorsement
* **MSP Integration**: Leverages Fabric MSP for device identity validation and signing
* **Event System**: Integrates with Fabric event delivery system for real-time notifications
* **Configuration**: Extends core.yaml with CoAP gateway settings (ports, DTLS, mappings)

**Error Handling:**

* Uses standard CoAP response codes (2.01 Created, 2.05 Content, 4.01 Unauthorized, 4.03 Forbidden, 4.04 Not Found, 5.00 Internal Server Error)
* Returns JSON error responses with transaction IDs and diagnostic messages

**Performance Considerations:**

* DTLS connection pooling to reuse secure connections across requests
* Request batching to process multiple operations in single network round-trip
* Query result caching to avoid repeated chaincode calls for same data
* Configurable limits for concurrent connections (default: 1000) and request timeouts (default: 30s)

## Impact Analysis

### Breaking Changes

    None. This RFC does not introduce breaking changes to existing Fabric functionality.

### Backward Compatibility

    The CoAP gateway is an optional feature and disabled by default, which does not affect existing Fabric deployments. gRPC gateway functionalities remain unchanged, and both can operate simultaneously.

### Performance Impact

    Minimal. The CoAP gateway runs independently, with separate resource allocation to avoid interference with core peer operations. Configurable limits can be set for resource usage.

### Security Impact

    Improved security by adding a DTLS security layer for IoT communications, without degrading existing security measures. Certificate management for CoAP clients is separate.

## Resource Requirements

* Memory Requirements: ~50-100MB additional per peer, with ~10-20MB per 100 concurrent CoAP connections and ~5-10MB for certificate and policy caching.

* CPU Requirements: Additional overhead for DTLS handshakes and encryption, minimal impact for request routing and validation, and moderate usage for event streaming.

* Network Requirements: Standard UDP port 5683 (or 5684 for DTLS), minimal additional bandwidth for lightweight CoAP messages, and configurable limits for concurrent connections.

* Storage Requirements: Space for server and client certificates, minimal additional configuration storage, and additional storage for CoAP gateway operation logs.

# Drawbacks

* Adds another protocol layer to Fabric.

* DTLS adds computational overhead.

* Not all gRPC gateway functionalities will be available.

* Additional code to maintain and test.

* CoAP has inherent limitations compared to gRPC.

# Rationale and alternatives

## Why CoAP?

It is an established IETF standard (RFC 7252), designed specifically for resource-constrained devices, has native DTLS support, offers the Observe pattern for event streaming, and a familiar REST-like API design.

## Alternatives Considered

* MQTT: More complex, requires a message broker.

* Custom UDP Protocol: Would require custom security implementation.

* HTTP/1.1: Higher overhead, no native event streaming.

* gRPC over HTTP/1.1: Would defeat the purpose of a lightweight protocol.

# Prior art

The concept of a CoAP gateway and its implementations are well-established:

Eclipse Californium: CoAP implementation in Java.

go-coap: Go CoAP library used in this implementation.

IoT Edge: Microsoft's IoT edge computing platform.

AWS IoT: Amazon's IoT platform with CoAP support.

# Testing

The testing strategy for the CoAP gateway will include:

* Unit Tests: Tests of individual components.

* Integration Tests: End-to-end workflow tests.

* Performance Tests: Load and stress tests.

* IoT Device Testing: Tests with real constrained devices.

# Unresolved Questions

Performance Benchmarks: What are the performance characteristics compared to gRPC?

Monitoring: How to monitor and debug CoAP connections?

## Implementation Plan

The implementation plan is divided into phases:

Phase 1: Core Implementation: Basic CoAP server with DTLS, authentication and authorization, basic transaction operations (endorse, submit, evaluate), documentation, and deployment.

Phase 2: Event Streaming: CoAP observe implementation and event streaming optimization.

Phase 3: Advanced Features: Batch requests, monitoring and metrics, advanced functionalities, and testing.

## References

RFC 7252 - The Constrained Application Protocol (CoAP)

RFC 6347 - Datagram Transport Layer Security Version 1.2

Hyperledger Fabric Gateway Documentation

go-coap Library

Eclipse Californium
