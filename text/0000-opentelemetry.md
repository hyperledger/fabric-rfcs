---
layout: default
title: OpenTelemetry Integration
nav_order: 3
---

- Feature Name: OpenTelemetry Integration
- Start Date: 2021-01-12
- RFC PR: (leave this empty)
- Fabric Component: core, chaincode, sdks
- Fabric Issue: (leave this empty)

# Summary
[summary]: #summary

This request for comments proposes integrating Hyperledger Fabric, its SDKs, core and chaincode components 
with the OpenTelemetry project. It advocates for relying on OpenTelemetry to record and send metrics.
It introduces the concept of tracing, reporting the execution of chaincode, core components and SDKs to help correlate activities with the chain.

# Motivation
[motivation]: #motivation

The OpenTelemetry project is a CNCF project that aims to standardize observability, especially in a cloud-native environment.
OpenTelemetry is working to create a standard around representing metrics, traces and logs.
We aim to bring full observability of Hyperledger Fabric to allow tracing of transactions all the way from the client
to each of the chaincode invocations. We also want to capture chaincode interactions.
We want to be able to trace and correlate actions from chaincode events.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In this guide, we will cover the notion of observability. Observability comes from the DevOps world, and is defined in
[Wikipedia](https://en.wikipedia.org/wiki/Observability) as "a measure of how well internal states of a system 
can be inferred from knowledge of its external outputs".
Software that is observable exposes its behaviors by sending traces, which represent the path of execution of a particular request.
Each trace can be represented as a set of spans that are correlated as child/parent or sequential relationships.
Observable software is exposing metrics, representing the internal state of the components, as well as logs, emitted from the execution of the software.

In Hyperledger Fabric, we rely on the OpenTelemetry framework to report traces, metrics and logs, and correlate them across services.

In practice, this means a request made by a client connected via the Fabric SDK to a node can pass on the context of a trace. 
When chaincode executes, it can correlate to that trace, or create its own, and report to a collector storing this information.
Chaincode can also expose traces in chaincode events. Any code triggered from such an event can then use that trace to report its origin.

Blockchain operators can reconstitute a graph of the interaction of all the components at play to create a service map.
This helps uncover trends and issues of performance, as well as shortening the time it takes to investigate problems.
It also allows to quickly iterate when upgrading chaincode and witnessing the effects it has on the system.
It offers some security capabilities, such as detecting unexpected executions, or react quickly to metric changes.

Hyperledger Fabric developers can take advantage of those techniques with no code changes to their existing chaincode deployments.
Each chaincode execution generates a top-level trace and will report to an endpoint provided by the environment.
They may desire to build additional spans to capture additional data, or report errors. The spans they create will 
correlate automatically to the parent span.

Developers may also create a trace before calling out to Fabric in their client code.
The current trace information will be sent along with the message to the peer.

Finally, developers may like to report metrics as part of chaincode execution. They can use OpenTelemetry counters,
gauges and other metrics constructs to report metric data.
The metric endpoint is configured when the chaincode is deployed. 
In the case of external chaincode, the developers can set standard [OpenTelemetry variables](https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/sdk-environment-variables.md)
to point at an OpenTelemetry collector instance.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Message headers

The trace information is passed in as additional data to the message, using a new field containing headers.
Trace headers are filled in by the SDK and read by the chaincode execution framework.

## Changes to the chaincode shims

Each shim implementation should be modified to explicitly handle the additional message field to capture the trace ID passed in.
If no parent trace information is present, the shim continues and creates a new trace with no parent relationship.

As a demonstration, in this [PR](https://github.com/hyperledger/fabric-chaincode-java/pull/153/files#diff-db5cb40a9966f929cdf9963d606b8961248f94aac09ed150491fdc41e73d82b9R94), the Java shim implementation `call()` method which calls the chaincode execution changes to embed a try/finally statement
that creates and ends the span once execution is finished. It also captures exceptions thrown during execution to set the status of the trace as an error.

The Java chaincode shim also changes to implement the MetricsProvider interface to [provide metrics through OpenTelemetry](https://github.com/hyperledger/fabric-chaincode-java/pull/153/files#diff-d97166bc281ed3eec5bb83ccd164698c2ee3ae1bd5e8e4ab6a8ea1d86068e951R24).

The same set of changes must be applied to the Go and Javascript shim implementations.

## Changes to client SDKs

Client SDKs automatically create a trace to capture the call to the chain.

The trace is created with the kind Span.Client.

If a current trace exists, the new client trace adds it as its parent.

The trace and span ID must be sent to the chain as a message header.

The SDK should use the standard environment variable environment to let users define how and if they want to report
trace data to an endpoint of their choosing.

# Drawbacks
[drawbacks]: #drawbacks

OpenTelemetry has not reached 1.0. It has unequal maturation between metrics, traces and logs, and across languages.
However, traces and metrics are well ahead of logs, and Java, Go and Javascript are well supported.

The changes require a change in the structure of Protobuf messages, which may require a major upgrade of Fabric.

The OpenTelemetry reporting system happens securely over Protobuf, with chaincode containers and client applications sending data.
This requires that an OpenTelemetry-compatible endpoint is present to receive the data.

# Rationale and alternatives
[alternatives]: #alternatives

This design allows full observability of Hyperledger Fabric, to a degree of detail that will help developers understand
the impact of the chaincode and organize operations using the latest framework. This allows Hyperledger Fabric
to report meaningful data in a similar way that cloud-native applications are built right now.

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
showcasing how Hyperledger Fabric can rely on OpenTelemetry to report the state of execution of chaincode and 
eliminate the black box feeling operators contend with.


# Testing
[testing]: #testing

In addition to the current integration testing scenarios, it will be required to test the system alongside an instance
of the OpenTelemetry collector collecting metrics and traces from the client applications and the chaincode executors.
The OpenTelemetry instance can report to stdout, a zipkin UI, and to a Prometheus server all this information
so testers can verify the correctness of the traces and metrics published.

# Dependencies
[dependencies]: #dependencies

None.

# Unresolved questions
[unresolved]: #unresolved-questions

## Trace ID header format
We need to pick a standard form for the message header that will represent the caller trace ID.
Right now, the Java shim example uses a B3 header.
We can choose to support any header, and try all known header extraction mechanisms.
We probably should change the header format to the [W3C trace id standard](https://www.w3.org/TR/trace-context/).
