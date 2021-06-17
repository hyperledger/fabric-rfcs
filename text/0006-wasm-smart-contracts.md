---
layout: default
title: Wasm_Smart_Contracts
parent: RFCs
---

- Feature Name: Support writing Smart Contracts to be executed within a WebAssembly runtime
- Start Date: 2020-05-20
- RFC PR: (leave this empty)
- Fabric Issue: (leave this empty)

# Summary
[summary]: #summary

This RFC is providing:

* a WebAssembly runtime that can be used as a Chaincode implementation in Fabric
* implementation of a set of APIs to permit Smart Contracts to be written in Rust and then compiled to Wasm bytecode
* documented interface to permit the community to implement Smart Contracts in other languages. 

A Rust developer would write their Smart Contract with Rust and produce a library. This they would then target to be compiled to Wasm bytecode - the resulting wasm binary would then be deployed using Fabric v2 "External Builders" into a Wasm runtime. This contract is then treated just the same as any other contract or chaincode. 

The APIs will be implemented at a level consistent with the existing Contract APIs but taking the opportunity to make improvements. 

Note: code examples are provided for illustration, and should be considered pseudo-code to outline the functionality that will be produced.

Prototypes are available for the [Rust API](https://github.com/hyperledgendary/fabric-contract-api-rust), [Wasm Chaincode](https://github.com/hyperledgendary/fabric-chaincode-wasm); work is still taking place in these repos so please get contact with questions.

# Motivation
[motivation]: #motivation

Wasm provides a "well defined, tightly specified, isolated" runtime. It is ideally suited to executing code in a well-controlled and safe manner. Originally intended for use within the browser it is perfectly suited to running outside the browser. The requirements of chaincode and smart contracts are ideally suited to the Wasm model. 

The Contract Programming Model initially introduced in v1.4 provided higher-level abstractions to both the contract and the client SDKs. Many users have already written high-level utility or support code within their applications. Abstractions help us think about and solve problems, improving usability, speed of development and performance. The API in this RFC will be the evolution of the existing work. 

Language support for compiling to Wasm is growing; Rust is one of the languages today that can be efficiently compiled to Wasm. C/C++ and Assembly Script are also candidates. Other languages will arrive over time, for example, Go. One of the deliverables of the Wasm runtime will be the protocol and API to allow the community to develop bindings for new languages. 


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

A complete solution using Fabric can be represented as follows:
![](0000-Complete-ConceptualArchitecture.png)

An application would typically connect to a peer (the 'gateway' into the network channel), and obtain a reference to a smart contract. The Network-shared transaction logic in the Smart Contract is then be invoked by name with optional data arguments and return values. The data arguments can include transient data if needed. The logic in the Smart Contract will typically reference and update the shared transaction data, subject of course to exactly how the code is written.

There are no application API changes as part of this RFC, the existing SDKs should be used.

To write a Smart Contract in Rust for deployment to the Wasm chaincode runtime, a developer will follow these steps

- create a Rust library crate
    - add the crate dependencies for the Contract-API and macros
    - and anything else they wish.  The developer is responsible for determining that the other libraries can be compiled correctly to Wasm
- implement the Contract trait for any struct they wish to be a Contract
    - using the documented macros for transaction functions
    - use the Ledger API to interact with data
- implement the DataType trait for any struct they wish to be passed to and from Transaction Functions
    - using the documented macros again here
- compile the crate targeted at Wasm
- configure the supplied External Builder to run the Wasm binary
    - using the external chaincode builder scripts supplied to run the Wasm chaincode
- interact with the deployed chaincode as per other chaincode implementations

## Development APIs
 For writing contracts, the API logically splits into three parts

- Definitions to define what a contract is, and mark transaction functions
- Definitions to define how the implementation of the contract will 
    - work with the data in the ledgers, and other collections.  
    - work with the concepts of Transactions, Invoking other Contracts. 
- Definitions to let the contract implementation discover identity information about the organization submitting the current transaction.

The first is generally called the Contract API, the second Ledger API. 

### Contract

The model used here is consistent with the existing Contract Model, with some minor improvements (that can be cascaded back to Java/Nodejs/Golang). 

- A deployed 'chaincode' will be implemented by one or more 'contracts'. 
- Each 'contract' will consist of one or more 'transaction functions'. 
- Each 'transaction function' can take arguments at the discretion of the developer. 
- Within each 'transaction function' the developer will then use the Ledger API to interact with the ledger data.
- Transaction Functions are addressed by name defined as the name of the contract, followed by the name of the transaction function
    - A default contract is defined implying that the transaction functions within the default contract can be addressed by transaction function name alone.
    - If only one contract is defined, this will implicitly be assumed to be the default.

For example:

```rust
// macros for marking up the contract
use contract_macros::contract_impl;
use contract_macros::transaction;

/// Structure for the AssetContract, on which implemenation transaction functions will be added
pub struct AssetContract {

}
/// Implementation of the contract trait for the AssetContract
/// There are default implementation methods, but can be modified if you wish
/// 
/// Recommended that the name() function is always modified
impl Contract for AssetContract {
    //! Name of the contract
    fn name(&self) -> String {
        format!("AssetContract")
    }
}

#[contract_impl]
impl AssetContract {
    #[transaction]
    pub fn asset_exists(assset_id:  String) -> Result<bool,ContractError> {
        let ledger = Ledger::access_ledger();
        let world = ledger.get_collection(CollectionName::World);

        Ok(world.state_exists(assset_id))
    }
}
```

The developer could write several such contracts. As rust is a static language without runtime introspection, the developer will need to explicitly identify the contracts to the runtime.

For example:

```rust
launch_handler!(register);

/// Register the contracts
/// Look to fold this into the macro (or the contract_impl macro)
pub fn register() -> i32 {
    ContractManager::register_contract(Box::new(AssetContract::new()));

    // return 0 to indicate success, other values imply failure and will terminate
    // the chaincode container
    0
}
```

### Ledger

A Ledger's current value is held in a set of collections. A single ledger is available to any given deployed contract, though other ledgers may be available within the Network Channel and can be accessed via other contracts

Of the collections that are reference by the ledger, there a single collection in the ledger that is public (the 'WorldState') and held by every organization in the Network Channel. A ledger may have 1 or more private collections that are available only in some organizations. Starting at Fabric v2.0 there will be an implicit private collection per organization.

## Collection

A collection is a set of states, with each state holding a business object or data. Each state being addressed by a key. Private collections are identified by name and held within a set of policy-defined organizations.

Accessing any given collection is via the Ledger instance. 

```rust
    let Ledger = Ledger::access_ledger()

    // predefined constant for the world state
    let world = ledger.get_collection(CollectionName::World);

    // Private Data collections are reference by name
    let private_collection = ledger.get_collection(CollectionName::Private(String::from("PrivateSalesData")));

    // Each organization will have an implicit private data collection created for them
    let orgs_collection = ledger.get_collection(CollectionName::Organization(String::from("org1")));

```

Should any operation error or not be applicable in some circumstances, a 'not-supported' error will be returned.

## State

A state holds the value of a business object or data, addressed by a key. The format of this object or data is currently defined by the overall solution, and encoding the format of 'the value' of the state is the responsibility of the contract developer.

The state can also anchor the following functions
 - Access to individual states can be controlled by state-level endorsement policies, APIs are provided to assist in the construction and parsing of these policies. 
 - Access to the Private Data Hash - essential for private data scenarios.
 - Access to history details of the state, what transactions affect this state in the past
 - Creation of composite keys and general key handling helpers


The approach being adopted is that data will be put into 'states' that are then given to 'collections. Using the `from` and `into` pattern in rust - more information is in the DataTypes section below. 

## Example Interfaces

### Putting data

The 'issue' method will create a new business object based on supplied data, and store this within the Ledger.

```rust

    #[Transaction(submit = true)]
    fn create_asset(assetIdString issuer, String assetId, int faceValue) -> Result<MyAsset,ContractError>  {

        // get the collection that is backed by the world state
        let ledger = Ledger::access_ledger();
        let world = ledger.get_collection(CollectionName::World);

        let new_asset = MyAsset::new(my_assset_id,value);
        world.create(State.from(new_asset));

        Ok(())
     }
```

Access to the ledger is achieved via the `Ledger::access_ledger()`. Access then to the 'world-state' or default collection is via `getCollection(...)`

A key is needed and the `State::make_composite()` provides a helper function to create this key. `collection.create(state)` stores the state.  

#### Update and Create

Previously a stub API of `putState` was used; issues have been raised referring to what is called 'blind writes'. Namely, if a key is **put** without first being **get**  then the key will only be in the write set. This can have the effect that if two transactions do this the last one wins.

A solution presented in one of the [JIRA Issues](https://jira.hyperledger.org/browse/FAB-10480) was the create and update semantics.

- Create: if the key already exists fail, only put if the key does not exist.
- Update: if the key already exists put the state, if it doesn't fail. 

### Retrieval and Deleting

Retrieval and Deleting of state follow similar patterns. The key lines are

```rust
    // get the state
    String key = State::make_composite(...);
    let asset: MyAsset = collection.retrieve(key).into_datatype();
    // ....
    let asset: MyAsset = collection.delete(key).into_datatype();

```

### State-Based Endorsement

Endorsement Policies can be defined at the Contract, State, and (in v2.0) the collection level. Control of the State endorsement is by the contract code invoking APIs on the State object.  (details of the syntax are defined in the main docs)[https://hyperledger-fabric.readthedocs.io/en/master/endorsement-policies.html#endorsement-policy-syntax]


```rust
    // Create three principals
    let p1 = Principal::new("Org1", Role.MEMBER);
    let p2 = Principal::new("Org2", Role.MEMBER);
    let p3 = Principal::new("Org3", Role.MEMBER);

    // create an Endorsment based on the required logic
    let sbe1 = Endorsement::build(
            Expression::and(
                    Expression::or(p1,p2),
                    p3
                    )
            );
    assetState.set_endorsement(sbe1);
    
    // or the textual version can be parsed
    let sbe2 = Endorsement::build("AND( OR ('Org1.member','Org2.member'), 'Org3.member' )");
    assetState.set_endorsement(sbe2);
        
```

### Query

There are two broad approaches to querying data. Either by passing a query to be parsed by CouchDB (also called Rich Query), or by using the State keys. Composite keys form a hierarchy that can be used for querying a range of states.

To unify the different approaches, the API has a query method taking a configuration of the query to be performed, that returns an iterable object of results.  This iterable will contain the results, and be iterated with the appropriate language idioms. Control of pagination and the request for more data is controlled within this iterable.

For key-based query, the configuration can give any start and end keys, any pagination required. For rich query, this can contain the appropriate query string. 

```rust

    fn query_assets(startkey: String, endkey) {
        let ledger = Ledger::access_ledger();
        let collection = ledger.get_collection(CollectionName::World);

        // query a range, and use rust iterators support to process
        let states = collection.query(QueryHandler::Range(startkey,endkey)).map(|s| s.into_dataType());

        for a in assets {
            totalValue += a.get_value();
        }

    }

```

### State History

The history of a state, its value at a point time as a result of a transaction can be queried as follows. Iteration here follows the same patterns as for query. 

```rust
        
    // get the state
    let state = collection.retrieve_state(key);
    
    let history = state.get_history();
    for h in history {
        let timesstamp = h.get_timestamp();
        let txid = h.get_txid();
        let is_deleted = h.is_deleted();
    }

```

### DataTypes

As well as the core types (String, primitive scalar values and arrays of those) applications require more complex types. 

The existing ContractAPI implements this by providing a decorator to mark classes/structs etc as a 'DataType' and within those marking the 'Properties' that make up the object. This permits the contract metadata to be established so the external 'API Surface Area' of the contract is available to tooling and client applications. Users then define their serialization formats for both the wire and ledger protocols.

For example:

```rust
/// this defines a simple asset structure with a String
#[derive(DataType)]
pub struct MyAsset {
   #[property]
   value: String
}

impl DataType for MyAsset {
    fn get_key(&self){
        self.value
    }
}
```

The flexability of Rust with traits providing static polymorphism permits user defined serialization elegant manner. For example:

```rust
impl LedgerSerializer for MyAsset {
    fn to_buffer(&self) -> Vec<u8> {
        /// my own form of serialization eg convert to XML
    }

    fn from_buffer(buffer: Vec<u8>) -> MyAsset{
        /// my own form of serialization eg convert to XML
    }
}
```

By using the `from` and `into` standard patterns in then can be used. For example.

```rust
    let new_asset = MyAsset::new(my_assset_id,value);
    world.create(State.from(new_asset));
```

## Transactional Information

### Current Transaction 

To obtain the current transaction and the essential information:

```rust
    let transaction = Transaction::current_transaction();
    transaction.get_timestamp();
    transaction.get_id();
    transaction.get_mspid();
```

`get_mspid` is important for handling situations involving private data and being able to determine the originating organization. 

### Events

Currently, a single event payload and name can be produced from a chaincode. Feedback has indicated that at a minimum it should be possible to have multiple names and payloads.

The transaction, therefore, can support this

```rust
    transaction.set_event(evtName, eventData);
    transaction.get_event(evtName, eventData);
```

These will then be put into a JSON Object structure and returned. The further specification will be left to later work. 

### Identity

Information about the identity that requested the transaction can be obtained:

```rust
        let identity = transaction.get_client_identity();
        identity.attribute_exists(attrName,attrValue);
        identity.attribute_value(attrName);
        identity.getID()
        identity.getMSPID();
```

Will validate that this is the appropriate set of the accessors to support against real use-cases. 

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Wasm Runtime

These are the major components within the runtime.

- a Wasm runtime engine responsible for the execution of the Wasm binary
- grpc communications with the peer
- data marshalling for transfer of data and requests into and out of the Wasm engine
- Along with required code to bootstrap the whole Wasm engine and load in the binary

### Wasm Engine
Several different implementations are available; initial prototypes have used the LifeVM, but for final implementation, Wasmer is being considered. The WebAssembly landscape is still developing so there is some risk with changing standards and implementations, however, some of the new standards could be of benefit in the future.

### grpc Peer Communications
This will use the go chaincode shim today to provide connectivity to the peer.

### Data Marshalling
Currently, the API to import/export functions and data between the host and the executing code in the wasm runtime is limited to primitive data types only. A data and function marshalling interface is required, waPC will be used to provide this. 

The logical functions that will be marshalled over waPC from host to client will form part of the deliverables. This is to allow for new languages to be supported and also to permit changes to the peer communications protocol. (these logical functions are referred to as the Contract Core SPI). It will be designed to use the protobuf specification to permit accurate description.

## Lifecycle of the Wasm binary

The Wasm binary that is produced will be an input to the normal chaincode lifecycle with supplied Wasm external chaincode builder scripts. 

The Wasm binary will not be stored within the world state; on-chain storage and execution of WebAssembly smart contracts are beyond the scope of this RFC.

## Contract and Ledger APIs

A prototype implementation of the interfaces is available in [github](https://github.com/hyperledgendary/fabric-contract-api-rust) 

The core functionality of Fabric, previously available in the shim and stub APIs, is replicated within this RFC. 

# Repositories and Deliverables

It is anticipated that the following github repositories will be created.

- `fabric-builder-wasm`
    - external builder scripts and Dockerfile for a Wasm aware peer
- `fabric-chaincode-wasm`
    - the chaincode as a server to host wasm contracts
- `fabric-contract-api-rust`  
    - contracts 
- `fabric-ledger-protos`      
    - source for protos
- `fabric-ledger-protos-go`   
    - auto-generated from protos repo
- `fabric-ledger-protos-rust` 
    - auto-generated from protos repo

Documentation will be provided for the API for developers, and also for contributors who wish to allow the language of their choice to be used for contract development.


The implementation of this RFC will take place in the master branch of any repositories.


