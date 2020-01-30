# User Guide
We will begin with a brief discussion regarding the importance of the nameserver module within a Proof-of-Stake (Pos) model for distributed applications. The Tendermint core requires 75% consensus for its BFT consensus algorithm. It follows that for a public blockchain, an application must have a minimum of four validators to meet Tendermint BFT requirements. See ==(include link)== for an in-depth review of blockchain consensus algorithms and how the Cosmos blockchain network model addresses the scalability issue associated with the PoS requirement to know all of the nodes (blockchains) and accounts in the network.
## Introduction
The Nameserver module state holds node and account data in a single key/value store (KVStores). Modules can have multiple KVStores. The SDK application, has one store called *multistore*, which contains multiple KVStores. 

The Nameserver module processes messages for the following string maps. Every account on a node (blockchain) is either a validator or a delagator as long as its stake is positive. The nameStore KVStore contains the mappings Name => Value, which is a machine readable string that resolves to an account on the node; Name => Owner, which is who owns the name; and Name => Price, which is the bid to buy the name. 

The Cosmos SDK provides a scaffolding tool to produce a boilerplate application. Within the SDK, there are two important components that tie everything together and handle communication with the Tendermint consensus engine. The two components are app.go and baseapp respectively. 

## App Package
This is where the application developer stitches together the Cosmos SDK components required for the application. Integrating the nameserver module is a matter of importing the module. The Type, CLI, Handlers, and Keeper are all defined to implement a distributed nameserver within the application.

    package app
    
    import (
    "github.com/cosmos/cosmos-sdk/x/nameserver"
    )

## CLI


