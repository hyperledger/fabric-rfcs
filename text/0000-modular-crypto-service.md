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
There are some crypto library usages outside of bccsp, such as communication protocol, X509 format conversion. 
It relates to fabric architect design thus we need broader consensus and feedbacks from community as an RFC level change. 

Additionally, the new design should take care of not only Chinese but for any other national crypto standards. So discussion below will focus on two parts.
- Adjust with new crypto standards to fabric in bccsp.
- Adjust with new crypto standards at community level.
 
# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

There are 3 configurations controlling hash algorithm used in fabric, separately.
- Fabric uses hash function in bccsp related to peer/orderer (controlling feature samples: signing)
- Fabric uses hash function based on `crypto_config` section in organization/msp level (controlling feature samples: private data)
- Fabric uses hash algorithm based on `HashingAlgorithm` value in channel level (controlling feature samples: block hash)

To apply new crypto standards at bccsp:
- As golang language, we have to rebuild the library as there no hot module adding or replacement way for golang.
- New crypto standards implemented following bccsp is needed. So we need to adjust parts of bccsp for ex
when a new X509 interface is introduced as much with national crypto standard or any new crypto standard. Rework of usage of `crypto/X509` includes making `crypto/X509` as an implementaion of new X509 interface.
- The changes for the new crypto standards should be less impacts for fabric and in a configurable way if possible.

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
## Detail proposal for BCCSP
```
- sw  -> generalBSSCP
```
Overall speaking, we wanna to make BCCSP becomes a general workflow as blockchain crypto provider.
Which means, we are going to refactor current sw workflow with serval kind of design patterns.

Adapter pattern:
We can imaging that we have an abstract proto type of interface there invoked by bccsp interface(as key.go) and plays role as adapter with specific crypto library.(ECDSA, PKCS11, SM2, etc...)

Prototype pattern:
We can use ECDSA, and PKCS11 as prototype as default for software crypto and hardware crypto samples, which used in CI, release etc...

Factory pattern:
For example, to adapt with any new crypto library
- Someone need the crypto provider to implement the type(following prototype) and tested by themself.
- Once the adaptor been implemented, everyone hope it can be easily integrated with fabric.
1. <del>Someone fork fabric and code change etc... all doing by themself in specific forked branch.</del>
1. Someone fork fabric and copy pasted adaptor impl as part of bccsp package, then recomplie fabric. (so far sample shows this way)
1. Someone use fabric and compile adaptor, start fabric with adaptor binary as go plugin.

Command Pattern:
- Ref to above possbile ways of implemention, in either of solution, there need any paramter to decide with crypto library should be used.

Take publicKey To EncryptedPEM as sample:

#### Here is logic need to be added as a global wrapper to load plugin
currently, some part of logic is implemented here. https://github.com/Hyperledger-TWGC/fabric/tree/bccsp-gm
```go
	// to do package gm
	swbccsp.AddWrapper(reflect.TypeOf(&sm2.SM2PublicKey{}), &sm2.SM2PublicKeyKeyVerifier{})
	......
	func InitCryptoProviders() {
		lock.Lock()
		defer lock.Unlock()
		// init
		// publicKeyToEncryptedPEM
		puk2epem = make(map[reflect.Type]func(k interface{}, pwd []byte) ([]byte, error))

		//default AddWrapper for ecdsa
		puk2epem[reflect.TypeOf(&ecdsa.PublicKey{})] = ECDSApublicKeyToEncryptedPEM

		//AddWrapper for sm2
		puk2epem[reflect.TypeOf(&ccssm2.PublicKey{})] = sm2.SM2publicKeyToEncryptedPEM
	}
```
Full changes here: https://github.com/SamYuan1990/fabric/blob/packageRefactor/bccsp/sw/new.go#L107-L217

#### Here is the logic changed in current sw to fill with wrapper above. 
```go
package sw

publicKeyToEncryptedPEM(k interface{}, pwd []byte){
	for i,v := range GetfromWrapperMap() {
		if k.type == i.type {
			return do(k,v)
		}
	}
	return "not supported"
}
```
Full changes here: https://github.com/SamYuan1990/fabric/blob/packageRefactor/bccsp/sw/keys.go#L315-L322

#### A sample function as specific crypto wrapper/provider
```go
package specific
import specific_crypto_lib

//example function
publicKeyToEncryptedPEM(k interface{}, pwd []byte){
    specific impl
}
```
Full changes here: https://github.com/Hyperledger-TWGC/fabric/tree/bccsp-gm/bccsp/sm2

Option One:
With help for go plugin, we are able to do:
- Impl an interface as struct in plugin code.
- Load the type of the interface by reflect.
- Invoking interfaces method which provided by plugin.
- From raw type to create plugin struct.
For details, please refer https://github.com/SamYuan1990/go-plugindemo/blob/poc/main.go#L84-L110

```go
AddWapperMap(path string){
p, err := plugin.Open(path.so)
	if err != nil {

	panic(err)

	}
	
	initFunc, err := p.Lookup("init")
  obj := initFunc().()

	f, err := p.Lookup("publicKeyToEncryptedPEM")
	...

	my_type=reflect.Typeof(&obj)
	//default for all type replacement
	GlobalCryptoWrapper[my_type] = f
}
```

Then we can make different kind of plugin.so file with interface defined in BCCSP and make the crypto implmentation plugable by switching the file.

Option Two:
We don't use the go plugin, as the limitation for ex: https://github.com/golang/go/issues/31354
We put the specific crypto package as a part of bccsp package and rebuild fabric with the specific crypto package.(as build tag)
Full changes here: https://github.com/Hyperledger-TWGC/fabric/tree/bccsp-gm

Option Three:
Pre discussed on Fabric contributors meeting, there seems a 3rd option here for discussion.
https://github.com/hyperledger/fabric/blob/main/msp/msp.go#L200
https://github.com/hyperledger/fabric/blob/main/msp/identities.go#L170
https://github.com/hyperledger/fabric/blob/main/msp/identities.go#L255
With technical backgroud, as any identity will be a part of MSP in hyperledger fabric. And if we look at MSP interface, we can see that MSP interface as an unimplement type as others and current MSP implementation depends on BCCSP. 
This dependency also supports option one and two, when we try to add a new crypto implementation for hyperledger fabric. 
Which means, the question is equal with modular MSP service?
```golang
| MSP   |
-------
| BCCSP |
-------
| Crypto curves |
```
In further details, with relationship between MSP and BCCSP.
for reference, https://github.com/hyperledger/fabric/blob/main/msp/identities.go#L55-L85
The MSP interface depends on BCCSP interface as implementation.
https://github.com/hyperledger/fabric/blob/main/msp/mspimpl.go#L42-L106
also some fix use between MSP package, BCCSP package, and x509 package.
Hence, with this rfc, we hope to refine interface definition and responsibility.
```golang
| MSP  (response for upper case usage, as get MSP ID etc...) |
-------
| BCCSP (plays role as adapter between fabric msp concepts and crypto curves and implementations) |
-------
| Crypto curves s as ecdsa/ed25519/others |
```
- [x] Redefine MSP responsibilities, as it relays on some crypto provide ex BCCSP interface implementation and response for fabric data model on business and logic level.
- [x] Redefine BCCSP responsibilities, as it relays on crypto library implementation, plays role as adapter interface between MSP and cyrpto library implementations response for fabric data model on implementation level.

With current investigation, we need to remove `x509` dependency from `msp` package which means, we'd better add a new type of key as cert_key extend key interface in `bccsp` to make `msp` decouple with `x509` interface.

Remove `x509` dependency from `msp` package steps: 
1. add a cert interface in bccsp
2. impl import cert function in bccsp for msp creation
3. impl cert related function one by one from msp to bccsp

a sample commit for new cert structure in bcssp can be found at https://github.com/Hyperledger-TWGC/fabric/pull/19/commits/6b7d0955fa1fcea5f59f6502249ffd7d06c005b8 the second sample commit for decouple msp interface with x509 can be found at https://github.com/Hyperledger-TWGC/fabric/pull/19/commits/78afd370ecc1292a1d16b24be60d84194db7c236

the benefits for this kind of refactor:
1. When a new crypto curve is considered to be supported in Hyperledger Fabric, we can just make a BCCSP implementation and by addition an option in MSP level to make it. For ex: ed25519
1. For specific crypto curves, we can also support it by dynamic type, as a binary plugin of BCCSP and a MSP option to enable the binary plugin. For ex: Chinese national crypto curves.
1. When MSP level interface is considered to add new logic, we don't need to worry it affects exisiting BCCSP logic and how the changes integrate with crypto related packages either bccsp or x509. 

### Refactor current bccsp with new process logic.
To make something as classloader in bccsp. There may need to build some global level map in `<type, function>` way. For example with below logic, we can reused `fileks.go` and by reflect, the code will auto switch between crypto logics which added.

### Implementing APIs basing on new crypto standards
Here are the list below for the APIs should be implemented.

## Detail proposal for sdk changes

On fabric sdk side, we need align with the changes, so that from client to network able to use same crypto alg/crypto cruve or tls cert...

### Detail Proposal for java sdk changes.
For java SDK, we need refactor `org.hyperledger.fabric.sdk.security` package.
1. supporting dynamic class loading with a modular crypto service jar package as implementation of `org.hyperledger.fabric.sdk.security`.
1. the implation should implate for both msp/identity load from cert file.
1. the implation should implate for tls connection.

#### detail changes for `org.hyperledger.fabric.sdk.security`
step 1: As `org.hyperledger.fabric.sdk.security.certgen` also a factory pattern on design which builder class to create keypair class for tls. we are able to refactor those two class into a factory interface and a impl interface.
step 2: as all classes in `org.hyperledger.fabric.sdk.security` in factory pattern, a refactor can be made to make this package following factory pattern.
step 3: the factory of `org.hyperledger.fabric.sdk.security` should support dynamic class load from an external jar file if provided. or use current implementation by default.

#### if someone is going to implate a modular crypto service for java sdk
step 1: for msp loading, impls interface defined in https://github.com/hyperledger/fabric-sdk-java/blob/master/src/main/java/org/hyperledger/fabric/sdk/security/CryptoSuite.java
step 2: for tls config, impls new interface defined in step 1 above as refactor result for package `org.hyperledger.fabric.sdk.security.certgen`
step 3: for class loader, impls factory class defined as steps 3 above as refacotr result for package `org.hyperledger.fabric.sdk.security`

#### Drawbacks & A sample solution/workaround
For java sdk, there some low level code outside fabric java sdk scope but as dependencies, for ex `io.netty`.
A general solution for this is treating those jar dependencies as interfaces. 
Modular crypto sevice provider should impl a changes for those jar dependencies by themselves.

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

