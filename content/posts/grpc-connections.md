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

A great solution for decoupling the TCP client/server from the gRPC
client/server is to implement some sort of tunnel. In the above
example with a laptop and a cloud instance, the laptop can make a
single TCP connection to the cloud instance, and then by implementing
a language-specific interface, the "dial" operation to have the cloud
instance (gRPC client) connect to the laptop (gRPC server) can simply
use that existing TCP connection.

SSH is an incredible protocol that has more uses than most of its
users know of. In our case, it's a perfect way to decouple our TCP
connection from the gRPC connection. It has other benefits too:
although gRPC provides authentication and encryption, you can use
those from SSH if it's more convenient.

These examples are Go specific, but you can do something similar in
any language. The gRPC server doesn't need to listen on a port; you
can pass in any type that implement's Go's `net.Listener`. So we can
make a `net.Listener` that will take an SSH connection, and any time a
new SSH channel of our custom type is requested, then we will accept
that and return a new `net.Conn`, which is another type that we will
implement, that simply shuttles data over our tunnel.

Let's start with the SSHDataTunnel, which is our `net.Conn`

``` go
import (
  "net"
  "time"
  "golang.org/x/crypto/ssh"
)

// SSHDataTunnel implements net.Conn
type SSHDataTunnel struct {
    Chan ssh.Channel
    Conn net.Conn
}

func NewSSHDataTunnel(sshChan ssh.Channel, carrier net.Conn)
*SSHDataTunnel {
    return &SSHDataTunnel{
        Chan: sshChan,
        Conn: carrier,
    }
}

func (c *SSHDataTunnel) Read(b []byte) (n int, err error) {
    return c.Chan.Read(b)
}

func (c *SSHDataTunnel) Write(b []byte) (n int, err error) {
    return c.Chan.Write(b)
}

func (c *SSHDataTunnel) Close() error {
    return c.Chan.Close()
}

func (c *SSHDataTunnel) LocalAddr() net.Addr {
    return c.Conn.LocalAddr()
}

func (c *SSHDataTunnel) RemoteAddr() net.Addr {
    return c.Conn.RemoteAddr()
}

func (c *SSHDataTunnel) SetDeadline(t time.Time) error {
    return c.Conn.SetDeadline(t)
}

func (c *SSHDataTunnel) SetReadDeadline(t time.Time) error {
    return c.Conn.SetReadDeadline(t)
}

func (c *SSHDataTunnel) SetWriteDeadline(t time.Time) error {
    return c.Conn.SetWriteDeadline(t)
}

// Static checking for type
var _ net.Conn = &SSHDataTunnel{}
```

That last line just ensures that our type actually does implement
`net.Conn`, and won't compile if it doesn't.

Now, when we `Read()` and `Write()` our new `*SSHDataTunnel`, the
data is sent directly over a dedicated channel on an SSH
connection.

The `SSHChannelListener` is the `net.Listener` type that produces our `SSHDataTunnel`.


``` go
type SSHChannelListener struct {
    // channel requests are essentially the same as incoming TCP connections
    Chans   <-chan ssh.NewChannel
    SSHConn ssh.Conn
    TCPConn *net.TCPConn
}

// Accept waits for and returns the next connection to the listener.
func (l *SSHChannelListener) Accept() (net.Conn, error) {
    chanRq := <-l.Chans
    if chanRq == nil {
        return nil, net.ErrClosed
    }
    if chanRq.ChannelType() != "grpc-tunnel" {
        chanRq.Reject(ssh.UnknownChannelType, "unknown channel type")
        return nil, errors.New("could not accept on ssh channel listener: unknown channel type")
    }
    channel, reqs, err := chanRq.Accept()
    if err != nil {
        return nil, err
    }
    go ssh.DiscardRequests(reqs)
    return &SSHDataTunnel{
        Chan: channel,
        Conn: l.TCPConn,
    }, nil
}

// Close closes the listener.
// Any blocked Accept operations will be unblocked and return errors.
func (l *SSHChannelListener) Close() error {
    return l.SSHConn.Close()
}

// Addr returns the listener's network address.
func (l *SSHChannelListener) Addr() net.Addr {
    return l.SSHConn.LocalAddr()
}

```

Now we have adapters to use SSH in place of a `net.Listener` and
`net.Conn`, which is what gRPC requires. We still need to set up the
SSH connection itself to the cloud instance, in order
to construct the `SSHChannelListener`. This will be done on the TCP
client, which is the gRPC server (the "laptop").

``` go
// Connect makes an outgoing tunnel connection from the TCP client
// (which is the gRPC server) to the TCP server
func Connect() (*SSHChannelListener, error) {
    // Resolve and connect over TCP
    addr, err := net.ResolveTCPAddr("tcp4", ServerAddress)
    if err != nil {
        // real error handling here
        return nil, err
    }
    conn, err := net.DialTCP("tcp", nil, addr)
    if err != nil {
        // real error handling here
        return nil, err
    }

    // Using our TCP connection, establish an SSH connection
    sshConn, chans, requests, err := ssh.NewClientConn(conn, ServerAddress, c.ClientConfig)
    if err != nil {
        // real error handling here
        return nil, err
    }
    // ignore all requests (this does not include new channels)
    go ssh.DiscardRequests(reqs)

    return &SSHChannelListener{
        Chans:   chans,
        SSHConn: sshConn,
        TCPConn: conn,
    }, nil
}

```

Using the new `SSHChannelListener` on the laptop is easy with
gRPC. Assuming you have a service named `XServer` in a package
`protocol`:

``` go
listener, err := Connect()
if err != nil {
    // real error handling here
    panic(err}
}
s := grpc.NewServer()
protocol.RegisterXServer(s, &xServer{})
s.Serve(listener)
```

On the cloud instance (TCP server/gRPC client) we can have the gRPC
connect easily as well. First we will define a function that satisfies
`grpc.WithContextDialer`:

``` go
 func (c *Client) SSHConnDialer(context.Context, string) (net.Conn, error) {
     sshChan, reqs, err := c.sshConn.OpenChannel("grpc-tunnel", nil)
     if err != nil {
             return nil, err
     }
     go ssh.DiscardRequests(reqs)
     conn := &SSHConn{
             Chan: sshChan,
             Conn: nConn,
     }
     return conn, nil
}
```

And now we will use that dialer on the cloud instance to construct the
gRPC client:

``` go
grpcConn, err := grpc.Dial(ServerAddress,
    grpc.WithContextDialer(sshConnDialer),
    grpc.WithBlock(),
    grpc.WithTransportCredentials(insecure.NewCredentials()))
if err != nil {
    // real error handling here
    panic(err)
}
defer grpcConn.Close()
svc := protocol.NewXClient(grpcConn)

// Now you can use svc like a normal client:
svc.Frobulate()
```

## Handling Disconnects

Detecting and handling disconnects with `gRPC` can be a bit tricky so
I will outline one approach here. Instead of invoking a method on the
client side and getting an error (which is, of course, one way of
detecting disconnects), we can use a `grpc.Handler` to provide some
sort of notification as soon as a disconnect is detected. Using a
`chan` makes this really easy. The cloud instance (TCP server/gRPC
client) will just need a few more lines of code.

``` go
type DisconnectDetector struct {
    // add a logger or something too, if you want it
    CloseChan chan struct{}
}

// NewDisconnectDetector returns a valid DisconnectDetector
func NewDisconnectDetector() *DisconnectDetector {
    return &DisconnectDetector{
        CloseChan: make(chan struct{}),
    }
}

// TagRPC is a no-op
func (h *DisconnectDetector) TagRPC(context.Context, *stats.RPCTagInfo) context.Context {
    return context.Background()
}

// HandleRPC is a no-op
func (h *DisconnectDetector) HandleRPC(context.Context, stats.RPCStats) {
}

// TagConn is a no-op
func (h *DisconnectDetector) TagConn(context.Context, *stats.ConnTagInfo) context.Context {
    return context.Background()
}

// HandleConn processes the Conn stats.
func (h *DisconnectDetector) HandleConn(c context.Context, s stats.ConnStats) {
    switch s.(type) {
    case *stats.ConnEnd:
        h.CloseChan <- struct{}{}
    }
}
```


In the `grpc.Dial` call above, we can add another parameter:

``` go
disconnectDetector := NewDisconnectDetector()
grpc.Dial( // same as before, plus:
    grpc.WithStatsHandler(disconnectDetector))
```

Now whenever you want to be notified of a disconnection (in a separate
goroutine), just use:


``` go
<- disconnectDetector.CloseChan
// ... code to run upon disconnection
```
