# Ethereum NACL (NAtive CLient) development, part 1.

Ethereum HTML/JS & QML DApps are pretty powerful and let you create a DApp in only a couple of minutes, however, you don't have to stop there. Both the **ethereal** and **ethereum** clients are written using the [eth-go](https://github.com/ethereum/eth-go) package. It has been the intention that this package will be used for any NACL Decentralised Application Development using the go programming language. With this in mind I've tried to minimise the complexity of setting up an ethereum node. Setting up a node only requires a couple lines of code. 

Here's an example which demonstrates using the `eth-go` package and setting up an ethereum node:

```go
// 1. We start by initialising ethereum's Config object.
//      a. Configuration file
//      b. Data directory for storing database objects
//      c. Prefix
ethutil.ReadConfig("~/.config/coin.ini", "~/.coin", "COIN")

// 2. Create a logger
//      a. Default log stream
//      b. Log flags (see: go doc log)
//      c. The amount of log you're interested in
//          i.   SilenceLevel (0)
//          ii.  ErrorLevel (1)
//          iii. WarnLevel (2)
//          iv.  InfoLevel (3)
//          v.   DebugLevel (4)
//          vi.  DebugDetail (5) Please note: This includes network traffic as well.
lSys := ethlog.NewStdLogSystem(os.Stdout, log.LstdFlags, ethlog.LogLevel(LogLevel))

// 3. Add the logger to the default log system (multiple loggers allowed)
ethlog.AddLogSystem(lSys)

// 4. Create a new database which will be used to store the block chain
db, err ;= ethdb.NewLDBDatabase("my_database")
if err != nil {
    panic(err)
}

// 4. Create a key manager which will manage the private & public keys.
keyManager := ethcrypto.NewDBKeyManager(db)
err = keyManager.Init("", 0, false)
if err != nil {
    panic(err)
}

// 5. Create a new client identity. Client identities are used to identify this particular node in the network
//      a. Client name
//      b. Client version
//      c. Optional identifier
clientIdentity := ethwire.NewSimpleClientIdentity("NACL Coin", "0.1", "")

// 6. Create a new Ethereum manager. The manager will take care of peer handling and all incoming and outgoing traffic,
//    block & tx propagation and block processing.
//      a. The database for storing the block chain
//      b. Client identity
//      c. Key manager
//      d. Client capacity
//          i.   CapPeerDiscTy - Peer discovery or peer forwarding
//          ii.  CapTxTy - Transaction relaying
//          iii. CapChainTy - Block chain relaying
//          iv.  CapDefault - All of the above
//      e. Boolean indicating whether you want to use UPnP or not
ethereum, err := eth.New(db, clientIdentity, keyManager, eth.CapDefault, false)
if err != nil {
    panic(err)
}

// 7. Start up the node
        a. Boolean indicating whether you want to use the peer server to find peers.
ethereum.Start(true)

// 8. Wait for ethereum to shut down. In most cases you'd register an interrupt handler (outside of ethereum)
//    and signal ethereum you want to shutdown using "ethereum.Stop()"
ethereum.WaitForShutDown()

// 9. Flush out remaining log messages
ethlog.Flush()
```

As you can see in the example above it's fairly easy to set up a full node. From here on out it's only a matter of querying ethereum to retrieve the information you require.

## The State of Ethereum

Before we continue you need to understand how ethereum stores data and how it handles past and present data. Ethereum is always in a constant state of change. If data changes in the ethereum network, ethereum's state has changed. State object (which together make up the state) are stored in a [modified patricia trie](). The state changes when an object is retrieved, changed and stored. Without going in to too much detail assume for now that changing data, changes the state of ethereum.

Retrieving and storing data can be done through a simple **key / value** interface:

```go
// Get the state object
stateObject := state.GetStateObject([]byte("key_of_the_object"))
// Change a key in the object's storage
stateObject.SetStorage(k, v)
// Update the state
state.Update()
```

State objects (i.e., accounts & contracts) are usually retrieved and stored by their address, therefor if you want to get information about an object you'd need to know the address of the object. If your account's address was `e6716f9544a56c530d868e4bfbacb172315bdead` we could retrieve like this:

```go
const address = "e6716f9544a56c530d868e4bfbacb172315bdead"
// Convert hex to byte slice
addr := ethutil.Hex2Bytes(address)
// Retrieve account
account := state.GetStateObject(addr)
fmt.Println("Balance:", account.Balance)
```

Each state object, and this is especially useful for contracts, have storage. Storage has it's own state and can be changed through another **key / value** interface. Lets create a new object in our current state and set/get some values.

```go
// Create a new state object. State objects created through the state are automatically cached and tracked.
stateObject := state.NewStateObject([]byte("my address"))
// Set the new value
account.SetStorage(key, ethutil.NewValue(1))
// Retrieve a value
value := account.GetStorage(key)
fmt.Println(value)
```

Iterating over a state object's storage can be done using the `EachStorage` method. Note that values stored in a contract are encoded and have to manually decoded by calling the `Decode()` method.

```go
stateObject.EachStorage(func(key string, value *ethutil.Value) {
    // Decode the value
    value.Decode()
    fmt.Println(value.Bytes())
})
```

Ethereum's is in a constant state of being cached. If an object is retrieved and changed or if a new object has been created on the state it doesn't mean it's automatically applied to the current state. This has a few reasons and the biggest being optimisations. Update the state simply by calling the `Update()` method. There's one more caveat: once applied it's not **saved**. In order to limit the amount of disk i/o, the `trie` doesn't apply to database when it changes, it first commits all it changes to a local `map[string]string`. If you want to make your changes permanent you need to call the `Sync()` method on the state.

```go
// Change the balance on the account
account.Balance = big.NewInt(1000)
// Update the state
state.Update()
// Sync trie with the database
state.Sync()
```

It's also possible to create a snapshot of the current state to which you can revert to at any given time. This is handy for situations where you're uncertain about a change. Ethereum uses this internally incase a VM throws up or an invalid transaction is being send.

```go
stateObject := state.GetStateObject(addr)
// Creates a full copy of the state
snapshot := state.Copy()
// Change the balance on the account
account.Balance = big.NewInt(1000)
// Unhappy about the change we've just made :-(
state.Set(snapshot)
```

We've now covered the basics of setting up an ethereum stack. In the next instalment of NACL Development we'll cover the basics of creating an Ethereum wallet which will cover basic simple sending of transactions and balance checks. 

## Structure

What follows now is a brief summary of the sub-packages included in the eth-go package. Most of the packages are irrelevant for NACL development. They are listed here merely for informational purpose.

#### ethutil

The `ethutil` package contains ethereum's utility functions such as byte manipulation, big integer additions, serialisation and an additional dynamic `Value` object which can represent any native type (including structs) and has the ability to cast to the a requested types:

```go
a := NewValue(1)               // Value represents an integer
fmt.Println(a.BigInt())        // Cast to big.Int and print it.
// Mix ints and big ints
b := NewValue(big.NewInt(10))  // Value represents a big.Int
fmt.Println(a.Add(b))          // Prints out 11
```

#### ethwire

The `ethwire` package contains low level communication to the ethereum network.

#### ethdb

The `ethdb` package contains two database backends which conform to the `ethutil.Db` interface.

1. LevelDB
2. MemoryDB

#### ethtrie

The `ethtrie` package contains ethereum's *modified patricia trie* which is being used to store ethereum's `state`. The trie is used to derive the current state root of the ethereum network.

```go
// "db" is one of the database in ethdb. Second argument is the root (empty for new tries)
trie := ethtrie.NewTrie(db, "")
trie.Update("eth", "blockchain")
trie.Update("smart contract", "mutan")
```

#### ethstate

The `ethstate` package contains ethereum's state object. State objects are stored in the `ethtrie`. A state object is a contract with a certain state (storage).

```go
// "trie" is an object from the ethtrie package
state := ethstate.NewState(trie)
// Retrieve a state object
stateObject := state.GetStateObject(objAddr)
// XXX big.Int as key will change to bytes.
stateObject.SetStorage(key, value)
storage := stateObject.GetStorage(key)
```

#### ethvm

The `ethvm` package contains a Virtual Machine. The VM is used by ethereum to run and validate contracts. The VM can be used completely standalone from ethereum.

```go
// *env* is an interface that is required by the VM. Please see the code for it's implementation
vm := ethvm.New(env)
closure := ethvm.NewClosure(initiator, stateObject, code, big.NewInt(1000000), big.NewInt(0))
closure.Call(vm, data)
```

#### ethrpc

The `ethrpc` package contains RPC interfaces for setting up a RPC server between RPC and the outside world.

#### ethlog

The `ethlog` package contains the general logger

#### ethcrypto

The `ethcrypto` package contains ethereum's key manager and crypto functions.

#### ethchain

The `ethchain` contains the actual consensus engine that drives ethereum.
