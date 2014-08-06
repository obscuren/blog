# Ethereum Pipe API

While working on the native [JeffCoin client](https://github.com/obscuren/jeffcoin) I noticed that interfacing with Ethereum isn't always as trivial as I initially thought, especially using the `ethpub` API which really started out as an interface for use with JS/QML (and specificly textual arguments).

I set out and decided `eth-go` could use a new, better, easier to use API for interfacing with Ethereum's **state** and **objects**. `ethpipe` was the result of this and this package should make Ethereum **NACL** development a whole lot easier. 

## Creating a new Pipe

Creating a new `Pipe` can be done through the `ethpipe.New(ethereum)` function. It initialises a new `*Pipe` object through which you have full access to the state of Ethereum. This object will function as a proxy between your code and Ethereum's `World` object.

```go
import (
    "github.com/ethereum/eth-go/ethutil"
    "github.com/ethereum/eth-go/ethpipe"
)

func main() {
    var MyAddress = ethutil.Hex2Bytes("e6716f9544a56c530d868e4bfbacb172315bdead")
    /*
     * initialise ethereum stack (see previous post in NACL dev)
     */
    ethereum := ...
     
    pipe := ethpipe.New(ethereum)
    
    // Check if object exists in Ethereum
    fmt.Printnf("%x exists: %v\n", MyAddress, pipe.Exists(MyAddress))
    
    // Get the balance of an address
    fmt.Println("balance", pipe.Balance(MyAddress).BigInt())
}
```

Most of the methods of the `Pipe` object are actually wrappers for the `World` object. This object is the current state in which it operates. The world holds information such as whether it's mining and if it's currently in `listening` mode and accepting connections, as well as query addresses directly and retrieve the associated `Object`. It has one handy method and object in particular; `Config`. The `Config` object is an easy to use wrapper for the contract that resides at `661005d2720d855f1d9976f88bb10c1a3398c77f`. This is Ethereum's config contract which is a simple to use **key value** contract holding the addresses to contracts such as **NameReg**.

```go
// Following up on the above example

cfg := pipe.World().Config()
// Check if the object actually exists
if cfg.Exist() {
    // Retrieve namereg object
    nameReg := cfg.Get("NameReg")
    
    fmt.Printf("Jeff's Address is: %x\n", nameReg.GetString("Jeff"))
}
```

`Object` has a few handy methods for retrieving contract storage values. For example the `GetString` method will make sure that the given string is right padded with `0`'s (mutan, LLL and serpent all right pad strings with zero's), or `StorageValue` and `Storage` which will both left-pad with zero's. `Object` is actually just a `ethstate.StateObject` and has one as an embedded type.

## Execution

Last but certainly not least is the ability to preview/run a contract and evaluate its return value. `Pipe` has `Execute` and `ExecuteObject` which will take a copy of the current state and executes the given object (or address). This is especially handy if you'd like to preview a contract and it's return value.

```go
var JeffCoinAddr = ethutil.Hex2Bytes("2b984e578b430612e38d19014cff841ba39cdb9a")

// Execute the object associated with the address. Method ExecuteObject takes an object as first argument instead.
ret, err := pipe.Execute(JeffCoinAddr, nil, ethutil.NewValue(0), ethutil.NewValue(10000), ethutil.NewValue(0))
if err != nil {
    fmt.Println("err", err)
}

fmt.Printf("return value %x\n", ret)
```

If you don't want a copy of the current state, but would like to supply your own state instead you could do the following.

```go
pipe.Vm.State = state
pipe.Execute( ... )
```

I hope this will further speed up the NACL development. Suggestion, patches, etc are all welcome on GH :-)

For some further reading please see the [Pipe Reference](https://github.com/ethereum/go-ethereum/wiki/Pipe)
