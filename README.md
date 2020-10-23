# nRPC

[![Build Status](https://travis-ci.org/nats-rpc/nrpc.svg?branch=master)](https://travis-ci.org/nats-rpc/nrpc)

nRPC is an RPC framework like [gRPC](https://grpc.io/), but for
[NATS](https://nats.io/).

It can generate a Go client and server from the same .proto file that you'd
use to generate gRPC clients and servers. The server is generated as a NATS
[MsgHandler](https://godoc.org/github.com/nats-io/nats.go#MsgHandler).

## Why NATS?

Doing RPC over NATS'
[request-response model](http://nats.io/documentation/concepts/nats-req-rep/)
has some advantages over a gRPC model:

- **Minimal service discovery**: The clients and servers only need to know the
  endpoints of a NATS cluster. The clients do not need to discover the
  endpoints of individual services they depend on.
- **Load balancing without load balancers**: Stateless microservices can be
  hosted redundantly and connected to the same NATS cluster. The incoming
  requests can then be random-routed among these using NATS
  [queueing](http://nats.io/documentation/concepts/nats-queueing/). There is
  no need to setup a (high availability) load balancer per microservice.

The lunch is not always free, however. At scale, the NATS cluster itself can
become a bottleneck. Features of gRPC like streaming and advanced auth are not
available.

Still, NATS - and nRPC - offer much lower operational complexity if your
scale and requirements fit.

At RapidLoop, we use this model for our [OpsDash](https://www.opsdash.com)
SaaS product in production and are quite happy with it. nRPC is the third
iteration of an internal library.

## Overview

nRPC comes with a protobuf compiler plugin `protoc-gen-nrpc`, which generates
Go code from a .proto file.

Given a .proto file like [helloworld.proto](https://github.com/grpc/grpc-go/blob/master/examples/helloworld/helloworld/helloworld.proto), the usage is like this:

```
$ ls
helloworld.proto
$ protoc --go_out=. --nrpc_out=. helloworld.proto
$ ls
helloworld.nrpc.go	helloworld.pb.go	helloworld.proto
```

The .pb.go file, which contains the definitions for the message classes, is
generated by the standard Go plugin for protoc. The .nrpc.go file, which
contains the definitions for a client, a server interface, and a NATS handler
is generated by the nRPC plugin.

Have a look at the generated and example files:

- the service definition [helloworld.proto](https://github.com/truyet/nrpc/tree/master/examples/helloworld/helloworld/helloworld.proto)
- the generated nrpc go file [helloworld.nrpc.go](https://github.com/truyet/nrpc/tree/master/examples/helloworld/helloworld/helloworld.nrpc.go)
- an example server [greeter_server/main.go](https://github.com/truyet/nrpc/tree/master/examples/helloworld/greeter_server/main.go)
- an example client [greeter_client/main.go](https://github.com/truyet/nrpc/tree/master/examples/helloworld/greeter_client/main.go)

### How It Works

The .proto file defines messages (like HelloRequest and HelloReply in the
example) and services (Greeter) that have methods (SayHello).

The messages are generated as Go structs by the regular Go protobuf compiler
plugin and gets written out to \*.pb.go files.

For the rest, nRPC generates three logical pieces.

The first is a Go interface type (GreeterServer) which your actual
microservice code should implement:

```
// This is what is contained in the .proto file
service Greeter {
    rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// This is the generated interface which you've to implement
type GreeterServer interface {
    SayHello(ctx context.Context, req HelloRequest) (resp HelloReply, err error)
}
```

The second is a client (GreeterClient struct). This struct has
methods with appropriate types, that correspond to the service definition. The
client code will marshal and wrap the request object (HelloRequest) and do a
NATS `Request`.

```
// The client is associated with a NATS connection.
func NewGreeterClient(nc *nats.Conn) *GreeterClient {...}

// And has properly typed methods that will marshal and perform a NATS request.
func (c *GreeterClient) SayHello(req HelloRequest) (resp HelloReply, err error) {...}
```

The third and final piece is the handler (GreeterHandler). Given a NATS
connection and a server implementation, it can accept NATS requests in the
format sent by the client above. It should be installed as a message handler for
a particular NATS subject (defaults to the name of the service) using the
NATS Subscribe() or QueueSubscribe() methods. It will invoke the appropriate
method of the GreeterServer interface upon receiving the appropriate request.

```
// A handler is associated with a NATS connection and a server implementation.
func NewGreeterHandler(ctx context.Context, nc *nats.Conn, s GreeterServer) *GreeterHandler {...}

// It has a method that can (should) be used as a NATS message handler.
func (h *GreeterHandler) Handler(msg *nats.Msg) {...}
```

Standing up a microservice involves:

- writing the .proto service definition file
- generating the \*.pb.go and \*.nrpc.go files
- implementing the server interface
- writing a main app that will connect to NATS and start the handler ([see
  example](https://github.com/truyet/nrpc/blob/master/examples/helloworld/greeter_server/main.go))

To call the service:

- import the package that contains the generated *.nrpc.go files
- in the client code, connect to NATS
- create a Caller object and call the methods as necessary ([see example](https://github.com/truyet/nrpc/blob/master/examples/helloworld/greeter_client/main.go))

## Features

The following wiki pages describe nRPC features in more detail:

- [Load Balancing](https://github.com/truyet/nrpc/wiki/Load-Balancing)
- [Metrics Instrumentation](https://github.com/truyet/nrpc/wiki/Metrics-Instrumentation)
  using Prometheus

## Installation

nRPC needs Go 1.7 or higher. $GOPATH/bin needs to be in $PATH for the protoc
invocation to work. To generate code, you need the protobuf compiler (which
you can install from [here](https://github.com/google/protobuf/releases))
and the nRPC protoc plugin.

To install the nRPC protoc plugin:

```
$ go get github.com/truyet/nrpc/protoc-gen-nrpc
```

To build and run the example greeter_server:

```
$ go get github.com/truyet/nrpc/examples/helloworld/greeter_server
$ greeter_server
server is running, ^C quits.
```

To build and run the example greeter_client:

```
$ go get github.com/truyet/nrpc/examples/helloworld/greeter_client
$ greeter_client
Greeting: Hello world
$
```

## Documentation

To learn more about describing gRPC services using .proto files, see [here](https://grpc.io/docs/guides/concepts.html).
To learn more about NATS, start with their [website](https://nats.io/). To
learn more about nRPC, um, read the source code.

## Status

nRPC is in alpha. This means that it will work, but APIs may change without
notice.

Currently there is support only for Go clients and servers.

Built by RapidLoop. Released under Apache 2.0 license.
