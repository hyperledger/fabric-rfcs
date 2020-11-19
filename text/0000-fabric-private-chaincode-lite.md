---
layout: default
title: Fabric Private Chaincode - Lite
nav_order: 3
---

- Feature Name: Fabric Private Chaincode (FPC) - Lite
- Start Date: 2020-01-28)
- RFC PR: (leave this empty)
- Fabric Component: Core (fill me in with underlined fabric component, core, orderer/consensus and etc.)
- Fabric Issue: (leave this empty)


# Summary
[summary]: #summary

This RFC introduces a new framework called Fabric Private Chaincode (FPC), built on Hyperledger Fabric.
FPC enhances data confidentiality for chaincodes by executing them in a Trusted Execution Environment (TEE), such as Intel&reg; SGX.
Most importantly, FPC protects transactional data while in use by the chaincode, in transit to/from a client, and stored on the ledger.
Hence, differently from typical chaincode applications, curious Fabric peers can only handle encrypted data related to FPC chaincodes.
Ultimately, this mitigates the requirement for endorsing peers to be fully trusted for confidentiality.

The design of FPC thus extends the existing model of privacy in Fabric, enabling the secure implementation of additional use cases.
Some examples include private voting systems, high-stakes sealed-bid auctions, and confidential analytics.
FPC is available open-source on Github (https://github.com/hyperledger-labs/fabric-private-chaincode) as patch-free runtime extension of Hyperledger Fabric v2.2.

**FPC in a nutshell.**
FPC operates by allowing a chaincode to process transaction arguments and state without exposing the contents to anybody, including the endorsing peers.
Also, the framework provides interested parties (clients and peers) with the capability to establish trust in an FPC chaincode.
This is achieved by means of a hardware-based remote attestation, which parties use to verify that a genuine TEE protects the intended chaincode and its data.
Clients can thus establish a secure channel directly with the FPC chaincode (as opposed to the peer hosting the chaincode) which preserves the confidentiality of transaction arguments and responses.
On the hosting peer, the TEE preserves the confidentiality of the data while the chaincode processes it.
Such data includes secret cryptographic keys, which the chaincode uses to secure any data that it stores on the public ledger.


# Motivation
[motivation]: #motivation
<!-- 
    Description of section: General motivation on why Fabric does not cover use-cases having strong privacy requirement and that TEE/FPC can close that gap.
    Note: At this level there is no distinction between full FPC and FPC Lite and hence use the term FPC through-out without qualification 
    and also still mention all possible use-cases. 
    The distinction is done later in the architecture/design section
-->
## Enable new use-cases with strong privacy requirements
FPC is motivated by the many use cases in which it is desirable to embody an application in a Blockchain architecture, but where in addition to the *existing integrity assurances*, the application *also requires privacy*. This may include private voting, sealed bid auctions, operations on sensitive data such as regulated medical or genomic data, and supply chain operations requiring contract secrecy. With Fabric's current privacy mechanisms, these use cases are not possible as they still require the endorsement nodes to be fully trusted.
For example, the concept of channels and Private Data allows to restrict chaincode data sharing only within a group of authorized participants, still when the chaincode processes the data it is exposed to the endorsing peer in clear. In the example of a voting system, where a government may run an endorsing peer it is clear that this is not ideal.

## Enable performance improvements
<!-- Below section does not apply to FPC Lite but as we haven't introduced that and this motivation can be address in a later extensions we still mention it here -->
A second motivation for FPC is its integrity model based on hardware-enabled remote cryptographic attestation; this model can provide similarly high confidence of integrity to the standard Fabric model of integrity through redundancy, but using less computation and communication. With TEE-based endorsement and remote attestation, a new set of endorsement policies are made possible, which can reduce the number of required endorsements and still provide sufficient assurance of integrity for many workloads.

FPC adds another line of defense around a chaincode. Over time and with continued development of support for other languages and Trusted Execution Environments, we intend FPC to become the standard way to execute many or even most chaincodes in Fabric, similar to what HTTPS has become for the Web.


# User Experience
[functional-view]: #functional-view
<!-- 
    Description of section: This section describes the functional view as experienced by developers of chaincode and applications interacting with it as well as admins deploying them.
    Note: At this level there is no distinction between full FPC and FPC Lite and hence use the term FPC through-out without qualification. 
    The distinction is done later in the architecture/design section
-->

## Overview

Fabric Private Chaincode is best thought of as a way of running smart contract chaincode inside a Trusted Execution Environment (TEE), also called an _enclave_, for strong assurance of privacy and a more computation-efficient model of integrity.
Programs executed in the TEE are held in encrypted memory even while running, and this memory can't be read in the clear even by a root user or privileged system process. Chaincode state is encrypted with keys only known to the chaincode. Each chaincode runs in its own TEE to provide the best possible isolation among chaincodes on an endorsing peer. Because the execution is opaque, a modified set of integrity controls is also implemented in order to be certain that what's running is exactly what is intended, without tampering.
<!-- commented out as not provided by FPC Lite and anyway not crucial for this section:
With FPC’s hardware-rooted cryptographic integrity mechanisms, it requires less redundancy of computation than in the standard model of Fabric to get a high level of trust.
-->
With FPC, a chaincode can process sensitive data, such as cryptographic material, health-care data, financial data, and personal data; without revealing it to the endorsing peer on which it runs.

Overall, FPC can be considered an addition to Fabric wherein all chaincode computation relies only on the correctness of data provided by an authenticated Client or computed inside of and signed by a validated enclave.
The Endorsing Peer outside of the enclave is considered untrusted.
For this reason, all transaction data and chaincode state are encrypted by default in a way that only the FPC chaincode can access them in clear.

Writing chaincode for FPC should come natural to developers familiar with Fabric as the programming model (e.g., chaincode lifecycle, chaincode invocations and state) is the same as for normal Fabric chaincode.
The main differences are a (for now at least) different programming language (C++) and a Shim API which implements a subset of the current Fabric API.
The Shim is responsible to provide access to the ledger state as maintained by the `untrusted` peer. In particular, the FPC Shim, under the cover and transparent to the developer, encrypts all state data that is written to the ledger and decrypts them when retrieved later. Similarly, it also encrypts and authenticates all interaction with the applications, see below. Lastly, it attests to the result and the state update based on the enclave's hardware identity to provide a hardware-trust rooted endorsement.

Applications can interaction with a FPC chaincode using an extension of the Fabric Client Go SDK.
This FPC extension exposes the Fabric `gateway` interface and transparently encrypts and authenticates all interactions with a FPC chaincode.

Note that FPC hides all interactions with the TEE technology from the developers, i.e., they they do not have to understand the peculiarities of TEEs.  This largely also applies to the adminstrators deploying FPC chaincode, although they will have to understand the general concepts of TEE to make informed decisions on security policies and to configure the attestation credentials.

To illustrate the interaction between an application and a FPC chaincode see the following figure. In particular, this figure highlights the encrypted elements of the FPC architecture.

![Encryption](../images/fpc/high-level/Slide1.png)

Encrypted elements of the FPC architecture (over and above those in Fabric, such as TLS tunnels from Client to Peer) include:

- The arguments of a transaction proposal.
- The results of execution in a proposal response message returned to the Client.
- All contents of memory in both the Chaincode Enclave(s).
- All entries of the transaction writeset (by default).

Note that with the exception of the results, where also a legitimate requestor knows the secret keys, all secret/private keys are known only by to the enclaves or, for memory encryption, to the HW memory encryption engine (with the hardware also enforcing that only legitimate enclaves have access to the unencrypted memory).


## FPC Development

### Chaincode
<!-- this section should cover: platform (x86, sgx sdk, linux; but also via docker); cmake; shim.h -> hello world tutorial -->

As mentioned earlier, FPC chaincode is executed in an enclave.
In the initial version, FPC will supports Intel SGX as trusted execution technology.
For this reason, a FPC chaincode must currently be written in C++ using our FPC SDK, which builds on top of the Intel [SGX SDK](https://github.com/intel/linux-sgx).
The current development platform is Linux (Ubuntu). However, we also do enable seamless development via docker. 
Hence, development is also easily possible with MacOS or Windows as host.

To ease the development process, FPC provides a `cmake` based build system which allows the developer to focus on the chaincode without having to understand SGX build details.
The programming interface against which a FPC chaincode has to be programmed is encapsulated in the C header file [`shim.h`](https://github.com/hyperledger-labs/fabric-private-chaincode/blob/master/ecc_enclave/enclave/shim.h).
To get a more in-depth understanding of the chaincode development, consult the detailed [`HelloWorld` Tutorial](https://github.com/hyperledger-labs/fabric-private-chaincode/tree/master/examples) that guides new FPC developers through the process of writing their first FPC chaincode.

The outcome of the build process are two deployment artifacts: 
(1) a file `enclave.signed.so`, the enclave binary containing both the chaincode as well as the (trusted part) of the FPC shim, and 
(2) the SGX (code) identity of the chaincode, `MRENCLAVE`, which can be imagined as a form of cryptographic hash over `enclave.signed.so` and related SGX deployment metadata.
<!-- mrenclave described as simple hash is a gross over simplification but should be ok here to give some intuition without getting into the details.-->

Note: It is the goal for the project is to support additional languages in the future, e.g., there is an ongoing effort to add support for WebAssembly.

### Application

FPC extends the Fabric Client SDKs with extra functionality that allows users to write end-to-end secure FPC-based applications.
In particular, the FPC Client SDK extension provides these core functions:
First, FPC transaction proposal creation, including transparent encryption of arguments.
Second, FPC transaction proposal response validation and decryption of the result.
The encryption and decryption is performed by the Client SDK "under the covers" without requiring any special action by the users, i.e.,
the users still use normal `invoke`/`query` functions to issue FPC transaction invocations.
Last, the Client SDK takes care of enclave discovery, that is, the Client SDK is responsible to fetch the corresponding chaincode encryption key and to determine the endorsing peers that host the FPC chaincode enclave.
Extended support for other Fabric Client SDK, such as the  NodeSDK, will be future work.

An application can interact with the asset store chaincode from our [`HelloWorld` Tutorial](https://github.com/hyperledger-labs/fabric-private-chaincode/tree/master/examples) using the FPC extension based on the gateway API of the Fabric Client Go SDK. Here an example ***app.go***:
```go
// Get FPC Contract
contract := fpc.GetContract(network, "hellloWorld")

result, err = contract.SubmitTransaction("storeAsset", "myDiamond", "100000")
if err != nil {
    log.Fatalf("Failed to Submit transaction: %v", err)
}
```

The FPC Gateway API is documented [here](https://github.com/hyperledger-labs/fabric-private-chaincode/blob/master/client_sdk/go/fpc/contract.go) in detail.

## FPC Chaincode Deployment

### Peer Setup 
There are two preparation steps required before one can deploy FPC Chaincode:

- The chaincode administrator has to register with the [*Intel Attestation Service (IAS)*](https://software.intel.com/content/www/us/en/develop/topics/software-guard-extensions/attestation-services.html) to obtain attestation credentials
  and configure the local platform correspondingly.
  (This is not necessary if running Intel SGX in simulation mode)
- The peer adminstrator has to add the FPC external builder and launcher scripts have to the `externalBuilders` section in `core.yaml` and restart the peer. See an example `core.yaml` [here](https://github.com/hyperledger-labs/fabric-private-chaincode/blob/master/integration/config/core.yaml#L569).


### Deployment
The actual chaincode deployment follows mostly the standard Fabric pattern:

- The deployer creates a package with the main FPC deployment artifact, the `enclave.signed.so` enclave binary, and deployment metadata.
  FPC provides convenience scripts to facilitate this step.
  (See [Chaincode Development Section](#Chaincode) for more info on `enclave.signed.so` and `MRENCLAVE` referenced below.)
- Subsequently, the deployer follows the standard Fabric 2.0 Lifecycle steps, i.e., `install`, `approveformyorg` and `commit`. 
  A noteworthy FPC specific aspect is that the version used *must* be the code identity of the FPC chaincode, i.e., `MRENCLAVE` for SGX.
  <!-- 
    - We might eventually want also to explain why requireing `version=MRENCLAVE` is necessary, 
      i.e., to get agreement among all participants on the functionality.  
      This thought then also requires explaining that due to the privacy requirements and contrary to what Fabric enabled in v2.0,
      different peers *must* run the same code and cannot provide their own implementation. 
      (see related discussion on this on old "full FPC RFC".)
      As this moot in FPC Lite which supports only a single enclave, i omit the corresponding discussion for now.
	  Note, though, we do have some corresponding text, though, already later in Section "Fabric Features Not (Yet) Supported"
  -->
- Lastly, once the chaincode definition for an FPC chaincode has been approved by the consortium, 
  a Chaincode Enclave is created via a new `initEnclave` lifecycle management command. 
  <!-- commented out as not really user visible
    This command triggers the creation of an enclave and registers it at the FPC Registry.  
    The registration includes the attestation of the Chaincode Enclave.
  -->
  For more information on additional management commands, see the [FPC Management API document](https://github.com/hyperledger-labs/fabric-private-chaincode/blob/master/docs/design/fabric-v2%2B/fpc-management.md).

<!-- commented out as not really user visible
The FPC Registry stores all attestation reports, as signed by the TEE vendor. 
The attestation reports include information on what chaincode is running; what specific TEE is in use, including version number and hash of the chaincode (MRENCLAVE), and including the public keys of the enclave. 
Upon creation each chaincode enclave generates public/private key pairs for signing and for encryption. The public signing key also denotes the chaincode enclave identity.

In order to support the endorsement model of Fabric, chaincode enclaves executing the same FPC chaincode need access to a set of shared secrets. In particular, these shared or common keys are used to encrypt and decrypt the transaction arguments and state.
That is, there is also a public/private key pair and a symmetric state encryption key per FPC chaincode that are shared among all chaincode enclaves that run the same FPC chaincode using the key distribution protocol.
-->

The deployment is described in detail below and can be found in the [Full Detail Diagrams](#full-detail-diagrams) section.

## Requirements

- In the initial version of FPC, chaincode must be written in C/C++ using our FPC SDK. More language support is planned.
- FPC Endorsing Peers must use the FPC chaincode runtime.
- FPC’s runtime relies on the External Builder and Launcher feature of Fabric
  <!-- comment out below as we haven't introduced registry yet and, at least so far, we have put the registration under the cover of cli
   FPC's attestation infrastructure requires the installation of an FPC Registry chaincode per channel.
  -->
- FPC currently requires that the Endorsing Peers run on an Intel&reg; x86 machine with the SGX feature enabled. We plan to add support for other Trusted Execution Environments in future releases.


# FPC Lite Architecture
[architecture]: #architecture
<!-- 
    Description of section: This section makes the FPC vs FPC Lite distinction (& related restrictions)
    and then provides an outline of the architecture of FPC Lite.
    Note: we should use here preferably the term FPC Lite rather than FPC.
-->

The first realization of FPC is called *FPC Lite*.
The framework enables a class of applications which do not require release of sensitive data conditioned on private ledger state.
This class includes smart contracts which operate on sensitive medical data, or enforce confidential supply chain agreements.
The [Roll-back Protection Extension](#rollback-protection-extension) to FPC Lite can enable support of an even larger class of applications.


### Use case: Privacy-enhanced Federated Learning on FPC Lite

***TODO resolve below***
- *MB:I dont see this section here under architecture; better as subsection of motivation;*
- *MS: this section is here as it is to motivate FPC Lite and why even a restricted programming model makes sense.  Note right now before architecture we were generic and discussed the broader uses-cases; also FPC Lite is only intrduceded here as are the constraints which are referenced further down in this section. So i still think it has to be here (or the motivation section has to be greatly changed.)*

***TODO: @Jeb can you please make pass on below and replace this section as appropriate with info from HBP, UMBC or alike***

For example, Federated Learning on private sensitive information is a real-world use-case which FPC Lite enables securely on Hyperledger Fabric.
To illustrate this, consider the case of training a model, e.g, Convolutional Neural Network (CNN), for detecting brain abnormalities such as precancerous lesions or aneurysms.
To achieve high accuracy we need considerably more data than single entities (e.g., a hospital) usually has.
Yet regulations like HIPAA make sharing brain CT scans or MRI studies, labeled by radiologists, hard if not impossible. Furthermore, to allow widest use of a derived model, it should be freely shareable without any privacy concerns. Lastly, to provide the necessary accountability and audit trail, such a federated application is ideally tied to a ledger. While there are cryptographic solutions to perform federated learning in a strongly private manner, e.g., based on differential privacy, for such a setting, they are very expensive in terms of computation and, in particular, communication complexity.

A sketch on how this could be solved as follows:
- The overall approach would be to follow the [“PATE, for Private Aggregation of Teacher Ensembles”](https://blog.acolyer.org/2017/05/09/semi-supervised-knowledge-transfer-for-deep-learning-from-private-training-data/) approach to achieve the necessary strong privacy, i.e., differentially private, guarantees on the learned model to be released to the public.
- Above papers assume that the learning of the privacy-preserving model is performed by a trusted entity. In our example, an FPC Lite chaincode will perform this role, ensuring the integrity of the computation as well as the confidentiality of the training data.
- More specifically, the participating hospitals would compute separate teacher models on their own data and send the resulting model encrypted and signed to the chaincode. The chaincode would authenticate and validate the teacher models based on parameters apriori agreed and built into the chaincode, accumulate and record the submission in the ledger and, once sufficient inputs are received, will perform privately inside the chaincode the final student model computation. It will then publish the resulting model, e.g., via put_public_state,on the ledger.

Note such an application would not release any sensitive data conditioned on private state and hence meets our criteria.

## Overview of Architecture

The FPC-Lite architecture is constituted by a set of components which are designed to work atop of an unmodified Hyperledger Fabric framework: the FPC chaincode package and the Enclave registry chaincode, which run on the Fabric Peer; the FPC client, which sits onto the Fabric client. The architecture is agnostic to other Fabric components such as the ordering, gossip or membership services.
 
![Architecture](../images/fpc/high-level/Slide2.png)

Within the peer, the TEE (i.e., the enclave) determines the trust boundary that separates the sensitive FPC chaincode (and shim) from the rest of system.
In particular, the TEE enhances confidentiality and integrity for code and data inside the enclave against external threats from untrusted space.
Also, the code inside the enclave can use secret keys and cryptographic mechanism to securely store (resp. retrieve) any data to (resp. from) the ledger in untrusted space.

The FPC chaincode implements the smart contract logic (see the FPC chaincode development section).
The FPC shim interface is similar to the Fabric shim interface.
Most importantly, it implements the security features to protect any sensitive data (e.g., authenticated encryption/decryption of ledger data, digital signatures over responses, etc.).
Notably, none of these features (or relative cryptographic keys) are exposed to the FPC chaincode.

During an FPC transaction invocation, the FPC shim represents one endpoint of the secure channel between the FPC client and the FPC chaincode.
Hence, at the lower level of the protocol stack, the Fabric client and the peer only handle encrypted and integrity protected transactional information.

The Enclave Registry is a regular chaincode that helps establish trust in the enclave and the secure channel.
After an FPC chaincode definition is committed on the channel, the chaincode's hosting enclave must be registered with the Enclave Registry, in order to become operational.
The registry verifies and stores on the ledger the enclave attestation (signed by the trusted hardware manufacturer) and its public keys.
This allows any channel member to verify the genuinity of the enclave, including the intended FPC Chaincode, and establish trust in its public keys.

Finally, the Validation Logic verifies enclave responses and persists updates to the ledger.
In particular, the FPC client performs a regular Fabric invocation to the Validation Logic for validating signed and encrypted enclave responses, before delivering any response to the upper layer.
The validation logic verifies the correctness of the enclave execution through the Enclave Registry and applies any state updates.
As the logic is bundled together with the FPC chaincode in a single Fabric chaincode package,
these updates are eventually committed within the same namespace.
Hence, they will be visible to the FPC chaincode in subsequent invocations.

**Not part of the architecture**. The Ordering Service is treated as a trusted element in FPC networks, and securing it is outside the scope of FPC.
Likewise, FPC does not directly address problems of clients attempting to manipulate the Chaincode's output by providing bad input.
This must always be a matter of application-specific code discipline, and rely on native Fabric features which provide resilience and integrity through redundancy and distribution.
Ultimately, FPC is complementary to these existing features of Fabric.


## FPC Shim

The framework offers a C++ based FPC Shim to FPC chaincode developers.
Such shim follows the programming model of the Fabric Go shim, though using a different language.
For MVP, the FPC Shim comprises a subset of the standard Fabric Shim and is complemented in the future.
These details are documented separately in the Shim header file itself: **[ecc_enclave/enclave/shim.h](https://github.com/hyperledger-labs/fabric-private-chaincode/blob/master/ecc_enclave/enclave/shim.h)**

## Enclave Registry

Also referred to as the Enclave Registry Chaincode (ERCC), this is a component which maintains a list of all Chaincode Enclaves deployed on the peers in a channel.
The registry associates with each enclave their identity, associated public keys and an attestation linking them.
Additionally, the registry manages chaincode specific keys, including a chaincode public encryption key, and facilitates corresponding key-management among authorized chaincode enclaves.
Lastly, the registry also records information required to bootstrap the validation of attestation.

All of the above listed information is committed on the ledger.
This enables any channel member to inspect the attestation results before taking actions such as connecting to that chaincode or committing transactions produced by an FPC chaincode.
The registry is particularly relevant for clients, for retrieving an FPC chaincode's public keys and set up a direct secure channel.



## Deployment Process

We assume that:
* an existing Fabric channel is up and running,
* organizations are equipped with the FPC External Launcher,
* the FPC chaincode admin has registered with the Intel® Attestation Service (IAS).

![Deployment](../images/fpc/high-level/Slide3.png)

### Deploy the Enclave Registry Chaincode in the channel

The organizations follow the usual procedures the install the chaincode package, approve it and commit it.
In fact, the Enclave Registry is designed as a regular chaincode.


It is recommended to specify a strong endorsement policy (e.g., majority), since the Enclave Registry operations are integrity-sensitive.


### Deploy an FPC Chaincode package

An FPC Chaincode package is no different than a regular Fabric package.
Hence, the deployment follows the usual procedures.

For the successful completion of the deployment process,
it is necessary that all organizations use the **same** package.
This will ensure that the organizations will use the same chaincode version (i.e., the same MRENCLAVE, FPC Chaincode hash, and thus same in-enclave executable).
The chaincode version is stored in the approved and committed chaincode definition.
At registration time, this value will be checked by the Enclave Registry.

It is recommended to specify a strong endorsement policy (e.g., majority), since the Enclave Endorsement Validation operations are integrity-sensitive.

Following this deployment step, from a Fabric perspective, the FPC chaincode is operative.
However, any FPC Chaincode query/invocation will return an error because the FPC chaincode enclave (and its cryptographic material) are still uninitialized. 

### Initialize and Register the FPC Chaincode enclave.

**(FPC Lite makes the simplifying assumption that, for each deployed FPC package, a single enclave is initialized and registered.)**

The administrator of the peer hosting the FPC chaincode enclave is responsible for the initialization and registration of the enclave.
The operation is performed by executing the `initEnclave` admin command.

The command can be triggered through the FPC Client SDK.
<!--
Alternatively, the administrator can use the Fabric client to:
query the `initEnclave` FPC chaincode function;
pass the result as an argument in the invocation to the `registerEnclave` Enclave Registry chaincode function.
-->
A successful initialization and registration will result in a new entry (the enclave credentials, which include the chaincode definition, attestation and key material) in the registry namespace on the ledger.

Internally, the command operates as follows.
First, it issues an `initEnclave` query which reaches the FPC Shim.
The shim initializes the enclave (which will process FPC chaincode invocations) with the chaincode parameters received from the peer, namely: the chaincode definition and the channel identifier.
Then, it generate the key material: public/private key-pairs for signing and encryption, and the state encryption key.
The public signing key is used as enclave identifier.
The shim completes the execution by returning the `credentials` of the FPC Chaincode: all public parameters and a hardware-based remote attestation covering them.
Importantly, the code identity (or hash of the FPC Chaincode) specified in the attestation should match the version in the chaincode definition.

In the case of EPID-based Intel® SGX, attestations must be converted into a publicly-verifiable evidence by contacting IAS.
The FPC Client SDK performs this step using the admin's IAS subscription.

At this point the root CA certificate of the trusted hardware manufacturer represents the root of trust for the publicly-verifiable credentials.

The command then continues with the enclave registration.
In particular, it invokes the `registerEnclave` function of Enclave Registry chaincode, supplying the publicly-verifiable FPC chaincode credentials as argument.
The registry verifies the enclave credentials and, in particular: the publicly-verifiable evidence; the key material; and the expected chaincode definition by locally querying the Fabric `__lifecycle` chaincode.
The recommended FPC Registry endorsement policy is meant to protect the integrity of:
these checks; and the credentials stored on the ledger and thus available to channel members.

## FPC Transaction Flow

![Transaction](../images/fpc/high-level/Slide4.png)

### Client Invocation of the FPC Chaincode

The client application performs an FPC chaincode invocation through the FPC Client SDK.
If not cached locally, the SDK will query first the Enclave Registry chaincode to retrieve the FPC chaincode's public encryption key.
The SDK then generates a symmetric key, for the encryption of the response, and encrypts it and the invocation arguments with the FPC chaincode's public encryption key.
This encrypted message is then sent as a query to the FPC chaincode on the peer hosting the Enclave.
Information on the location of that peer is also stored in the Enclave Registry.
This query, as well as the query to the registry, are through calls to the regular Fabric SDK.
All these steps are fully transparent to the applications layer.

The query is forwarded via the peer and some glue code to the FPC Shim inside the Enclave.
Inside the Enclave, the FPC Shim decrypts the arguments of the proposal and saves the client's response encryption key.
After that, it invokes the FPC Chaincode with the plaintext arguments.

### Chaincode Execution

The FPC Chaincode processes the invocation according to the implemented chaincode logic.
While executing, the chaincode can access the World State through `getState` and `putState` operations provided by the FPC Shim. From the chaincode perspective, the arguments of the former, and the output the latter, are plaintext data.

The FPC Shim fetches the state data from the peer and loads it into the enclave, and similarly it stores data by forwarding it to the peer.
Most importantly, the shim maintains the read/writeset, and uses the State Encryption Key to
(a) authenticate and encrypt data with AES-GCM during store operations, and
(b) to decrypt and check the integrity of data during fetch operations.
From the peer perspective, store and fetch operations have encrypted arguments and outputs.
Note, though, that while values are encrypted, the keys are maintained in cleartext and visible to the peer.

An FPC Chaincode invocation completes its execution by returning a (plaintext) result to the FPC Shim.
The FPC encrypts the result with the client's response encryption key.
Then, it produces a cryptographic signature over the input arguments, the read-write set, and the (encrypted) result.
This is conceptually similar to the endorsement signature produced by the endorsing peer.

The FPC Shim completes its execution by returning the signature, the read/writeset and the result to the peer.
Finally, the peer packages the received data in transaction response and signs it.
This is a regular Fabric endorsement, which is sent back to the client.

### Enclave Endorsement Validation

The FPC Client SDK receives the proposal response (to the FPC Chaincode query) and starts the enclave endorsement validation.
Recall that the response contains the enclave signature, the read/writeset and the (encrypted) result.
The FPC Client SDK performs chaincode invocation to the enclave endorsement validation, and provide the response as an argument.

The enclave endorsement validation logic processes the response as follows.
It makes a chaincode-to-chaincode query to the Enclave Registry to get the chaincode definition and the enclave verifying key related to the FPC chaincode.
Then it checks that the FPC chaincode definition matches the one stored in the registry, and that the key verifies the enclave signature.

If checks pass, the logic re-applies the read/writeset through regular `getState` and `putState`.
Note that the enclave endorsement validation and the FPC chaincode belong to the same chaincode package and, therefore, they share the namespace.
The logic completes its execution by returning a `success` (or `error`) result to the peer.
In turn, the peer endorses the result and returns it is to the client.

At this point, we emphasize the importance of the endorsement policy related to the enclave endorsement validation.
As the validation is integrity-sensitive and designed as a regular chaincode, it is recommended the use of a strong policy (e.g., majority).

The Fabric client will therefore wait for enough peer endorsements (related to the execution of the enclave endorsement validation).
Then, it will package the responses in a transaction, which is sent to the orderers.
As the transaction commits (or returns an error), the Fabric client returns the outcome to the FPC client.

Finally, the FPC client decrypts any successful encrypted response, coming directly from the FPC chaincode.
Then, it deliveres the plaintext response to the application layer.


## TEE Platform Support
***TODO***

Currently, our FPC Runtime and the SDK focuses on [Intel&reg; SGX SDK](https://github.com/intel/linux-sgx).
However, components such as the FPC Registry are already designed to support attestations by other TEE platforms as they mature and gain remote attestation capabilities. Also, other components such as the Go part of the FPC Shim don't have an Intel&reg; SGX depency and can easily be reused. We plan to explore other TEE platforms such as AMD SEV in the future.

## Fabric Touchpoints

FPC-Lite does ***not*** require any changes to Fabric.
We recommend reading the chaincode development and deployment sections to know more about the private chaincode coding language and the use of Fabric's external builder and launcher capabilities.

## Fabric Features Not (Yet) Supported

In order to focus the development resources on the core components of FPC, the MVP `FPC Lite` excludes certain Fabric features, which will be added in the future.

- Multiple implementations for a single chaincode.
This feature is supported in Fabric v2 and gives organizations the freedom to implement and package their own chaincode.
It allows for different chaincode implementations as long as changes to the ledger state implement the same agreed upon state machine, i.e., the application integrity is ensured.

FPC chaincodes have a stronger requirement: Not only must we be assured of the application integrity but we also require that all information flows be controlled to meet our confidentiality requirements.   As the execution during endorsement is unobservable by other organizations, they will require the assurance that any chaincode getting access to the state decryption keys, and hence sensitive information, will never leak unintended information.  Therefore, the implementation of a chaincode must allow for public examination for (lack of) potential leaks of confidential data.
Only then clients can establish trust in how the chaincode executable treats their sensitive data.

For a given chaincode, the FPC Lite currently supports only a single active implementation.
Most importantly, the version field of the chaincode definition precisely identifies the chaincode's binary executable.

- Multiple key/value pairs and composite keys as well as secure access to MSP identities via `getCreator` will be supported once below [Roll-back Protection Extension](#rollback-protection-extension) is added.
- Arbitrary endorsement policies
- State-based endorsement
- Chaincode-to-chaincode invocations (cc2cc)
- Private Collections
- Complex (CouchDB) queries like range queries and the like

## Rollback-Protection Extension

FPC Lite is not designed for chaincodes which are implemented to release confidential data once some conditions are met on the ledger.
In fact, although chaincodes can protect the confidentiality and integrity of any data that they store on the ledger, they have no means to verify whether such data has been committed.
Hence, their hosting peer might provide them with legitimate yet stale, or non-committed, ledger data.

FPC Lite is therefore not suitable for the class of applications that require a proof of committed ledger data.
This includes, for example, smart contracts implementing sealed auctions, or e-voting mechanisms.
Arguably, these applications require checking whether a condition is met (e.g., "if the auction is closed") in order to release confidential data (e.g., "then the highest bid is X").

Extensions to the FPC Lite architecture can enable coverage of this larger class of private applications.
In particular, the framework can provide chaincodes with a verifiable proof of committed ledger data by implementing a trusted ledger enclave.
The design documents referenced in [Design Documents](#design-documents) already outline the path to realize such architecture extension.

## References to Design Documents

The full detailed protocol specification of FPC is documented in a series of UML Sequence Diagrams:

- The [fpc-lifecycle-v2](../images/fpc/full-detail/fpc-lifecycle-v2.png) diagram describes the lifecycle of a FPC chaincode, focusing in particular on those elements that change in FPC vs. regular Fabric.
- The [fpc-registration](../images/fpc/full-detail/fpc-registration.png) diagram describes how an FPC Chaincode Enclave is created on a Peer and registered at the FPC Registry, including the Remote Attestation process.
- The [fpc-key-dist](../images/fpc/full-detail/fpc-key-dist.png) diagram describes the process by which chaincode-unique cryptographic keys are created and distributed among enclaves running identical chaincodes. Note that in the current version of FPC, key generation is performed, but the key distribution protocol has not yet been implemented.
- The [fpc-cc-invocation](../images/fpc/full-detail/fpc-cc-invocation.png) diagram illustrates the the chaincode invocation part of an FPC transaction flow, focusing on the cryptographic operations between the Client and Peer leading up to submission of an FPC transaction for Ordering.
- The [fpc-cc-execution](../images/fpc/full-detail/fpc-cc-execution.png) diagram provides further detail of the execution phase of an FPC chaincode, focusing in particular on the `getState` and `putState` interactions with the Ledger and verification of state with the Ledger Enclave.
- The [fpc-validation](../images/fpc/full-detail/fpc-validation.png) diagram describes the FPC-specific process of validation and establishing a trusted view of the ledger using the Ledger Enclave.
- The [fpc-components](../images/fpc/full-detail/fpc-components.png) diagram shows the important data structures of FPC components and messages exchanged between components.
- The [interfaces](https://github.com/hyperledger-labs/fabric-private-chaincode/blob/master/docs/design/fabric-v2%2B/interfaces.md) document defines the interfaces exposed by the FPC components and their internal state.

Note: The source of the UML Sequence Diagrams are also available on the [FPC Github repository](https://github.com/hyperledger-labs/fabric-private-chaincode/tree/master/docs/design/fabric-v2%2B).

Additional google documents provide details on FPC Lite:
- The [FPC Lite for Health use case](https://docs.google.com/document/d/1jbiOY6Eq7OLpM_s3nb-4X4AJXROgfRHOrNLQDLxVnsc/) describes how FPC Lite enables a health care use case, without requiring a trusted ledger.
- The [FPC Lite externalized endorsement validation](https://docs.google.com/document/d/1RSrOfI9nh3d_DxT5CydvCg9lVNsZ9a30XcgC07in1BY/) describes the FPC Lite enclave endorsement validation mechanism.

# Repositories and Deliverables

***TODO: first cut into such a section picking up on what Dave said. This is modeled after the [WASM PR](https://github.com/hyperledger/fabric-rfcs/pull/28). Look also in markdown comment here!***

It is anticipated that the following github repositories will be created.

- `fabric-private-chaincode`
    - The core infrastructure for FPC including the C Chaincode and Go client-side SDK, simple sample application with deployment based on `first-network` and related documentation.

<!--  Potentially we might want to split into additonal repos liks below, maybe also splitting Go and C from `fabric-private-chaincode`
- `fabric-private-chaincode-sdk-go`
    - Client side SDK in Go, including management API extensions
- `fabric-private-chaincode-wamr`
	- FPC chaincode WASM runtime via WAMR
-->

# Feature Roadmap

***TODO***

- Implement the [Roll-back Protection Extension](#rollback-protection-extension)

- WebAssembly Chaincode: A primary goal for FPC moving forward is to support WebAssembly chaincode, and by extension all languages that compile to WASM.
There has already been extensive development of a high-performance open source WASM Interpreter / Compiler for Intel&reg; SGX Enclaves in the [Private Data Objects](https://github.com/hyperledger-labs/private-data-objects) project, and our current plan is to adopt that capability in the next major phase of FPC.
FPC's modular architecture has been designed from the beginning to enable this drop-in capability.

- Other TEEs: The FPC team is also participating in early discussions in the [Confidential Computing Consortium](https://confidentialcomputing.io/), which aims to provide a standardized way of deploying WASM across multiple TEE technologies.
We hope to leverage this work when extending FPC to support these other TEEs.

- Endorsing Policies: The current version supports only a single designated FPC Endorsing Peer; it is our intention to support multiple FPC Endorsing Peers in the upcoming MVP release.
In future releases this would enable rich Endorsement Policies as described above.

- Risk Management and Deployment Policies: We intend to support deployment policies which allow the users to define where a FPC chaincode is allowed to be deployed.  For instance, the user could define that a certain FPC chaincode can only executed by an endorsing peer on a on-premise node of a certain organization.  This allows enhanced risk management.

- Private Data Collections: We have not tested the combination of FPC with Private Collections in Fabric 2.0, but intend to support this combination of features in a future release.

- Chaincode2Chaincode Invocations: The initial version of FPC does not support
to call other chaincodes from a FPC chaincode; as this is a useful feature often used by chaincode applications, we intend to support this functionality in a future release.

- Multiple FPC Channels: The current version only supports a single FPC Channel; it is our intention to support arbitrary numbers of Channels of both FPC and regular Fabric in future releases.

- Trusted Ledger State Snapshot:
  Currently, the Trusted Ledger keeps the ledger meta-data state in secure memory only, i.e., it does not persist it.
  This means that on restart of the peer, the complete transaction-log has to be re-read and also limits how large the ledger meta state can be.
  In a future release, we will add secure storage of the meta-data state which will address both of these concerns and will provide better restart performance and better scalability.

# Rationale and Alternatives
[alternatives]: #alternatives

The non-adoption of the FPC design in Hyperledger Fabric has the following impact:
to miss the opportunity to expand the application domain with privacy-sensitive use cases, which Fabric is not designed for.
The execution of chaincodes at the endorsing peers requires in fact to expose their information to the peer platforms and their respective organizations.

Some alternative approaches to FPC are:

- Support a different hardware-based TEEs (e.g., AMD SEV).
The FPC architecture can be extended to run chaincodes in other TEEs, which have support for publicly-verifiable hardware-based remote attestation.
- Use Secure Multi Party Computation, Fully Homomorphic Encryption, or other pure software-based cryptographic solution.
These technologies have been proved effective to deliver confidentiality for execution and data, albeit with a high computational overhead.
- Run the entire peer inside a TEE.
This approach is viable, and tools like Graphene SGX might be starting point towards the objective.
However, this design would result in a bloated Trusted Computing Base (TCB). Hence, the security issues of a component may affect the others within the TCB.

# Prior art
[prior-art]: #prior-art

The initial architecture of FPC is based on the work in the paper:

* Marcus Brandenburger, Christian Cachin, Rüdiger Kapitza, Alessandro
Sorniotti: Blockchain and Trusted Computing: Problems, Pitfalls, and a
Solution for Hyperledger Fabric. https://arxiv.org/abs/1805.08541

FPC is closely related to the Private Data Objects (PDO) project (https://github.com/hyperledger-labs/private-data-objects) and Hyperledger Avalon (https://github.com/hyperledger/avalon).
A distinctive aspect is that FPC is tighly integrated with Hyperledger Fabric, while PDO and Avalon are ledger-agnostic and might require additional components as well as an extended trust model.
Note that the PDO team is also involved in the design and development of FPC.

Additionally, Corda (https://docs.corda.net/design/sgx-integration/design.html) proposes the use of Intel&reg; SGX to protect privacy and integrity of smart-contracts.

# Development and Testing
[testing]: #testing

FPC relies on the presence of Trusted Execution Environment (TEE) hardware which is not available to all developers; in the initial releases the TEE is Intel&reg; SGX, which is not available on Mac, for example. Therefore, FPC provides a Docker-based development environment containing a preconfigured SGX Simulator in a Linux container.
This environment can also be used in hardware mode if the underlying platform includes SGX support.
Beyond SGX software, the containerized development environment includes also all other dependencies needed for a new developer to start the container and immediately begin working on their first FPC Chaincode.
It is setup in a way which still also you to easily edit files on the host using your normal development environment.

The FPC team’s current practices include both unit and integration testing, using above docker environment to automate end-to-end tests. 
CI/CD is enabled on Github via Travis.
This includes also linter checks for both C and Go as well as checking for appropriate license headers.
<!-- Comment out auction demo as intrinsically not FPC Lite enabled, will have to wait for Full FPC ..
With the Auction Demo scenario, we also include a representative example which illustrates end-to-end how to design, build and deploy a secure FPC application across the complete lifecycle.  In addition, this demo serves as an additional comprehensive integration test for our CI/CD pipeline. 
-->

Once FPC becomes maintained as an official Fabric project, we will explore publishing our (existing) FPC-specific docker images in a registry.

# Terminology

* Attestation: An important cryptographic feature of Trusted Execution Environments by which the hardware can reliably state exactly what software it is running. This statement is signed by the hardware so that anyone reading it can verify that the statement came from an actual valid and up-to-date TEE. The Attestation can cover both the software portion of the TEE itself as well as any program to be run in it. This attestation takes the form of a "quote" containing (among other things) a measurement of the code and data in the TEE and its software version number, which is signed by the TEE and can be verified by anyone using well known Public Key Infrastructure technology.

* Trusted Execution Environment (TEE): The isolated secure environment in which programs run in encrypted memory, unreadable even by privileged users or system processes. FPC chaincodes run in TEEs.

* Enclave: The TEE technology used for the initial release of FPC will be Intel&reg; SGX.  In SGX terminology a TEE is called an _enclave_.  In this document and in general, the terms TEE and Enclave are considered interchangeable.  Intel&reg; SGX is the first TEE technology supported as, to date, it is the only TEE with mature support for remote attestation as required by the FPC integrity architecture.  However, our architecture is generic enough to also allow other implementations based on AMD SEV-SNP, ARM TrustZone, or other TEEs.

Note on Terminology: The current feature naming scheme includes several elements that originated in Intel&reg; SGX and therefore use the formerly proprietary term Enclave rather than TEE. Earlier versions of this RFC stated an aim to replace the term Enclave with TEE, but since then the two terms have come to be accepted as interchangeable. We therefore decided not to try to expunge the term, but to use it as the industry has begun to do, to refer to various TEEs. The project team currently participates in the new Confidential Computing Consortium which aims to promote standards for deployment of workloads across TEEs/Enclaves, and we intend to align with their terminology as it evolves.
