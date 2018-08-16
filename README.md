# tzsuite

Suite of tools for working with Tezos in Golang. Includes packages for key manipulation, signing and rich RPC interactions, such as automated transaction batching.

## Usage
Send transactions with no hassle. They will be organized into batches automatically.
```go
// user1keys not shown ...
user1 := "tz1f8zZNaJdiiQepZwHJGKVnna6X9sWPgW7P"
user2 := "tz1aTkCpuboKhLEmmetQ52dmpPomwQWKZPFv"
err := r.BatchTransfer(123000, user1keys, nil, user1, user2)
if err != nil {
	log.Fatal(err)
}
```

You can dispatch them as they come, without worrying about timing or batch size limits.
```go
for {
	err := r.BatchTransfer(123000, user1keys, nil, user1, user2)
	if err != nil {
		log.Fatal(err)
	}
	
	time.Sleep(time.Second * 5)
}
```

Transactions to smart contracts.
```go
// prepare contract parameters
params := fmt.Sprintf(`{"prim":"Pair","args":[{"string":"%s"},{"int":"%d"}]}`, "tz1Ph8mdwaRp71XvixaExcNKtPvQshe5BwcR", 123456)

err := r.BatchTransfer(123000, user1keys, params, user1, user2)
if err != nil {
	log.Fatal(err)
}
```

Expecting a deposit? You will be notified through a channel when it arrives.
```go
paid := r.ExpectBalance(123465, "tz1Ph8mdwaRp71XvixaExcNKtPvQshe5BwcR")

// block until deposit is received
blockHash := <-paid
```

## What's r?

It's a struct that holds state related to RPC interaction with Tezos node.
The features shown above are its methods.

```go
// config
const fee int64 = 50000

debug := true
verbose := true

ch := make(chan error)
shutdown := make(chan struct{})

// new block events, if we want to receive them
nbe := make(chan string)

r := rpc.New("", fee, nbe, shutdown, ch, debug, verbose)
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
