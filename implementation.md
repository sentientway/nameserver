# Implementation
## Introduction
## Concept
The nameserver module maps a name to an owner, a resolution value, and name price. The module transfers names between users, deletes names, and creates new names that satisfy the minimum name price. The minimum price is set elsewhere in the SDK application.
## State
The state of this module is stored in a single key/value store (KVStore), which is added to a single multistore. The triggers for state change are:

1. Creating a new name.
2. Removing an existing name.
3. Transfer of ownership.
## Messages
Messages trigger state transitions. Messages are passed to *Handlers* who then passes the action to the appropriate *Keeper*, which updates the *State*.

The separation of message processing between the module's handler and keeper enhances security. Each keeper, which updates *State*, is specialized to a *type*, whereas, depending on the module, a handler can be specified several *types*.
### Nameserver Messages
Messages are contained within transactions, which trigger state changes. The nameserver module uses:
- MsgSetName
- MsgBuyName
- MsgDelName
### Nameserver Msg Type
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
### Nameserver Handlers
The nameserver uses three Handlers, one for each nameserver message type, SetName, BuyName, and DeleteName.

- MsgSetName

		package types

		import (
			sdk "github.com/cosmos/cosmos-sdk/types"
		)

		const RouterKey = ModuleName // this was defined in your key.go file

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

- MsgBuyName

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
		func handleMsgBuyName(ctx sdk.Context, keeper Keeper, 		msg MsgBuyName) sdk.Result {
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

- MsgDeleteName

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


Msg Interface

	// Route should return the name of the module
	func (msg MsgSetName) Route() string { return RouterKey }

	// Type should return the action
	func (msg MsgSetName) Type() string { return "set_name" }

	package nameservice

	import (
	"fmt"
	"github.com/[user]/[repo]/x/nameservice/internal/types"
	sdk "github.com/cosmos/cosmos-sdk/types"
	)

	// NewHandler returns a handler for "nameservice" type messages.
	func NewHandler(keeper Keeper) sdk.Handler {
		return func(ctx sdk.Context, msg sdk.Msg) sdk.Result {
			switch msg := msg.(type) {
			case MsgSetName:
				return handleMsgSetName(ctx, keeper, msg)
			default:
				errMsg := fmt.Sprintf("Unrecognized nameservice Msg type: %v", msg.Type())
				return sdk.ErrUnknownRequest(errMsg).Result()
			}
		}
	}
NewHandler directs messages coming into the namesever to the proper handler.

	// Handle a message to set name
	func handleMsgSetName(ctx sdk.Context, keeper Keeper, msg MsgSetName) sdk.Result {
		if !msg.Owner.Equals(keeper.GetOwner(ctx, msg.Name)) { // Checks if the the msg sender is the same as the current owner
			return sdk.ErrUnauthorized("Incorrect Owner").Result() // If not, throw an error
		}
		keeper.SetName(ctx, msg.Name, msg.Value) // If so, set the name to the value specified in the msg.
		return sdk.Result{}                      // return
	}

### Nameserver Type
The type ==nameserver== is embedded in ==baseapp==, which handles all communication with Tendermint Core.

Adding the nameserver type:
    
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

The nameserver module uses two keys that are stored in a file key.go, which contains all keys used in the sdk.

    package types
    
    const (
    	// module name
    	ModuleName = "nameserver"
    	
    	// StoreKey to be used when creating the KVStore
    	StoreKey = ModuleName
    )
### Nameserver Error
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
### Nameserver Keeper
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
### Getters and Setters
Getters and Setters add methods to interact with the nameserver KVStore.

    // Sets the entire Whois metadata struct for a name
    func (k Keeper) SetWhois(ctx sdk.Context, name string, whois types.Whois) {
    	if whois.Owner.Empty() {
    		return
    	}
    store := ctx.KVStore(k.storeKey
    store := ctx.KVStore(k.storeKey
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

## Parameters




