---
layout: default
title: Create a standalone config transaction library
parent: RFCs
---

- Feature Name: Create a standalone config transaction library that generates channel config creation and update envelopes
- Start Date: 2020-03-20
- RFC PR: (leave this empty)
- Fabric Component: fabric, fabric-config
- Fabric Issue: (leave this empty)

# Summary
[summary]: #summary

This feature will implement a consumable standalone library that supports generating configuration envelopes for
operations including application and system channel creation, channel configuration updates, and attaching
endorsing signatures to generated envelopes.

# Motivation
[motivation]: #motivation

The motivation for this work is to provide a consistent API that is actually intended for consumption by external
clients for any operations involving channel config transactions. One example usage case could be a
CLI tool for creating config updates that takes as flags properties to be updated alongside an existing config transaction.
This CLI tool could then pass these parameters down to the configtx package for generating a config update envelope with the computed
read and write sets. Currently `configtxgen` is available and frequently misused as a tool for these operations, but it was never intended for
production usage. Much of the existing process for generating configuration updates is both tedious and error prone involving
decoding a config transaction to JSON with the protolator tool and manually modifying fields prior to encoding back to a protobuf
and submitting for update.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Work on this consumable library has already begun in [`fabric/pkg/configtx`](https://github.com/hyperledger/fabric/tree/master/pkg/configtx)
and tracked in [FAB-17628](https://jira.hyperledger.org/browse/FAB-17628). Example usage has been included in the
[godocs](https://godoc.org/github.com/hyperledger/fabric/pkg/configtx#pkg-examples) for this package and can be referenced for external tooling.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

API documentation already exists in the godocs for this package and can be referenced [here](https://godoc.org/github.com/hyperledger/fabric/pkg/configtx).

# Drawbacks
[drawbacks]: #drawbacks

Many potential consumers may already have a stable method for updating configuration transactions and may be
unwilling to make amendments to their processes by consuming this library. It could be argued that even if this library
exists, no external clients already using `configtxgen` would want to switch to consuming it. Another consideration is this library
will only be provided in Golang, but may eventually support TinyGo compilation to WebAssembly for usage as a web module.

# Rationale and alternatives
[alternatives]: #alternatives

Eventually it will become too onerous to maintain current workarounds for channel configuration management
as new features are added around channel config. Rather than having to manually update bits and pieces of workaround
code, consumption of this library will become a consistent method for adopting new features.

# Prior art
[prior-art]: #prior-art

Consumable Golang packages have consistent and effective APIs, are well-documented, leverage universal coding standards,
and provide useful examples of common usage.

# Testing
[testing]: #testing

This package has high coverage of core features on both a unit level and integration test level. Example usage tests are also
included in the godocs.

# Dependencies
[dependencies]: #dependencies

In an effort to make this library as lightweight as possible, we've reduced the number of required dependencies to just a handful including
the Golang standard library and the following imported `hyperledger` packages and components:
- `fabric-protos-go`
- `fabric/common/policydsl`
- `fabric/common/tools/protolator`

# Unresolved questions
[unresolved]: #unresolved-questions

As part of this RFC process we would like to also move the configtx package code we've already started on in `hyperledger/fabric/pkg/configtx` to
a separate `hyperledger/fabric-config` repo to remove it from fabric's release cycle. As a standalone consumable library,
this package does not need to exist in `hyperledger/fabric` or be tied to a specific release.
