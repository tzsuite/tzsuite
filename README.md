# tzsuite

tzsuite is a Go client library for the Tezos RPC API. It offers rich RPC interactions, such as automated transaction batching, operation receipts or alerts for specific addresses. It also contains packages for key manipulation and signing.

## Usage
Send transactions with no hassle. They will be organized into batches automatically. You'll get notified when the transfer is transmitted to the network, and when it gets included into a block.
```go
// error handling not shown
// seed for user1keys supplied e.g. by command line argument
user1keys := tzutil.KeysFromStringSeed(*seed)
user1 := "tz1f8zZNaJdiiQepZwHJGKVnna6X9sWPgW7P"
user2 := "tz1aTkCpuboKhLEmmetQ52dmpPomwQWKZPFv"
done, err := r.BatchTransfer(user1keys, 123000, user1, user2, nil)
opHash := <-done
fmt.Println("transfer injected in operation:", opHash)
blockHash := <-done
fmt.Println("transfer included in block:", blockHash)
```

You can dispatch them as they come, without worrying about timing or batch size limits.
```go
for {
    done, err := r.BatchTransfer(user1keys, 123000, user1, user2, nil)
    go func(done <-chan string) {
        opHash := <-done
        fmt.Println("transfer injected in operation:", opHash)
        blockHash := <-done
        fmt.Println("transfer included in block:", blockHash)
    }(done)
    
    time.Sleep(time.Second * 5)
}
```

Transactions to smart contracts.
```go
// prepare contract parameters
params := fmt.Sprintf(`{"prim":"Pair","args":[{"string":"%s"},{"int":"%d"}]}`, "tz1Ph8mdwaRp71XvixaExcNKtPvQshe5BwcR", 123456)

done, err := r.BatchTransfer(user1keys, 123000, user1, user2, params)
opHash := <-done
fmt.Println("transfer injected in operation:", opHash)
blockHash := <-done
fmt.Println("transfer included in block:", blockHash)
```

Publish (originate) contracts programatically. You'll get the address of the newly created contract over a channel, to use in further transactions and other parts of your program.
```go
// arguments mirror the tezos-client originate command
// keys of the address used to pay for origination, then manager of the new contract, starting balance,
// address that pays for the origination, then script (code + initial storage) of the new contract,
// spendable and delegatable flags, and optionally the delegate address
done, published, err := r.BatchOriginate(user1keys, manager, 0, user1, json.RawMessage(myScript), false, false, "")
included := make(chan string)
go func(done <-chan string, included chan<- string) {
    opHash := <-done
    fmt.Println("origination injected in operation:", opHash)
    blockHash := <-done
    fmt.Println("origination included in block:", blockHash)
    included <- blockHash
}(done, included)
newContract := <-published
// wait until the operation makes it into a block
<-included
fmt.Println("new contract address:", newContract)
```

Get notified about operations involving an address.
```go
activity, unsub := r.WatchAddress(user1)
timer := time.NewTimer(time.Minute * 2)
FOR:
for {
    select {
    case _ = <-timer.C:
        break FOR
    case a := <-activity:
        fmt.Printf("Detected address activity in block: %s\noperation: %#v\n", a.BlockHash, a.Operation)
    }
}

fmt.Println("Unsubscribing.")
close(unsub)
```

Expecting a deposit? You will be notified through a channel when it arrives.
```go
paid := r.ExpectBalance(123465, user1)

// block until deposit is received
blockHash := <-paid
```

## What's r?

It's a struct that holds state related to RPC interaction with Tezos node.
The features shown above are its methods.

```go
// use a config preset for alphanet
// we can specify a remote node, or use empty string for localhost:8732

ch := make(chan error)
shutdown := make(chan struct{})

// new block events, if we want to receive them
nbe := make(chan string)

r := rpc.New(rpc.ConfAlphanet(""), nbe, shutdown, ch)
go func(ch chan error, shutdown chan struct{}) {
    for {
        select {
        case _, more := <-shutdown:
            if !more {
                return
            }
        case err := <-ch:
            fmt.Println("error from goroutine:", err)
            close(shutdown)
        }
    }

}(ch, shutdown)
```

## Licensing

Contact us at hello@smartcontractlabs.ee for commercial licensing options. We offer consulting and integration with your existing systems as well.
