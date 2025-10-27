---
layout : default
title : hyperledger_fabric_flutter_labs
parent : RFCs
SPDX-License-Identifier : CC-BY-4.0
---

- Feature Name : support for hyperledger fabric version 2.4 or later on
flutter and dart platforms 
- Start Date : 2025-10-27 
- RFC PR :
https://github.com/LF-Decentralized-Trust-labs/LF-Decentralized-Trust-labs.github.io/pull/343
- Fabric Component : fabric-gateway 
- Fabric Issue :
https://github.com/LF-Decentralized-Trust-labs/LF-Decentralized-Trust-labs.github.io/issues/344

# Summary

bringing purely client side support on flutter and dart platforms ( e.g.
android ) for hyperledger fabric through hyperlefger fabric gateway
client api for java .

# Motivation

since the introduction of hyperledger fabric version 2.5 , earlier
releases of hyperledger fabric sdks deprecated .

some of the labs components are deprecated , too . this includes
archived fabric_client_flutter and fabric-server-node .

in reviving the outdated fabric_client_flutter lab in sight of the
hyperledger community's interests in the last five years and bringing up
the flutter and dart platforms up to date and separating from relaying
on the sdk servers , new platforms and components for android platform (
e.g. offline tls ca certificate loading on android instead of requesting
from a ca server over the internet ) will also be introduced for
conveniency , eliminating the certificate signing request requirements .

this is a conception proposal for the non-csr workflow of using the java
bindings of jnigen from hyperledger fabric gateway client api for java
on flutter and dart platforms to support offline signing and
certificates loading on mobile devices .

this will bring up mobile-only use cases for hyperledger fabric through
the hyperledger fabric gateway client api .

# Guide-level explanation

let ' s look at the workflow first :

workflow

- the client device creates a self-signed x.509 certificate for identity
.
- the client then generates a key pair and create a signing
implimentation based on the private key . 
- client identifies a uri for
the grpc connection and loads the tls ca certificate installed on the
device beforehand . 
- client connects to the gateway based on the
client identity . 
- client performs crud and smart contract operations
based on the identity .

this workflow differs from the old fabric_client_flutter lab with
offline tls ca certificate loading on device , high-level api calls for
the hyperledger fabric gateway client api , added interations with the
connected hyperledger fabric network via the identity , and eliminated
requirements for csrs and server-side sdks .

the new lab will only bring support for the android platform , since ios
or apple platforms require an apple developer account to install and
load a certificate from a device .

# Reference-level explanation

on flutter and dart platforms , android platform binding for dart
language is achieved through kotlin and java , and for external
third-party java api bindings is implemented via a dart package jnigen (
https://pub.dev/packages/jnigen ) . this will enable mobile platforms to
use client-only device to communicate with the hyperledger fabric
network through hyperledger fabric gateway client api for java ,
eliminating requirements for a sdk server or a third party acme server
to communicate to get into the network .

by referencing a pre-installed tls ca certificate and loading it on
device , tls connection of the certification part is guaranteed
attack-proof or mitm-proof , making the client-side application
completely offline on cryptographic operations . thus a hardware
security module and a pkcs # 11 interface aren ' t necessary for the
client-side .

# Drawbacks

backward incompatibility with hyperledger fabric version earlier than
2.4 .

the drawback could be eliminated by reusing the old
fabric_client_flutter lab for flutter and dart platforms .

# Rationale and alternatives

improving with the old fabric_client_flutter lab is possible . however ,
sdks of hyperledger fabric have diverged into two flows : v 2.4 or later
and versions ealier than v 2.5 . so it ' s rationale to create a new lab
and name it after version 2.4 for the hyperledger fabric sdks on
high-level network interations , while keeping the old lab compatible
with early sdks of hyperledger fabric or low-level interations with
hyperledger fabric network .

reusing the old lab for newly created sdk servers of hyperledger fabric
version 2.4 or later is also permissive , whatsoever , there is still a
requirement for creating a high-level workflow on the client side to
interate with the hyperledger fabric network . there has been seen
suggestions and improvement requests from the hyperledger community (
https://github.com/ghpZ54K8ZRwU62zGVSePPs97yAv9swuAY0mVDR4/fabric_client_flutter/issues
) .

# Prior art

fabric_client_flutter :
https://github.com/ghpZ54K8ZRwU62zGVSePPs97yAv9swuAY0mVDR4/fabric_client_flutter
.

fabric-server-node :
https://github.com/ghpZ54K8ZRwU62zGVSePPs97yAv9swuAY0mVDR4/fabric-server-node
.

# Testing

validate whether hyperledger fabric gateway client api for java supports
on android platform , as the early low-level java sdk does not implement
support for android .

complete client workflow on high-level defined apis from hyperledger
fabric gateway client api for java .

# Dependencies

the lab ' s objective for dependency is to replace the low-level offline
signing workflow described in fabn-895 with a high-level fabric-gateway
client api for offline signing .

fabric-gateway ( https://github.com/hyperledger/fabric-gateway ) .

fabn-895 (
https://docs.google.com/document/d/1gj5XB7yS-pfjpvZEUQh5lBGSIE6aQemu8A69tAYQtTc
) .

# Unresolved questions

- does fabric-gateway support for android platform ? 
- how does
pre-installing a tls ca certificate on device work , either through a
wired cable or over the internet ? 
- is loading from a signle tls ca
certificate sufficient ? how does choosing and loading one work if
multiple certificates are installed on device ? 
- does installing and
loading pre-existed tls ca certificates work in absent of an apple
developer account on ios ? 
- can the app be ported onto other platforms
( e.g. ios ) without pre-requisites conditions ( apple developer account
, etc. ) ? 
- will fabric-gateway happen to have more api definitions
for swift / objective-c / c programming languages on apple platforms ? 
