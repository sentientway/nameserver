# User Guide
We will begin with a brief discussion regarding the importance of the nameserver module within a Proof-of-Stake (PoS) model for distributed applications. The Tendermint core requires 75% consensus for its BFT consensus algorithm. It follows that for a public blockchain, an application must have a minimum of four validators to meet Tendermint BFT requirements. See ==(include link)== for an in-depth review of blockchain consensus algorithms and how the Cosmos blockchain network model addresses the scalability issue associated with the PoS requirement to know all of the nodes (blockchains) and accounts in the network.
## Introduction
The Nameserver module state holds node and account data in a single key/value store (KVStores). Modules can have multiple KVStores. The SDK application, has one store called *multistore*, which contains multiple KVStores. 

The Nameserver module processes messages for the following string maps. The nameStore KVStore contains the mappings Name => Value, which returns a machine readable string that resolves to an account on the node; Name => Owner, which returns the owner; and Name => Price, which returns the bid to buy the name. The owner and name are also strings.

The Cosmos SDK provides a scaffolding tool to produce a boilerplate application. Within the SDK, there are two important components that tie everything together and handle communication with the Tendermint consensus engine. The two components are app.go and baseapp respectively. 

## App Package
This is where the application developer stitches together the Cosmos SDK components required for the application. Integrating the nameserver module is a matter of importing the module. The Type, CLI, Handlers, and Keeper are all defined to implement a distributed nameserver within the application.

    package app
    
    import (
    "github.com/cosmos/cosmos-sdk/x/nameserver"
    )

## Commands

The Cosmos SDK adopts the naming convention that maps APPNAME onto an app name abreviation with cli appended. For example, an application named namerserver maps to nscli. For the list of commands specific to the nameserver module the app name abreviation is *.

**Queries:**

	*cli query account $(*cli keys show *Owner* -a)
	
	*cli query nameservice resolve *Owner*.id
	
	*cli query nameservice whois *Owner*.id
	
**Transactions:**
	
	*cli tx nameservice buy-name *name*.id 5nametoken --from seller

	# Set the value for the purchased name
	*cli tx nameservice set-name *name*.id 8.8.8.8 --from *Owner*

	# buys name from *Owner*
	*cli tx nameservice buy-name *Owner*.id 10nametoken --from Buyer

	# Delete Name
	*cli tx nameservice delete-name *name*.id --from alice



