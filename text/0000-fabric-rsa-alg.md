---
layout: default
title: RSA Algorithm Support
nav_order: 3
---

- Feature Name: Re-introduce RSA crypto support
- Start Date: 2020-11-11
- RFC PR: 
- Fabric Component: bccsp, crypto
- Fabric Issue: 

# Summary
[summary]: #summary

Fabric had RSA crypto feature implemented since the very beginning even if there was a mismatch between the code and the documentation.
With the introduction of Fabric 2.x the RSA code was removed, so currently existing production ledgers with transactions signed by RSA no longer could be validated after upgrade to the new version.  Original RSA support commits:

* [35af475](https://github.com/hyperledger/fabric/commit/35af475) BCCSP support for RSA signing
* [2f7153f](https://github.com/hyperledger/fabric/commit/2f7153f) BCCSP ECDSA/RSA/X509 public/private key import

This RFC was created in response to this issue:

[FAB-18308](https://jira.hyperledger.org/browse/FAB-18308) Migrate an existing ledger from fabric 1.4.6 to 2.x if there are RSA certificates in any block of a channel to which the orderers belong to"


# Motivation
[motivation]: #motivation

The removal of RSA from the Fabric reverted the Modular Crypto Library implementation approach [FAB-354](https://jira.hyperledger.org/browse/FAB-354)

A configuration update to the channel to remove the RSA certificates is not a viable option since the Hyperleger Fabric components validate the full ledger data on boot and it will fail when validating existing blocks with RSA certificates, even if the latest one doesn't have any.
Therefore, there are 3 options for existing Hyperledger Fabric ledgers that have blocks with RSA certificates:  
1) don't upgrade production systems to Fabric 2.x
2) create a fork in order to have a working fabric with RSA validation routines
3) re-introduce the RSA routines back to the Fabric code

Neither the 1st or the 2nd options are an ideal solution, so the best and desired approach is clearly the 3rd.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Fabric uses asymmetric key cryptography and every transaction is signed by the user private key.
RSA (Rivest-Shamir-Adleman) was first standardized in 1994, and to date, is one of the most widely used algorithms for encryption and signing. 
It's based on a simple mathematical approach and it’s easy to implement in the public key infrastructure (PKI) and it’s an extremely well-studied and audited algorithm.

RSA keys were fully supported on Fabric 1.4.x (last LTS) and are working fine in production systems containing RSA signed blocks. 
If Fabric doesn't have this support, the production systems can't be upgraded to the 2.x versions and will be stuck on an older version without any support.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The fabric crypto API was designed in a modular way, so it could satisfy all the crypto needs of existing & future peer and membership services.
The BCCSP specs were already been published in the past: [BCCSP.pdf](https://jira.hyperledger.org/secure/attachment/10124/BCCSP.pdf)

# Drawbacks
[drawbacks]: #drawbacks

There are no real drawbacks of supporting this.

# Rationale and alternatives
[alternatives]: #alternatives

Not doing this will make businesses that are already using Fabric with RSA in production to start their own forks and have to maintain those themselves.

# Prior art
[prior-art]: #prior-art

RSA is the most used, studied and tested algorithm proven to be secure. References to the original implementation are in the summary above.

# Testing
[testing]: #testing

RSA testing has been made in the past:
* [b17e846](https://github.com/hyperledger/fabric/commit/b17e846) [FAB-3441](https://jira.hyperledger.org/browse/FAB-3441) bccsp/sw ECDSA/RSA sign test coverage
* [7aa43d5](https://github.com/hyperledger/fabric/commit/7aa43d5) [FAB-3441](https://jira.hyperledger.org/browse/FAB-3441) bccsp/sw ECDSA/RSA verify test coverage

# Dependencies
[dependencies]: #dependencies

No dependencies.

# Unresolved questions
[unresolved]: #unresolved-questions

There is a question if RSA support should also be introduced for HSM module: https://github.com/hyperledger/fabric/pull/2069#issuecomment-720782059
This can be a related feature.
