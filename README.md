# nameserver
## Abstract
This document describes the distributed nameserver module that performs the peer-to-peer function of DNS in the client server model. It includes a short user guide explaining the modules integration into an SDK-based application and a detailed description of the modules implementation.

The Cosmos SDK is a structure of interoperable modules aggregated to support Cosmos application development. Each module has its own Message/Transaction processor. The baseapp component in the SDK handles all communication with Tendermint, and routes all queries and transactions to each of the respective modules handlers.

The nameserver module stores the value, owner, and price of a name. To impliment this functionality, the module uses the Auth module for account services, the Bank module for transactions between accounts, the Staking module for updating validators, and the Param module to populate a global parameter store.
## Contents
1. **[User Guide](userguide.md)**
	- [Introduction](userguide.md#Introduction)
	- [App Package](userguide.md#App-Package)
	- [CLI](userguide.md#CLI)
2. **[Implementation](implementation.md)**
	- [Introduction](implementation.md#Introduction)
	- [Concept](implementation.md#Concept)
	- [State](implementation.md#State)
	- [Messages](implementation.md#Messages)
	- [Handlers](implementation.md#Handlers)
	- [Types](implementation.md#Types)
	- [Keeper](implementation.md#Keeper)
	- [Parameters](implementation.md#Parameters)
	