# Implementation Guide
## Introduction
The Nameservice module is one of several modules a developer will use to create an SDK application, whose specific logic and structure conforms to the general module structure.

The AppModule stitches the different modules together into a complete SDK application using the Tendermint core for consensus. Communication with the consensus engine over the Application Blockchain Consensus Interface (ABCI) uses BaseApp to implement this core logic.

This document presents the nameservice module from concept to development of the CLI commands.
## Concept
The nameservice module maps a name to an owner, resolution value, and name price. The module transfers names between users, deletes names, and creates new names that satisfy the minimum name price. The minimum price is set elsewhere in the SDK application.
## State
The state of this module is stored in a single key/value store (KVStore), which is added to a single multistore that contains all KVStores for the application. The triggers for state change are:

1. Creating a new name.
2. Removing an existing name.
3. Transfer of ownership.

Each of these actions require several transactions within the nameservice state machine. Messages that prompt these actions must adhere to a msgType, have a handler for each message, a keeper for the KVStore, the ability to handle queries, reference a Codec, and create a CLI.

A module can call the KVStore(s) of another module if it is passed.
## Messages
Each module has its own message and transaction processor. Several *routines* and *subroutines* are used across modules to handle messages related to module specific transactions.

Messages trigger state transitions. They are passed to *Handlers* who then pass the action to the appropriate *Keeper*, which updates *State*.

The separation of message processing between the module's handler and keeper enhances security. Each keeper, which updates *State*, is specialized to a *type*, whereas, depending on the module, handlers can specify different *types*. In the case of this module, there is only one type across all three handlers.

The nameservice module uses three messages:
- MsgSetName
- MsgBuyName
- MsgDelName
### Msg Type
	// Transactions messages must fulfill the Msg
	type Msg interface {
	// Return the message type.
	// Must be alphanumeric or empty.
		Type() string

	// Returns a human-readable string for the message, intended for utilization within tags
		Route() string

	// ValidateBasic does a simple validation check that
	// doesn't require access to any other information.
		ValidateBasic() Error

	// Get the canonical byte representation of the Msg.
		GetSignBytes() []byte

	// Signers returns the addrs of signers that must sign.
	// CONTRACT: All signatures must be present to be valid.
	// CONTRACT: Returns addrs in some deterministic order.
		GetSigners() []AccAddress
	}
## Handlers
The nameservice module uses three Handlers, one for each nameserver message, which are SetName, BuyName, and DeleteName.

**MsgSetName:**

	package types

	import (
		sdk "github.com/cosmos/cosmos-sdk/types"
	)

	const RouterKey = ModuleName // this is defined in your key.go file

	// MsgSetName defines a SetName message
	type MsgSetName struct {
		Name  string         `json:"name"`
		Value string         `json:"value"`
		Owner sdk.AccAddress `json:"owner"`
	}

	// NewMsgSetName is a constructor function for MsgSetName
	func NewMsgSetName(name string, value string, owner sdk.AccAddress) MsgSetName {
		return MsgSetName{
			Name:  name,
			Value: value,
			Owner: owner,
		}
	}

**MsgBuyName:**

	// MsgBuyName defines the BuyName message
	type MsgBuyName struct {
		Name  string         `json:"name"`
		Bid   sdk.Coins      `json:"bid"`
		Buyer sdk.AccAddress `json:"buyer"`
	}

	// NewMsgBuyName is the constructor function for MsgBuyName
	func NewMsgBuyName(name string, bid sdk.Coins, buyer sdk.AccAddress) MsgBuyName {
			
		return MsgBuyName{
			Name:  name,
			Bid:   bid,
			Buyer: buyer,
		}
	}

	// Route should return the name of the module
	func (msg MsgBuyName) Route() string { return RouterKey }

	// Type should return the action
	func (msg MsgBuyName) Type() string { return "buy_name" }

	// ValidateBasic runs stateless checks on the message
	func (msg MsgBuyName) ValidateBasic() sdk.Error {
		if msg.Buyer.Empty() {
			return sdk.ErrInvalidAddress(msg.Buyer.String())
		}
		if len(msg.Name) == 0 {
			return sdk.ErrUnknownRequest("Name cannot be empty")
		}
		if !msg.Bid.IsAllPositive() {
			return sdk.ErrInsufficientCoins("Bids must be positive")
		}
		return nil
	}

	// GetSignBytes encodes the message for signing
	func (msg MsgBuyName) GetSignBytes() []byte {
		return sdk.MustSortJSON(ModuleCdc.MustMarshalJSON(msg))
	}

	// GetSigners defines whose signature is required
	func (msg MsgBuyName) GetSigners() []sdk.AccAddress {
		return []sdk.AccAddress{msg.Buyer}
	}

	// NewHandler returns a handler for "nameservice" type messages.
	func NewHandler(keeper Keeper) sdk.Handler {
		return func(ctx sdk.Context, msg sdk.Msg) sdk.Result {
			switch msg := msg.(type) {
			case MsgSetName:
				return handleMsgSetName(ctx, keeper, msg)
			case MsgBuyName:
				return handleMsgBuyName(ctx, keeper, msg)
			default:
				errMsg := fmt.Sprintf("Unrecognized nameservice Msg type: %v", msg.Type())
				return sdk.ErrUnknownRequest(errMsg).Result()
			}
		}
	}

	// Handle a message to buy name
	func handleMsgBuyName(ctx sdk.Context, keeper Keeper, 	msg MsgBuyName) sdk.Result {
		if keeper.GetPrice(ctx, msg.Name).IsAllGT(msg.Bid) { // Checks if the bid price is greater than the price paid by the current owner
			return sdk.ErrInsufficientCoins("Bid not high enough").Result() // If not, throw an error
		}
		if keeper.HasOwner(ctx, msg.Name) {
			err := keeper.CoinKeeper.SendCoins(ctx, msg.Buyer, keeper.GetOwner(ctx, msg.Name), msg.Bid)
			if err != nil {
				return sdk.ErrInsufficientCoins("Buyer does not have enough coins").Result()
			}
		} else {
			_, err := keeper.CoinKeeper.SubtractCoins(ctx, msg.Buyer, msg.Bid) // If so, deduct the Bid amount from the sender
			if err != nil {
				return sdk.ErrInsufficientCoins("Buyer does not have enough coins").Result()
			}
		}
		keeper.SetOwner(ctx, msg.Name, msg.Buyer)
		keeper.SetPrice(ctx, msg.Name, msg.Bid)
		return sdk.Result{}
	}

**MsgDeleteName:**

	// MsgDeleteName defines a DeleteName message
	type MsgDeleteName struct {
		Name  string         `json:"name"`
		Owner sdk.AccAddress `json:"owner"`
	}

	// NewMsgDeleteName is a constructor function for MsgDeleteName
	func NewMsgDeleteName(name string, owner sdk.AccAddress) MsgDeleteName {
		return MsgDeleteName{
			Name:  name,
			Owner: owner,
		}
	}

	// Route should return the name of the module
	func (msg MsgDeleteName) Route() string { return RouterKey }

	// Type should return the action
	func (msg MsgDeleteName) Type() string { return "delete_name" }

	// ValidateBasic runs stateless checks on the message
	func (msg MsgDeleteName) ValidateBasic() sdk.Error {
		if msg.Owner.Empty() {
			return sdk.ErrInvalidAddress(msg.Owner.String())
		}
		if len(msg.Name) == 0 {
			return sdk.ErrUnknownRequest("Name cannot be empty")
		}
		return nil
	}

	// GetSignBytes encodes the message for signing
	func (msg MsgDeleteName) GetSignBytes() []byte {
		return sdk.MustSortJSON(ModuleCdc.MustMarshalJSON(msg))
	}

	// GetSigners defines whose signature is required
	func (msg MsgDeleteName) GetSigners() []sdk.AccAddress {
		return []sdk.AccAddress{msg.Owner}
	}

**NewHandler:** NewHandler directs messages coming into the namesever to the proper handler.

	// NewHandler returns a handler for "nameservice" type messages.
	func NewHandler(keeper Keeper) sdk.Handler {
		return func(ctx sdk.Context, msg sdk.Msg) sdk.Result {
			switch msg := msg.(type) {
			case MsgSetName:
				return handleMsgSetName(ctx, keeper, msg)
			case MsgBuyName:
				return handleMsgBuyName(ctx, keeper, msg)
			case MsgDeleteName:
				return handleMsgDeleteName(ctx, keeper, msg)
			default:
				errMsg := fmt.Sprintf("Unrecognized nameservice Msg type: %v", msg.Type())
				return sdk.ErrUnknownRequest(errMsg).Result()
			}
		}
	}

	// Handle a message to delete name
	func handleMsgDeleteName(ctx sdk.Context, keeper Keeper, msg MsgDeleteName) sdk.Result {
		if !keeper.IsNamePresent(ctx, msg.Name) {
			return types.ErrNameDoesNotExist(types.DefaultCodespace).Result()
		}
		if !msg.Owner.Equals(keeper.GetOwner(ctx, msg.Name)) {
			return sdk.ErrUnauthorized("Incorrect Owner").Result()
		}
		keeper.DeleteWhois(ctx, msg.Name)
		return sdk.Result{}
	}



	// Handle a message to set name
	func handleMsgSetName(ctx sdk.Context, keeper Keeper, msg MsgSetName) sdk.Result {
		if !msg.Owner.Equals(keeper.GetOwner(ctx, msg.Name)) { // Checks if the the msg sender is the same as the current owner
			return sdk.ErrUnauthorized("Incorrect Owner").Result() // If not, throw an error
		}
		keeper.SetName(ctx, msg.Name, msg.Value) // If so, set the name to the value specified in the msg.
		return sdk.Result{}                      // return
	}

**Msg Interface:**

	package nameservice
	
	import (
		"fmt"
		"github.com/[user]/[repo]/x/nameservice/internal/types"
		sdk "github.com/cosmos/cosmos-sdk/types"
	)
	// Route should return the name of the module
	func (msg MsgSetName) Route() string { return RouterKey }

	// Type should return the action
	func (msg MsgSetName) Type() string { return "set_name" }

## Types
The type *nameservice* is embedded in *baseapp*, which handles all communication with Tendermint Core.
  
	package types
	
	import (
    	"fmt"
    	"strings"	
    	sdk "github.com/cosmos/cosmos-sdk/types"
	)
    
	// MinNamePrice is Initial Starting Price for a name that was never previously owned
    
	var MinNamePrice = sdk.Coins{sdk.NewInt64Coin("nametoken", 1)}
	// Whois is a struct that contains all the metadata of a name
	type Whois struct {
		Value string			`json:"value"`
		Owner sdk.AccAddress	`json:"owner"`
	Price sdk.Coins			`json:"price"`
	}
    
	// NewWhois returns a new Whois with the minprice as the price
	func NewWhois() Whois {
		return Whois{
		Price: MinNamePrice,
    		}
	}
    
	// implement fmt.Stringer
	func (w Whois) String() string {
		return strings.TrimSpace(fmt.Sprintf(`Owner: %s Value: %s Price: %s`, w.Owner, w.Value, w.Price))
	}

**Query Types:**

	package types

	import "strings"

	// QueryResResolve Queries Result Payload for a resolve query
	type QueryResResolve struct {
		Value string `json:"value"`
	}

	// implement fmt.Stringer
	func (r QueryResResolve) String() string {
		return r.Value
	}

	// QueryResNames Queries Result Payload for a names query
	type QueryResNames []string

	// implement fmt.Stringer
	func (n QueryResNames) String() string {
		return strings.Join(n[:], "\n")
	}

### Error
The file error.go handles all application errors. What follows is the portion of this file pertaining to the nameserver.
	package types
    
	import (
		sdk "github.com/cosmos/cosmos-sdk/types"
	)
    
	// DefaultCodespace is the Module Name
	const (
		DefaultCodespace sdk.CodespaceType = ModuleName
    	
		CodeNameDoesNotExist sdk.CodeType = 101
	)
    
	// ErrNameDoesNotExist is the error for name not existing
    func ErrNameDoesNotExist(codespace sdk.CodespaceType) sdk.Error {
	return sdk.NewError(codespace, CodeNameDoesNotExist, "Name does not exist")
	}
## Keeper
The keeper.go file contains all of the Keepers contained within the SDK application. What follows is the nameserver.Keeper.

	package keeper
        
	import(
		"github.com/cosmos/cosmos-sdk/types"
		"github.com/cosmos/sdk-tutorials/nameserver/x/nameserver/types"
	)
	// Keeper maintains the link to data storage and exposes getter/setter methods for the various parts of the state machine
	type Keeper struct {
		storeKey  sdk.StoreKey // Unexposed key to access store from sdk.Context
	}
	
**Keys:**

The nameserver module uses two keys that are stored in the file key.go, which contains all keys used in the sdk.

	package types
    
	const (
	// module name
		ModuleName = "nameserver"
    	
	// StoreKey to be used when creating the KVStore
		StoreKey = ModuleName
	)

**Getters and Setters:**
Getters and Setters add methods to interact with the nameserver KVStore.

	// Sets the entire Whois metadata struct for a name
	func (k Keeper) SetWhois(ctx sdk.Context, name string, whois types.Whois) {
		if whois.Owner.Empty() {
			return
    		}
    	store := ctx.KVStore(k.storeKey)
    	store := ctx.KVStore(k.storeKey)
	}

	// Gets the entire Whois metadata struct for a name
	func (k Keeper) GetWhois(ctx sdk.Context, name string) types.Whois {
	store := ctx.KVStore(k.storeKey)
		if !k.IsNamePresent(ctx, name) {
			return types.NewWhois()
		}
	bz := store.Get([]byte(name))
	var whois types.Whois
	k.cdc.MustUnmarshalBinaryBare(bz, &whois)
		return whois
	}

	// Deletes the entire Whois metadata struct for a name
	func (k Keeper) DeleteWhois(ctx sdk.Context, name string) {
		store := ctx.KVStore(k.storeKey)
		store.Delete([]byte(name))
	}

	// Get an iterator over all names in which the keys are the names and the values are the whois
	func (k Keeper) GetNamesIterator(ctx sdk.Context) sdk.Iterator {
	store := ctx.KVStore(k.storeKey)
		return sdk.KVStorePrefixIterator(store, []byte{})
	}
	// NewKeeper creates new instances of the nameservice Keeper
	func NewKeeper(coinKeeper bank.Keeper, storeKey sdk.StoreKey, cdc *codec.Codec) Keeper {
		return Keeper{
			CoinKeeper: coinKeeper,
			storeKey:   storeKey,
			cdc:        cdc,
		}
	}
**Codec:** Register types so they can be encoded/decoded.

	package types

	import (
		"github.com/cosmos/cosmos-sdk/codec"
	)

	// ModuleCdc is the codec for the module
	var ModuleCdc = codec.New()

	func init() {
		RegisterCodec(ModuleCdc)
	}

	// RegisterCodec registers concrete types on the Amino codec
	func RegisterCodec(cdc *codec.Codec) {
		cdc.RegisterConcrete(MsgSetName{}, "nameservice/SetName", nil)
		cdc.RegisterConcrete(MsgBuyName{}, "nameservice/BuyName", nil)
		cdc.RegisterConcrete(MsgDeleteName{}, "nameservice/DeleteName", nil)
	}

## CLI
The Cosmos SDK uses the golang library Cobra to implement the CLI. The basic structure is *APPNAME Command Args --Flags.

**Queries:**

	package cli

	import (
		"fmt"

		"github.com/cosmos/cosmos-sdk/client"
		"github.com/cosmos/cosmos-sdk/client/context"
		"github.com/cosmos/cosmos-sdk/codec"
		"github.com/cosmos/sdk-tutorials/nameservice/x/nameservice/internal/types"
		"github.com/spf13/cobra"
	)

	func GetQueryCmd(storeKey string, cdc *codec.Codec) *cobra.Command {
		nameserviceQueryCmd := &cobra.Command{
			Use:                        types.ModuleName,
			Short:                      "Querying commands for the nameservice module",
			DisableFlagParsing:         true,
			SuggestionsMinimumDistance: 2,
			RunE:                       client.ValidateCmd,
		}
		nameserviceQueryCmd.AddCommand(client.GetCommands(
			GetCmdResolveName(storeKey, cdc),
			GetCmdWhois(storeKey, cdc),
			GetCmdNames(storeKey, cdc),
		)...)
		return nameserviceQueryCmd
	}

	// GetCmdResolveName queries information about a name
	func GetCmdResolveName(queryRoute string, cdc *codec.Codec) *cobra.Command {
		return &cobra.Command{
			Use:   "resolve [name]",
			Short: "resolve name",
			Args:  cobra.ExactArgs(1),
			RunE: func(cmd *cobra.Command, args []string) error {
				cliCtx := context.NewCLIContext().WithCodec(cdc)
				name := args[0]

				res, _, err := cliCtx.QueryWithData(fmt.Sprintf("custom/%s/resolve/%s", queryRoute, name), nil)
				if err != nil {
					fmt.Printf("could not resolve name - %s \n", name)
					return nil
				}

				var out types.QueryResResolve
				cdc.MustUnmarshalJSON(res, &out)
				return cliCtx.PrintOutput(out)
			},
		}
	}

	// GetCmdWhois queries information about a domain
	func GetCmdWhois(queryRoute string, cdc *codec.Codec) *cobra.Command {
		return &cobra.Command{
			Use:   "whois [name]",
			Short: "Query whois info of name",
			Args:  cobra.ExactArgs(1),
			RunE: func(cmd *cobra.Command, args []string) error {
				cliCtx := context.NewCLIContext().WithCodec(cdc)
				name := args[0]

				res, _, err := cliCtx.QueryWithData(fmt.Sprintf("custom/%s/whois/%s", queryRoute, name), nil)
				if err != nil {
					fmt.Printf("could not resolve whois - %s \n", name)
					return nil
				}

				var out types.Whois
				cdc.MustUnmarshalJSON(res, &out)
				return cliCtx.PrintOutput(out)
			},
		}
	}

	// GetCmdNames queries a list of all names
	func GetCmdNames(queryRoute string, cdc *codec.Codec) *cobra.Command {
		return &cobra.Command{
			Use:   "names",
			Short: "names",
			// Args:  cobra.ExactArgs(1),
			RunE: func(cmd *cobra.Command, args []string) error {
				cliCtx := context.NewCLIContext().WithCodec(cdc)

				res, _, err := cliCtx.QueryWithData(fmt.Sprintf("custom/%s/names", queryRoute), nil)
				if err != nil {
					fmt.Printf("could not get query names\n")
					return nil
				}

				var out types.QueryResNames
				cdc.MustUnmarshalJSON(res, &out)
				return cliCtx.PrintOutput(out)
			},
		}
	}

**Transactions:**

	package cli

	import (
		"github.com/spf13/cobra"

		"github.com/cosmos/cosmos-sdk/client"
		"github.com/cosmos/cosmos-sdk/client/context"
		"github.com/cosmos/cosmos-sdk/codec"
		sdk "github.com/cosmos/cosmos-sdk/types"
		"github.com/cosmos/cosmos-sdk/x/auth"
		"github.com/cosmos/cosmos-sdk/x/auth/client/utils"
		"github.com/cosmos/sdk-tutorials/nameservice/x/nameservice/internal/types"
	)

	func GetTxCmd(storeKey string, cdc *codec.Codec) *cobra.Command {
		nameserviceTxCmd := &cobra.Command{
			Use:                        types.ModuleName,
			Short:                      "Nameservice transaction subcommands",
			DisableFlagParsing:         true,
			SuggestionsMinimumDistance: 2,
			RunE:                       client.ValidateCmd,
		}

		nameserviceTxCmd.AddCommand(client.PostCommands(
			GetCmdBuyName(cdc),
			GetCmdSetName(cdc),
			GetCmdDeleteName(cdc),
		)...)

		return nameserviceTxCmd
	}

	// GetCmdBuyName is the CLI command for sending a BuyName transaction
	func GetCmdBuyName(cdc *codec.Codec) *cobra.Command {
		return &cobra.Command{
			Use:   "buy-name [name] [amount]",
			Short: "bid for existing name or claim new name",
			Args:  cobra.ExactArgs(2),
			RunE: func(cmd *cobra.Command, args []string) error {
				cliCtx := context.NewCLIContext().WithCodec(cdc)

				txBldr := auth.NewTxBuilderFromCLI().WithTxEncoder(utils.GetTxEncoder(cdc))

				coins, err := sdk.ParseCoins(args[1])
				if err != nil {
					return err
				}

				msg := types.NewMsgBuyName(args[0], coins, cliCtx.GetFromAddress())
				err = msg.ValidateBasic()
				if err != nil {
					return err
				}

				return utils.GenerateOrBroadcastMsgs(cliCtx, txBldr, []sdk.Msg{msg})
			},
		}
	}

	// GetCmdSetName is the CLI command for sending a SetName transaction
	func GetCmdSetName(cdc *codec.Codec) *cobra.Command {
		return &cobra.Command{
			Use:   "set-name [name] [value]",
			Short: "set the value associated with a name that you own",
			Args:  cobra.ExactArgs(2),
			RunE: func(cmd *cobra.Command, args []string) error {
				cliCtx := context.NewCLIContext().WithCodec(cdc)

				txBldr := auth.NewTxBuilderFromCLI().WithTxEncoder(utils.GetTxEncoder(cdc))

				msg := types.NewMsgSetName(args[0], args[1], cliCtx.GetFromAddress())
				err := msg.ValidateBasic()
				if err != nil {
					return err
				}

				return utils.GenerateOrBroadcastMsgs(cliCtx, txBldr, []sdk.Msg{msg})
			},
		}
	}

	// GetCmdDeleteName is the CLI command for sending a DeleteName transaction
	func GetCmdDeleteName(cdc *codec.Codec) *cobra.Command {
		return &cobra.Command{
			Use:   "delete-name [name]",
			Short: "delete the name that you own along with it's associated fields",
			Args:  cobra.ExactArgs(1),
			RunE: func(cmd *cobra.Command, args []string) error {
				cliCtx := context.NewCLIContext().WithCodec(cdc)

				txBldr := auth.NewTxBuilderFromCLI().WithTxEncoder(utils.GetTxEncoder(cdc))

				msg := types.NewMsgDeleteName(args[0], cliCtx.GetFromAddress())
				err := msg.ValidateBasic()
				if err != nil {
					return err
				}
				return utils.GenerateOrBroadcastMsgs(cliCtx, txBldr, []sdk.Msg{msg})
			},
		}
	}

### Commands
The Cosmos SDK adopts the naming convention that maps APPNAME onto an app name abreviation with cli appended. For example, an application named namerserver maps to nscli. For the list of commands specific to the nameserver module the app name abreviation is *.

**Queries:**

	*cli query account $(*cli keys show *Owner* -a)
	
	*cli query nameservice resolve *Owner*.id
	
	*cli query nameservice whois *Owner*.id
	
**Transactions:**
	
	*cli tx nameservice buy-name *name*.id 5nametoken --from seller
	
	# Set the value for the purchased name
	*cli tx nameservice set-name *name*.id 8.8.8.8 --from *Owner*

	#  buys name from *Owner*
	*cli tx nameservice buy-name *Owner*.id 10nametoken --from Buyer

	# Delete Name
	*cli tx nameservice delete-name *name*.id --from alice

	


