---
layout: default
title: RFC Template
nav_order: 3
---

- Feature Name: modular_crypto_service
- Start Date: 2020-09-19
- RFC PR: (leave this empty)
- Fabric Component: core, fabric-sdk-*, fabric-ca
- Fabric Issue: (leave this empty)

# Summary
[summary]: #summary

One paragraph explanation of the feature.

This RFC aims to provide broader crypto service configurability or pluggable capability. 
It mainly changes design of crypt/X509 usage and hash algorithm opt-in in several layer.  

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected
outcome?

Following is the original initiative post, translated from Chinese.
```
The Hyperledger Fabric code project has a very wide range of adoptions in many domestic industries and institutions, and has a very basic role in the application of the blockchain industry.
According to Chinese policy requirements, it is necessary to support the national secret (GM) encryption algorithm. Many domestic enterprises and Fabric enthusiasts have successively carried out some corresponding national secret (GM) fabric forks with outstanding achievements. However, in this matter, there is serious duplication of working hours and waste of human resources.
Here, the Hyperledger Technical Working Group China (TWGC) proposes that the majority of Fabric enthusiasts and users gather and work together to create a standard national cryptographic project of Hyperledger Fabric, which is shared by everyone, and finally meet the needs of community maintainer request, merge into the Fabric up-stream.
```
We now have 3 streams of China crypto libraries under Hyperledger-TWGC github organization. 
Some of them has also provided successful ane enterprise-proven bccsp implementations. 
But as along as we go deeper into fabric source, we notice implementing bccsp only could not fulfill all of Chinese national crypto specification requirements.
There are some cypro library usages outside of bccsp, such as communication protocol, X509 format conversion. 
It relates to fabric architect design thus we need broader consensus and feedbacks from community as an RFC level change. 


Additionally, the new design should take care of not only Chinese but for any other national crypto standards. 
 
# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There are 3 configurations controlling hash algorithm used in fabric, separately.
- Fabric uses hash function in bccsp related to peer/orderer (controlling feature samples: signing)
- Fabric uses hash function based on `crypto_config` section in organization/msp level (controlling feature samples: private data)
- Fabric uses hash algorithm based on `HashingAlgorithm` value in channel level (controlling feature samples: block hash)

Beside bccsp, a new X509 interface is introduced. Rework of usage of `crypto/X509` includes making `crypto/X509` as an implementaion of new X509 interface.    

This change should not impact Fabric usages if they are on default configuration. But it may impact on Fabric fork developers since we exposed more extensibility.

To fulfill national crypto standards, security of communication is a must. 
Fabric family replies on classical https and gRPCs communication. So any implementation should firstly have another fork of crypto-complaint https/gRPCs.
After these forks become mature, incorporate them into individual national crypto fork of fabric. 
TWGC could provide a sample guideline of incorporation for Fabric forks, but not Fabric main-stream.

[**WIP**] feature examples, migration guideline

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient
detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and
explain more fully how the detailed proposal makes those examples work.

[**WIP**] along with previous feature examples

crypto/X509 ==> [X509 interface](https://github.com/Hyperledger-TWGC/fabric-gm-plugins/blob/078c5bac196c2c89190b48b9fa05102800a56c34/interfaces.go#L13) 
crypto/sha256 => hash.Hash interface
bccsp implemtation copy to bccsp folder before rebuild fabric



# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this? Are there any risks that must be considered along with
this rfc. 

**AS IS**. No matter how we improve fabric pluggable flexibility, developer still have to maintain a Fabric fork with all plugin equipped.
Does this RFC really help a lot when comparing fashions: 
1. let developer change every hardcode cryptoSuite into another national crypto hardcode cryptoSuite.
1. let developer implement a plugin under the restriction of interface, and carefully port into fabric source.
  
**Fabric Family interopt**   
For Fabric surrounding projects such as SDKs and other hyperledger projects with Fabric support, for a specific nation crypto standard Fabric fork. 
developer community still have to make another SDK fork in order to satisfy secured communication. A lot of cross validation is there.   

**System incompatible**
For a running fabric network (mainstream) using current default configuration value, according to our design, there is no upgrade operation needed.  
But some corner cases and risks are:
- if channel config `HashingAlgorithm` is changed to an unrecognizable value, there will be unpredictable behavior
- if channel config `HashingAlgorithm` is changed in the middle, there will be unpredictable behavior
  

# Rationale and alternatives
[alternatives]: #alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this?

same as Motivation section

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For consensus, global state, transaction processors, and smart contracts
  implementation proposals: Does this feature exists in other distributed
  ledgers and what experience have their communities had?
- For community proposals: Is this done by some other community and what were
  their experiences with it?
- For other teams: What lessons can we learn from what other communities have
  done here?
- Papers: Are there any published papers or great posts that discuss this? If
  you have some relevant papers to refer to, this can serve as a more detailed
  theoretical background.

This section is intended to encourage you as an author to think about the
lessons from other distributed ledgers, provide readers of your RFC with
a fuller picture.  If there is no prior art, that is fine - your ideas are
interesting to us whether they are brand new or if it is an adaptation.

Note that while precedent set by other distributed ledgers is some motivation,
it does not on its own motivate an RFC.  Please also take into consideration
that Fabric sometimes intentionally diverges from common distributed
ledger/blockchain features.

[**WIP**] We take several references from FISCO/BCOS for multiple language implementation of Chinese national standard.
We also take reference from famous Crypto library BouncyCastle which has almost native support for Chinese national EC.
BouncyCastle also provide a reference in X509 interface design.    


# Testing
[testing]: #testing

- Test plan automates importing this implementation to fabric and run regression fabric test



# Dependencies
[dependencies]: #dependencies

https://jira.hyperledger.org/browse/FAB-5496


# Unresolved questions
[unresolved]: #unresolved-questions

Current clumsy but fast solution is that we copy a bccsp into fabric source and rebuild. 
If fabric community can agree on a framework supporting dependency-inject in golang (aside from go plugin), it could be smarter.

