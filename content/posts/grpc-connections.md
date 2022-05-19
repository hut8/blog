+++
date = "2022-05-18T17:00:45-04:00"
draft = false
title = "Symmetric and Reverse gRPC Connections"
tags = [ "grpc", "go", "networking", "ssh", "rpc" ]
icon = "grpc.png"
+++

## Background

gRPC is a great, modern technology to do Remote Procedure Calls. It
allows you to create a "stub" object on a client, which is an object
whose purpose is to invoke methods on a server. It's a fantastic
alternative to many situations where REST or GraphQL would be used,
and generally is a worthwhile technology to learn. More information
can be found [on the official docs](https://grpc.io/docs/).

## Networking

Conceptually we have two different ideas of clients and servers.

* **TCP Server and Client** -  gRPC operates on top of HTTP/2, which
  runs on top of TCP, so I will discuss what "server" and "client"
  mean in TCP. On a TCP connection, the client is the
  *originator of the connection* and the server is the
  *recipient of the connection*. Once the connection is established,
  however, the connection is symmetrical; the client and server can
  both send and receive messages until one side shuts down via
  `shutdown(2)`, or close the connection via `close(2)`.

* **gRPC Server and Client** - the gRPC Client invokes methods, via
  stubs, that run on the Server. This is not symmetrical. The server
  cannot invoke methods on the client.



## Problems with coupling between server/client types

If you have a gRPC client, it's also the TCP client (it calls
`connect(2)`). If you have a gRPC server, it's also the TCP server (it
calls `listen(2)` and `accept(2)`))

Therefore, if you want to have two machines which can invoke gRPC
calls on one another, then each machine is both a client and server
(in both senses: TCP and gRPC) and now you have two TCP connections
which are unrelated to each other. In a scenario where you have two
cloud instances (to avoid overloading the use of the word "server"),
it's often not complicated to have such an architecture. But TCP is
often messy. Firewalls, NAT and dynamically assigned IP addresses can
make it complicated to make a client → server connection in one
direction.

Here's an example of such a case: Suppose your "server" is a program
running on a laptop behind a home router that must receive commands
(for example, to be controlled remotely) from a "client" (for example,
to be controlled remotely), which is a cloud computing
instance. According to gRPC, the client must be the cloud instance
(which is tempting to call a "server") because that is the side that
creates the remote procedure calls. The server is the laptop, because
that's where the procedure calls actually occur.

When you create a "stub" on the client (which is the cloud instance),
you must create a TCP connection and connect to the server (which is
the laptop). Now you encounter several possible problems:

* You don't know the laptop's IP address, so you now need a separate
  service in the opposite direction for the laptop to tell the cloud
  what its IP address is.
* The laptop is likely behind NAT, so once you find the IP address, the
  laptop will have to configure port forwarding. This is likely
  impossible, especially in a corporate environment.
* The laptop is likely behind a firewall which will forbid incoming
  connections. This may be local to the machine, or on the gateway
  router.

Some of these problems are potentially intractable, so we need another
approach.

## gRPC-only solution

A server cannot invoke methods on a client gRPC. Perhaps you could
have a client invoke a method on the server whose only purpose is to
receive messages that describe methods that the server would like the
client to run. This could be done with a streaming response. But that
is complex and requires a large amount of code to work around the
design of gRPC. I've seen it suggested elsewhere, but I think it's an
ugly kludge that creates more problems than it solves. Consider, for
example, how you would actually invoke these "client methods" in a
statically typed way.

## Tunnel-based solution
