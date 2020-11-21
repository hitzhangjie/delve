# dlv

`dlv` provides a command line user interface to debug. 

We mentioned the delve architecture before in architecture.md or service-layer.md, 
the service layer plays an important role in debugging. `dlv`, this command, it 
sends RPC requests to debug service to issue the debugging actions.

when `dlv` executed, it will do following actions:
- test the value `main.Build` written by `go build -ldflags="-X main.Build=${build}"`, 
  and update the `DelveVersion` if it's not empty. 
- initialize a `cobra.Command{}`, which holds the `dlv command` and its 
  `children commands`, including their full, inherited, private flagset.
- then `(*cobra.Command).Execute()`, to prompt user input and process.

`dlv` provides supports following operations, they're defined in file `cmd/dlv/cmds/commands.go:New()`:
- attach, 
- connect, connect to debug service, initialize terminal to prompt input to debug.
- 


