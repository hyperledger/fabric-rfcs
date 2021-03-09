---
layout: default
title: OpenTelemetry Integration
nav_order: 3
---

- Feature Name: OpenTelemetry Integration
- Start Date: 2021-01-12
- RFC PR: (leave this empty)
- Fabric Component: core, sdks
- Fabric Issue: (leave this empty)

# Summary
[summary]: #summary

This request for comments proposes integrating Hyperledger Fabric, its SDKs, peers and orderers components 
with the OpenTelemetry project.
It introduces the concept of tracing, reporting the transmission of messages to help correlate activities with the chain.

# Motivation
[motivation]: #motivation

The OpenTelemetry project is a CNCF project that aims to standardize observability, especially in a cloud-native environment.
OpenTelemetry is working to create a standard around representing metrics, traces and logs.

We aim to bring full observability of Hyperledger Fabric to allow tracing of messages between peers, orderers and clients initiating transactions.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Observability comes from the DevOps world, and is defined in [Wikipedia](https://en.wikipedia.org/wiki/Observability) as "a measure of how well internal states of a system 
can be inferred from knowledge of its external outputs".
Software that is observable exposes its behaviors by sending traces, which represent the path of execution of a particular request.
Each trace can be represented as a set of spans that are correlated as child/parent or sequential relationships.
Observable software is exposing metrics, representing the internal state of the components, as well as logs, emitted from the execution of the software.

In Hyperledger Fabric, we rely on the OpenTelemetry framework to report traces and correlate them across services.

In practice, this means a request made by a client connected via the Fabric SDK to a node can pass on the context of a trace.
Peers and orderers propagate the trace context and create spans indicating their own interaction.

Blockchain operators can reconstitute a graph of the interaction of all the components at play to create a service map.
This helps uncover trends and issues of performance, as well as shortening the time it takes to investigate problems.
It offers some security capabilities, such as detecting unexpected executions, or react quickly to metric changes.

Hyperledger Fabric developers can take advantage of those techniques with no code changes.
Each SDK execution generates a top-level trace and will report to an endpoint provided by the environment.

Developers may also create a trace before calling out to Fabric in their client code.
The current trace information will be sent along with the message to the peer.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Message headers

The trace information is passed in as an optional gRPC metadata header.

## Changes to client SDKs

Client SDKs automatically create a trace to capture the call to the chain.

The trace is created with the kind Span.Client.

If a current trace exists, the new client trace adds it as its parent.

The trace and span ID must be sent to the chain as a message header.

The SDK should use the standard environment variable environment to let users define how and if they want to report
trace data to an endpoint of their choosing.

## Changes to peers and orderers

Peers and orderers capture and propagate trace information using an optional gRPC metadata header, if enabled.

# Drawbacks
[drawbacks]: #drawbacks

OpenTelemetry has reached 1.0. Metrics and logs are still considered in alpha stage.
However, traces and metrics are well ahead of logs, and Java, Go and Javascript are well supported.

The OpenTelemetry reporting system happens securely over Protobuf, with containers and client applications sending data.
This requires that an OpenTelemetry-compatible endpoint is present to receive the data.

# Rationale and alternatives
[alternatives]: #alternatives

This design allows full observability of Hyperledger Fabric, to a degree of detail that will help developers understand
the impact of their deployment topology and organize operations using the latest framework. This allows Hyperledger Fabric
to report meaningful data just like cloud-native applications.

Alternatively, developers can develop their own homegrown designs to report data along the way, or use logs only
to understand how the system performed and investigate issues.

# Prior art
[prior-art]: #prior-art

We have [brought a similar design to Hyperledger Besu](https://github.com/hyperledger/besu/pull/1557), 
where we started adding tracing to the HTTP JSON-RPC service
used by clients to communicate with the node. We also created traces in critical processes throughout the client
to understand the performance of the code running there. We created a complete application that can report the state
of the client and the health of its internal mechanisms.

We also have published a [webinar with the OpenTelemetry project](https://www.cncf.io/webinars/observability-of-multi-party-computation-with-opentelemetry/)
showcasing how Hyperledger Fabric can rely on OpenTelemetry to report the state of execution and 
eliminate the black box feeling operators contend with.


# Testing
[testing]: #testing

In addition to the current integration testing scenarios, it will be required to test the system alongside an instance
of the OpenTelemetry collector collecting metrics and traces from the client applications.
The OpenTelemetry instance can report to stdout, a zipkin UI, and to a Prometheus server all this information
so testers can verify the correctness of the traces and metrics published.

# Dependencies
[dependencies]: #dependencies

None.

# Unresolved questions
[unresolved]: #unresolved-questions

