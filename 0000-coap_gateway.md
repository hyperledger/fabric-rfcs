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

The CoAP gateway introduces a **dual-certificate security model** that separates connection authentication from transaction authorization:

* **DTLS Certificate**: Validates the device can connect to the gateway (checked during connection establishment)

* **Signing Certificate**: Proves the device is authorized to execute specific transactions (checked by Fabric ACLs and endorsement policies)

Both certificates are issued by Fabric CAs and validated against the channel's MSP configuration.

## Connecting the Physical World to Fabric

Imagine you're building a supply chain network to track refrigerated goods. You have thousands of IoT sensors that monitor temperature and location. How do you get that data onto the blockchain in a secure, efficient, and standardized way?
Historically, this required a complex middle layer: an IoT platform or custom application server that would receive data from devices (perhaps over MQTT or CoAP), validate it, and then use a Fabric SDK to submit a transaction. This adds latency, a central point of failure, development overhead, ***and requires implicit trust on that middle layer***.

The Fabric CoAP Gateway solves this problem by creating a direct, lightweight, and secure bridge between CoAP-enabled IoT devices and the Fabric network. Using automatically generated client libraries, resource-constrained devices can interact with chaincode using the same APIs as traditional gRPC clients, but over the lightweight CoAP protocol. This extends the reach of Fabric to the very edge of your network without sacrificing developer experience or security.

## New Concepts

To understand the gateway, let's introduce a few new concepts:

* This is a new, **optional** peer service that exposes a CoAP endpoint. It listens for incoming CoAP requests from registered IoT devices, translates them into calls to the existing gRPC Gateway implementation, and returns the result to the device. Think of it as a thin protocol adapter that makes the Fabric Gateway accessible to CoAP clients without duplicating any business logic.

* The gateway acts as a lightweight protocol converter, translating CoAP requests into in-process gRPC Gateway calls. This API is an exact reflection of the existing Hyperledger Fabric gRPC gateway's interface, providing a consistent experience for developers across protocols.

* The gateway exposes RPC-style endpoints that mirror the Gateway service methods. The client is responsible for packaging all necessary information, such as the channel, chaincode name, and function arguments, within the Protobuf payload, just as they would when using the gRPC gateway directly.

Specifically, it exposes:

* `/rpc/gateway.Gateway/Evaluate` for evaluation requests

* `/rpc/gateway.Gateway/Endorse` for managing endorsements

* `/rpc/gateway.Gateway/Submit` for processing transaction submissions

* `/rpc/gateway.Gateway/ChaincodeEvents` for listening to events

* `/rpc/gateway.Gateway/CommitStatus` for checking transaction commit status

This design maximizes code reuse by delegating all business logic to the existing GatewayServer implementation. The CoAP adapter is purely a protocol translation layer, ensuring consistency and minimal maintenance overhead.

A key difference from typical REST endpoints is that these endpoints don't receive JSON strings. Instead, they accept a binary stream encoded with Protobuf, just like the gRPC gateway. This simplifies the resource-to-chaincode mapping by allowing the CoAP gateway to act as a new entry point to existing functionalities without needing format transformations.

Security is paramount. An IoT device cannot simply send data to the network. Each device must have a verifiable identity. The CoAP Gateway integrates directly with the Member Service Provider (MSP). Devices are enrolled through a Fabric CA and issued two X.509 certificates: one for DTLS transport security and one for signing transactions. These certificates establish a secure DTLS (Datagram TLS) session with the gateway and authenticate the device's transactions through Fabric's standard validation mechanisms.

## Client Generation and Usage

The CoAP gateway provides the same developer experience as gRPC through automated client generation. When fabric-protos is built, a protoc plugin (`protoc-gen-go-coap`) generates type-safe Go clients that mirror the gRPC gateway interface.

Developers simply import the generated client:

```go
import "github.com/hyperledger/fabric-protos-go-apiv2/gateway"

// Create CoAP connection with DTLS certificate
conn, err := coap.NewDTLSConnection("fabric-gateway.mycorp.com:5684",
    coap.WithDTLSCert(transportCert, transportKey),
    coap.WithRootCA(caCert))

// Use generated client
client := gateway.NewGatewayCoapClient(conn)

// Type-safe method calls with IDE autocomplete
resp, err := client.Endorse(ctx, &gateway.EndorseRequest{
    TransactionId: "tx123",
    ChannelId:     "mychannel",
    ProposedTransaction: signedProposal,  // Signed with identity certificate
})
```

This approach ensures that:

* Client code is automatically in sync with protocol changes
* Developers get full IDE support (autocomplete, type checking)
* The experience matches gRPC client patterns exactly
* When Gateway API changes, regeneration is automatic

## An Example: The Smart Cold Chain

Let's start with the cold chain example. A temperature sensor on a truck needs to report its reading. The device has been enrolled and provisioned with:

* **DTLS certificate**: For secure connection (`device-345-dtls.pem`)
* **Signing certificate**: For transaction signing (`device-345-identity.pem`)

* The sensor wakes up, reads the temperature (e.g., -18°C). It must then create a transaction proposal, endorse it and finally submit it:
  * Establishes DTLS connection using its **transport certificate**
  * Creates and signs the proposal using its **signing certificate**
  * **Target:** coaps://fabric-gateway.mycorp.com:5684/rpc/gateway.Gateway/Endorse
  * **Method:** POST
  * **Payload:** protobuf-encoded `SignedProposal` with embedded signing certificate

* The gateway receives this request over the DTLS-secured connection:
  * **Stage 1**: Validates the DTLS certificate - confirms device is allowed to connect
  * **Stage 2**: Extracts the signing certificate from `SignedProposal` and validates:
    * Signature is valid
    * Signing certificate belongs to `transporter-org` MSP
    * Device has permission per channel ACLs
  * Forwards the validated proposal to existing gRPC gateway implementation
  * Returns CoAP response with endorsement

* Once endorsed, the gateway sends a CoAP response:
  * **Success:** `2.05 Content` with marshaled `EndorseResponse`
  * **Failure:** `4.03 Forbidden` if ACL check failed, `5.00 Internal Server Error` for system issues

After the endorsement is completed, the device starts once again with the submission phase, where the process is similar to the endorsement.

## Teaching This to Different Developers

* **For Established Fabric Developers:** The CoAP Gateway is simply a new protocol adapter that plugs into the existing Gateway implementation with zero changes to business logic. The main new concepts are: (1) the dual-certificate model for separating connection and transaction security and (2) using the generated CoAP client from `fabric-protos-go-apiv2`, which works exactly like the gRPC client you're already familiar with.

* **For IoT Developers New to Fabric:** The CoAP Gateway enables direct integration of resource-constrained devices with blockchain networks. The dual-certificate approach is straightforward: one certificate connects you to the gateway, and another signs your transactions. Start with an example like our cold chain to see how device data flows securely onto the ledger.

# Reference-level explanation

This section provides the technical details of the CoAP gateway implementation. The gateway exposes a CoAP interface that mirrors the functionality of the existing gRPC gateway, allowing clients in constrained environments to interact with a Hyperledger Fabric network. The core design principle is to reuse the existing Fabric gateway logic and protocol message structures, with the CoAP server acting as a thin network wrapper.

## Protocol and Security

The gateway utilizes the Constrained Application Protocol (CoAP) over Datagram Transport Layer Security (DTLS) to ensure secure, low-overhead communication. Security is implemented through a **dual-certificate model** that separates transport-level authentication from transaction-level authorization.

### Dual-Certificate Architecture

Devices are provisioned with **two distinct X.509 certificate/key pairs**, both issued by a Fabric Certificate Authority (CA):

* **Transport Layer**: DTLS 1.2/1.3 with mutual X.509 authentication (connection security)  
* **Application Layer**: Signed transaction proposals with identity certificates (transaction authorization)

This is identical to the gRPC gateway's security model - the CoAP adapter validates DTLS certificates during connection establishment, then delegates all transaction-level security (ACL checks, endorsement policies) to the existing GatewayServer implementation.

Devices require two certificates from Fabric CA:

* **DTLS certificate:** For UDP transport authentication
* **Identity certificate:** For signing `SignedProposal` payloads (embedded in request body)

Both certificates are validated against channel MSP configuration.
Compromised transport certificates cannot sign transactions;
compromised identity certificates cannot intercept network traffic.

## API Endpoints

The CoAP gateway exposes five main endpoints corresponding to the Fabric Gateway service. All paths follow the pattern `/rpc/gateway.Gateway/{Method}` and are automatically generated as constants in `fabric-protos-go-apiv2`.

**Generated Path Constants** (in `gateway_coap.pb.go`):

```go
const (
    Gateway_Endorse_CoAPPath      = "/rpc/gateway.Gateway/Endorse"
    Gateway_Submit_CoAPPath       = "/rpc/gateway.Gateway/Submit"
    Gateway_CommitStatus_CoAPPath = "/rpc/gateway.Gateway/CommitStatus"
    Gateway_Evaluate_CoAPPath     = "/rpc/gateway.Gateway/Evaluate"
    Gateway_ChaincodeEvents_CoAPPath = "/rpc/gateway.Gateway/ChaincodeEvents"
)
```

Requests and responses use the same protobuf message types as the gRPC service, marshaled into binary payloads.

### 1. Evaluate Transaction

Used to invoke a transaction function on a peer to query the ledger state without submitting a transaction for ordering.

* **Endpoint**: `POST /rpc/gateway.Gateway/Evaluate`
* **Request Payload**: The body of the CoAP request must contain a marshaled `gateway.EvaluateRequest` protobuf message.
* **Success Response**:
  * **Code**: `2.05 Content`
  * **Payload**: The response body contains a marshaled `gateway.EvaluateResponse` protobuf message.

### 2. Endorse Transaction

Used to collect endorsements for a transaction from peers sufficient to satisfy the chaincode's endorsement policy.

* **Endpoint**: `POST /rpc/gateway.Gateway/Endorse`
* **Request Payload**: The body of the CoAP request must contain a marshaled `gateway.EndorseRequest` protobuf message.
* **Success Response**:
  * **Code**: `2.05 Content`
  * **Payload**: The response body contains a marshaled `gateway.EndorseResponse` protobuf message, which includes the prepared transaction envelope ready for signing by the client.

### 3. Submit Transaction

Used to submit a previously endorsed and client-signed transaction to the ordering service.

* **Endpoint**: `POST /rpc/gateway.Gateway/Submit`
* **Request Payload**: The body of the CoAP request must contain a marshaled `gateway.SubmitRequest` protobuf message.
* **Success Response**:
  * **Code**: `2.04 Changed`
  * **Payload**: The response body contains a marshaled `gateway.SubmitResponse` protobuf message.

**Note**: The `/rpc/gateway.Gateway/Submit` endpoint waits for the transaction to be submitted to the ordering service before returning. To check whether the transaction has been committed to the ledger, clients must use the `/rpc/gateway.Gateway/CommitStatus` endpoint.

### 4. Receive Ledger Events

Used to subscribe to a stream of chaincode events from the ledger. This endpoint uses the CoAP Observe option (RFC 7641) to deliver a stream of notifications to the client.

* **Endpoint**: `GET /rpc/gateway.Gateway/ChaincodeEvents` (with the `Observe` option set to `0`)
* **Request Payload**: The body of the initial CoAP GET request must contain a marshaled `gateway.ChaincodeEventsRequest` protobuf message.
* **Response Stream**:
  * The server will register the client as an observer and send a success response (`2.05 Content`).
  * Subsequently, as new blocks containing relevant chaincode events are committed to the ledger, the server will push notifications to the client. Each notification will have a `2.05 Content` response code and a payload containing a marshaled `gateway.ChaincodeEventsResponse` message.

### 5. Commit Status

Used to check the status of a previously submitted transaction.

* **Endpoint**: `POST /rpc/gateway.Gateway/CommitStatus`
* **Request Payload**: The body of the CoAP request must contain a marshaled `gateway.SignedCommitStatusRequest` protobuf message.
* **Success Response**:
  * **Code**: `2.05 Content`
  * **Payload**: The response body contains a marshaled `gateway.CommitStatusResponse` protobuf message.

## Developer Experience

### Client-Side Development

External developers working with the CoAP gateway get the same ergonomic experience as gRPC:

#### **Step 1: Import generated client**

  ```go
  import "github.com/hyperledger/fabric-protos-go-apiv2/gateway"
  ```

#### **Step 2: Create connection and client**

  ```go
  conn, err := coap.NewDTLSConnection("fabric-gateway.example.com:5684",
      coap.WithDTLSCert(transportCert, transportKey),  // DTLS certificate
      coap.WithRootCA(caCert))
  if err != nil {
      log.Fatal(err)
  }

  client := gateway.NewGatewayCoapClient(conn)
  ```

#### **Step 3: Use type-safe methods**

  ```go
  // Full IDE autocomplete and type safety
  endorseResp, err := client.Endorse(ctx, &gateway.EndorseRequest{
      TransactionId: txID,
      ChannelId:     "mychannel",
      ProposedTransaction: signedProposal,  // Signed with identity certificate
  })
  ```

### Server-Side Development

For Fabric core developers, the CoAP gateway requires minimal implementation:

1. **No changes** to existing `GatewayServer` implementation
2. **Thin adapter layer** in `fabric/internal/peer/gateway/coap/`
3. **Standard patterns** for all handlers (unmarshal → call gRPC → marshal)
4. **Reuse** existing tests and validation logic

When the Gateway service adds new methods:

* Protocol buffer definition updated
* Run `make genprotos` in fabric-protos
* `gateway_coap.pb.go` automatically regenerates
* Add simple handler in adapter following existing pattern
* External clients automatically get new methods

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

The CoAP gateway is implemented as a **lightweight adapter within the Fabric peer** that proxies requests to the existing gRPC Gateway implementation.

**Key Design Principles**:

* All transaction processing, endorsement, and validation logic remains in the existing `GatewayServer`
* CoAP adapter only handles protocol translation (CoAP ↔ gRPC)
* No network overhead between CoAP adapter and gRPC Gateway
* DTLS identity flows seamlessly to Fabric ACL checks

**Component Structure**:

```bash
fabric/
├── internal/peer/gateway/
│   ├── gateway.go              # Existing GatewayServer (UNCHANGED)
│   ├── endorser.go             # Existing endorsement logic (UNCHANGED)
│   ├── evaluator.go            # Existing query logic (UNCHANGED)
│   └── coap/                   # NEW: CoAP adapter
│       ├── adapter.go          # CoAP-to-gRPC translation
│       ├── server.go           # DTLS server and routing
│       └── auth.go             # DTLS certificate validation
└── core/peer/
    └── peer.go                 # Peer startup: initialize CoAP listener

fabric-protos/
└── bindings/go-apiv2/gateway/
    ├── gateway.pb.go           # Existing protobuf definitions
    ├── gateway_grpc.pb.go      # Existing gRPC client
    └── gateway_coap.pb.go      # NEW: Generated CoAP client
```

**Request Flow**:

```bash
IoT Device (CoAP client)
    ↓ DTLS connection (transport cert validation)
CoAP Adapter (fabric/internal/peer/gateway/coap/)
    ↓ Extract SignedProposal
    ↓ In-process call
Existing GatewayServer (fabric/internal/peer/gateway/)
    ↓ Validate signing cert, check ACLs
    ↓ Process transaction
Endorsing Peers / Orderer
```

**Server Configuration**:

* UDP port 5683 (CoAP) or 5684 (CoAPs/DTLS)
* DTLS 1.2/1.3 support
* X.509 certificate validation via MSP
* Configurable via peer's `core.yaml`

### Peer Lifecycle Integration

**Startup Sequence** (in `core/peer/peer.go`):

```go
1. Initialize MSP
2. Start event hub
3. Initialize gRPC Gateway
4. If peer.gateway.coap.enabled:
   - Validate CoAP configuration
   - Initialize DTLS certificates
   - Create CoAP adapter with reference to GatewayServer
   - Start CoAP listener on configured port
```

**Graceful Shutdown**:

* Stop accepting new CoAP connections
* Wait for in-flight requests (with timeout from config)
* Close DTLS connections
* Shutdown event observers

**Dependencies**: Requires MSP and GatewayServer to be initialized first

### Integration with fabric-protos Build Pipeline

The CoAP gateway leverages the existing fabric-protos infrastructure to automatically generate client libraries.

**buf.gen.yaml Configuration**:

```yaml
plugins:
  - local: protoc-gen-go-grpc
    out: bindings/go-apiv2
    opt:
      - paths=source_relative
      - require_unimplemented_servers=false
  - local: protoc-gen-go-coap      # New plugin
    out: bindings/go-apiv2
    opt:
      - paths=source_relative
```

**Build Process**:

1. `make genprotos` in fabric-protos repository
2. Plugin generates `gateway_coap.pb.go` in `bindings/go-apiv2/gateway/`
3. Generated files are published to `fabric-protos-go-apiv2` via existing release process
4. External clients import the package and get fully typed CoAP clients

**Benefits**:

* Zero manual maintenance of client code
* Automatic sync with protocol changes
* Consistent experience across gRPC, Java, and Node.js clients
* Full IDE support for external developers

### Configuration

Configured via the peer's `core.yaml` file:

```yaml
peer:
  gateway:
    coap:
      enabled: true
      address: 0.0.0.0
      port: 5684
      dtls:
        # Transport-level certificates (connection authentication)
        certFile: /etc/hyperledger/fabric/tls/server.crt
        keyFile: /etc/hyperledger/fabric/tls/server.key
        clientCACerts: /etc/hyperledger/fabric/tls/ca.crt
        requireClientCert: true
      # Performance tuning
      maxConcurrentRequests: 1000
      requestTimeout: 30s
      observeTimeout: 300s
```

### Security Model

The security model implements a dual-certificate architecture with two-stage validation:

* **Transport Authentication**: Based on X.509 DTLS certificates for connection establishment. Validated during DTLS handshake against channel MSP configuration. Determines if device can connect to the gateway.

* **Transaction Authorization**: Based on X.509 signing certificates embedded in `SignedProposal` payloads. Validated by Fabric's standard mechanisms, checking ACLs and endorsement policies. Determines if device can execute specific chaincode operations.

* **Encryption**: DTLS 1.2/1.3 for end-to-end encryption, support for Perfect Forward Secrecy, and mutual certificate authentication.

* **Separation of Concerns**: Transport and signing certificates are provisioned separately, allowing independent lifecycle management and minimizing security impact of certificate compromise.

### Implementation Details

#### **protoc-gen-go-coap Plugin:**

* Integrated into fabric-protos `buf.gen.yaml` configuration
* Generates `gateway_coap.pb.go` alongside `gateway_grpc.pb.go`
* Creates type-safe client interfaces and path constants
* Published to `fabric-protos-go-apiv2` automatically

#### **Generated Output Structure:**

```go
// Path constants for CoAP URIs
const (
    Gateway_Endorse_CoAPPath      = "/rpc/gateway.Gateway/Endorse"
    Gateway_Submit_CoAPPath       = "/rpc/gateway.Gateway/Submit"
    Gateway_CommitStatus_CoAPPath = "/rpc/gateway.Gateway/CommitStatus"
    Gateway_Evaluate_CoAPPath     = "/rpc/gateway.Gateway/Evaluate"
)

// GatewayCoapClient interface
type GatewayCoapClient interface {
    Endorse(ctx context.Context, in *EndorseRequest, opts ...CoapOption) (*EndorseResponse, error)
    Submit(ctx context.Context, in *SubmitRequest, opts ...CoapOption) (*SubmitResponse, error)
    CommitStatus(ctx context.Context, in *SignedCommitStatusRequest, opts ...CoapOption) (*CommitStatusResponse, error)
    Evaluate(ctx context.Context, in *EvaluateRequest, opts ...CoapOption) (*EvaluateResponse, error)
}
```

#### **protoc Plugin Implementation**

The `protoc-gen-go-coap` plugin uses `google.golang.org/protobuf/compiler/protogen`:

**Input**: Service definitions from `gateway.proto`  
**Output**: `gateway_coap.pb.go` with:

* Path constants (pattern: `/rpc/{package}.{service}/{method}`)
* Client interface with method signatures matching gRPC
* Client implementation with CoAP POST requests (unary) and GET+Observe (streaming)

**Code Generation Logic**:

```go
for each service method:
  - Generate path constant
  - If unary: POST request with protobuf body
  - If streaming: GET request with Observe option
  - Marshal request to protobuf bytes
  - Unmarshal response from protobuf bytes
```

**File Naming**: `{basename}_coap.pb.go` (matches gRPC pattern)

#### **Core Server Components** (within Fabric peer)

* **CoAP Adapter** (`fabric/internal/peer/gateway/coap/adapter.go`): Lightweight proxy that translates CoAP requests to gRPC Gateway calls
* **CoAP Server** (`fabric/internal/peer/gateway/coap/server.go`): DTLS server initialization and request routing
* **DTLS Authenticator** (`fabric/internal/peer/gateway/coap/auth.go`): Validates transport certificates during DTLS handshake
* **Existing Gateway** (`fabric/internal/peer/gateway/gateway.go`): UNCHANGED - all business logic remains here

Each handler follows a simple pattern:

```go
func (a *CoAPAdapter) handleEndorse(w coap.ResponseWriter, req *coap.Request) {
    // 1. Unmarshal CoAP request
    request := &gateway.EndorseRequest{}
    proto.Unmarshal(req.Body(), request)
    
    // 2. Call existing gRPC implementation (all business logic here)
    response, err := a.grpcGateway.Endorse(req.Context(), request)
    
    // 3. Marshal CoAP response
    payload, _ := proto.Marshal(response)
    w.Write(payload)
}
```

**Integration Points:**

* **Gateway Service**: All handlers delegate to existing `GatewayServer` implementation - zero business logic duplication
* **MSP Integration**: Leverages Fabric MSP for both DTLS certificate validation and signing certificate validation
* **Event System**: Integrates with Fabric event delivery system using CoAP Observe for real-time notifications
* **Configuration**: Extends core.yaml with CoAP gateway settings (ports, DTLS, MSP)
* **Context Flow**: DTLS peer certificate is extracted and added to context during handshake. This context flows to GatewayServer where the signing certificate from `SignedProposal` is validated. Both identities are logged for audit trails.
* **Timeout Handling**: CoAP request timeout is enforced first. If request times out before gRPC call completes, context is cancelled and client receives `5.03 Service Unavailable`.
* **Trace Propagation**: Request ID generated per CoAP request, passed to gRPC context for distributed tracing.

**Error Handling:**

Error mapping from gRPC to CoAP response codes:

| gRPC Status Code | CoAP Response Code | Scenario |
| ---------------- | ------------------ | -------- |
| OK | 2.05 Content | Success |
| INVALID_ARGUMENT | 4.00 Bad Request | Malformed protobuf |
| UNAUTHENTICATED | 4.01 Unauthorized | Invalid signature |
| PERMISSION_DENIED | 4.03 Forbidden | ACL failure |
| NOT_FOUND | 4.04 Not Found | Channel/chaincode not found |
| ABORTED | 4.12 Precondition Failed | MVCC conflict |
| UNAVAILABLE | 5.03 Service Unavailable | No peers available |
| Other | 5.00 Internal Server Error | Unexpected errors |

Error details from gRPC status message are included in CoAP response payload as UTF-8 text.

**Performance Considerations:**

* DTLS connection pooling to reuse secure connections across requests
* In-process communication eliminates network overhead between adapter and GatewayServer
* Configurable limits for concurrent connections (default: 1000) and request timeouts (default: 30s)
* Separate goroutine pools for CoAP and gRPC to prevent resource contention

## Impact Analysis

### Breaking Changes

  None. This RFC does not introduce breaking changes to existing Fabric functionality.

### Backward Compatibility

  The CoAP gateway is an optional feature and disabled by default, which does not affect existing Fabric deployments. gRPC gateway functionalities remain unchanged, and both can operate simultaneously.

### Performance Impact

  Minimal. The CoAP gateway runs independently, with separate resource allocation to avoid interference with core peer operations. Configurable limits can be set for resource usage.

### Security Impact

  Improved security by implementing a dual-certificate model that separates transport authentication from transaction authorization. DTLS provides network-layer security for IoT communications, while Fabric's standard SignedProposal validation ensures transaction-level security. This separation follows the principle of least privilege and minimizes the impact of certificate compromise. Certificate management for CoAP clients leverages existing Fabric CA infrastructure.

## Resource Requirements

* **Memory Requirements**: ~30-50MB additional per peer for the CoAP adapter, with ~10-20MB per 100 concurrent DTLS connections for connection state and certificate caching.

* **CPU Requirements**: Additional overhead for DTLS handshakes and encryption (primary cost), minimal overhead for protocol translation due to in-process communication with GatewayServer, moderate usage for CoAP Observe event streaming.

* **Network Requirements**: UDP port 5684 for DTLS (5683 for plain CoAP if needed), minimal additional bandwidth as CoAP messages are smaller than gRPC/HTTP2, configurable limits for concurrent connections (default: 1000).

* **Storage Requirements**: Space for server DTLS certificates (separate from gRPC TLS certs), MSP configuration for validating client signing certificates (shared with gRPC gateway), minimal additional configuration in core.yaml, CoAP gateway operation logs integrated with peer logging.

# Drawbacks

* DTLS handshakes and encryption add CPU overhead, particularly for constrained devices with limited crypto acceleration.

* Requires maintaining `protoc-gen-go-coap` plugin and coordinating releases with fabric-protos updates.

* CoAP/UDP has inherent limitations compared to gRPC/TCP (no connection state, message size limits, potential packet loss on unreliable networks).

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

* **Code Generation Tests**: Verify `protoc-gen-go-coap` plugin generates correct client interfaces, path constants, and marshaling code. Test with various protobuf service definitions.
* **Unit Tests**: Test individual adapter components (DTLS authentication, request unmarshaling, error mapping, response marshaling).
* **Integration Tests**: End-to-end workflow tests using generated CoAP client, covering all Gateway service methods (endorse, submit, evaluate, commit-status, events).
* **Security Tests**: Verify dual-certificate validation (DTLS handshake with transport cert, SignedProposal validation with signing cert). Test certificate revocation, expiration, and invalid scenarios.
* **Performance Tests**: Load and stress tests comparing CoAP adapter overhead vs. direct gRPC. Test concurrent connection handling and DTLS handshake performance.
* **IoT Device Testing**: Tests with real constrained devices to validate memory usage and protocol compatibility.

# Unresolved Questions

**Performance Benchmarks**: What are the performance characteristics of the CoAP adapter compared to direct gRPC? What is the overhead of the in-process protocol translation?

**Monitoring**: How should CoAP connections be monitored and debugged? Should metrics be integrated into existing peer metrics endpoints?

## Implementation Plan

The implementation plan is divided into phases:

### **Phase 1: Code Generation and Core Infrastructure**

* Develop `protoc-gen-go-coap` plugin for fabric-protos
* Integrate plugin into `buf.gen.yaml` configuration
* Generate initial `gateway_coap.pb.go` with client interfaces and path constants
* Implement basic CoAP adapter structure in `fabric/internal/peer/gateway/coap/`
* Implement dual-certificate DTLS authentication
* Documentation for client generation and certificate provisioning

### **Phase 2: Transaction Operations**

* Implement CoAP-to-gRPC adapter handlers (endorse, submit, evaluate, commit-status)
* Integrate two-stage validation (DTLS cert + signing cert)
* Add error mapping from gRPC to CoAP response codes
* End-to-end testing with generated client
* Performance benchmarking

### **Phase 3: Event Streaming and Advanced Features**

* Implement CoAP Observe for chaincode events
* Event streaming optimization
* Monitoring and metrics integration
* Load testing and optimization
* Security audit
* Production deployment guides

## References

**CoAP Protocol Specifications:**

* RFC 7252 - The Constrained Application Protocol (CoAP)
* RFC 7641 - Observing Resources in the Constrained Application Protocol (CoAP)
* RFC 6347 - Datagram Transport Layer Security Version 1.2

**Hyperledger Fabric:**

* Hyperledger Fabric Gateway Documentation
* Hyperledger Fabric Certificate Authority Documentation
* fabric-protos Repository and Build Pipeline

**Protocol Buffers and Code Generation:**

* Protocol Buffers Documentation
* protoc Plugin Development Guide
* protogen Package (google.golang.org/protobuf/compiler/protogen)

**CoAP Implementations:**

* go-coap Library (github.com/plgd-dev/go-coap)
* Eclipse Californium (Java CoAP Framework)
* libcoap (C CoAP Library)

**Security Models:**

* X.509 Certificate Best Practices for IoT
* DTLS 1.2/1.3 Implementation Guidelines
* Principle of Least Privilege in Certificate Management
