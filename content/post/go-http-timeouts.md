---
title: "HTTP Client Timeouts with Context in Go"
date: 2022-06-11T21:23:09Z
draft: false
tags: [ "http", "go", "timeout", "networking", "context" ]
---

# HTTP Client Timeouts in Go

The Go standard library is phenomenal. It includes
[net/http](https://pkg.go.dev/net/http) which implements a
production-quality HTTP client and server. But on a recent project, I
encountered a surprising design that makes a common case
difficult. What follows is a walkthrough of the solution.

## Client timeouts that are configurable

There are a large number of configurable HTTP(S) timeouts:

#### [http.Client.Timeout](https://pkg.go.dev/net/http#Client)

Basically this is the very high-level timeout which cannot be used in
many situations (e.g., downloading a large file). It is a very blunt
instrument.

From the docs:

    This field specifies a time limit for requests made by this
    Client. The timeout includes connection time, any redirects, and
    reading the response body. The timer remains running after Get, Read,
    Post, or Do return and will interrupt reading of the Response.Body. A
    Timeout of zero means no timeout.

``` go
client := &http.Client{Timeout: 50 * time.Second}
```

#### [http.Transport](https://pkg.go.dev/net/http#Transport) timeouts

Here is the setting of the default transport:

``` go
var DefaultTransport RoundTripper = &Transport{
    IdleConnTimeout:       90 * time.Second,
    TLSHandshakeTimeout:   10 * time.Second,
    ExpectContinueTimeout: 1 * time.Second,
}
```

So there are three HTTP/TLS timeouts directly on the Transport
itself.

#### [net.Dialer](https://pkg.go.dev/net#Dialer) timeout

This one just handles the TCP connection initially. The net.Dialer is
used by [http.Transport](https://pkg.go.dev/net/http#Transport) via
`DialContext`.

``` go
net.Dialer{
    Timeout:   30 * time.Second,
    KeepAlive: 30 * time.Second, // not exactly a timeout...
}
```

There are other timeouts that are configurable on `http.Server`, but
this post is limited to `http.Client`.

## Problematic scenario

Let's examine a scenario in which you begin downloading a large file
(10GB). The download starts fine, goes great for an hour, then slows
way down, and eventually stops. For an entire minute, not a single
byte is transferred. What timeout occurs? _*None*_.

But this is a real thing that happens. Here are some scenarios that
cause this:

* Anti-scraper measures, especially on ad-infested file hosts. This
  one is particularly insidious. Some file hosts will allow you to
  download a certain amount of data for free, then will dramatically
  slow the connection and eventually will stop the transfer of data
  completely, but without closing the connection or giving any
  indication of an error. This makes scrapers for these sites hard to
  write, which is clearly their intent.
* A mobile connection that has a bad signal but doesn't drop the
  connection.
* A misbehaving proxy. Upstream of the proxy, something goes wrong,
  but the proxy doesn't time out for too long.
* Tor. The circuit can break at some point but the connection might
  not be dropped for a long time.

## Solution

Go's `context.Context` is our friend here. Cloudflare provides [a
pointer to a great solution on their
blog](https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/).
Unfortunately, it is rather cumbersome to use so I will provide a
simple and reusable type.

We can implement an `io.Writer` that notifies its writer of a timeout via
`context.Context` and the cancellation it can provide. I'll call that
writer `TimeoutWriter`.

Here is the whole implementation:

``` go
import (
    "context"
    "fmt"
    "sync"
    "time"
)

type TimeoutWriter struct {
    d        time.Duration
    timer    *time.Timer
    ctx      context.Context
    cancel   context.CancelFunc
    timedOut bool
    once     sync.Once
    // about last time write was done
    lastTime  time.Time
    lastBytes int
}

func NewTimeoutWriter(ctx context.Context, d time.Duration) *TimeoutWriter {
    ctx, cancel := context.WithCancel(ctx)
    return &TimeoutWriter {
        d: d,
            timer: nil,
            ctx: ctx,
            cancel: cancel,
    }
}

func (t *TimeoutWriter) initialize() {
    t.timer = time.AfterFunc(t.d, t.doTimeout)
}

func (t *TimeoutWriter) doTimeout() {
    // potentially do other stuff here
    t.timedOut = true
    t.cancel()
}

func (t *TimeoutWriter) IsTimedOut() bool {
    return t.timedOut
}

func (t *TimeoutWriter) String() {
    last := "never"
    if !t.lastTime.IsZero() {
        elapsed := time.Now().Sub(t.lastTime)
        last = fmt.Sprintf("%0.3f s", elapsed.Seconds())
    }
    return fmt.Sprintf(
        "last write %v (%v B)",
        last, t.lastBytes)
}

func (t *TimeoutWriter) Cancel() {
    t.timer.Stop()
}

func (t *TimeoutWriter) Write(data []byte) (int, error) {
    t.once.Do(t.initialize)
    t.timer.Reset(t.d)
    t.lastTime = time.Now()
    t.lastBytes = len(data)
    return len(data), nil
}
```

Here's an example of consuming this:

``` go
func DownloadFile(path, url string) error {
    file, err := os.Create(path)
    if err != nil {
      return err
    }

    res, err := http.Get(url)
    if err != nil {
      return err
    }
    defer res.Body.Close()

    ctx := context.Background()
    request, err := http.NewRequestWithContext(
        ctx, http.MethodGet, url, nil)

    tw := NewTimeoutWriter(ctx, 20*time.Second)
    defer tw.Cancel()

    sink := io.MultiWriter(file, tw)
    readbytes, err := io.Copy(sink, res.Body)

    if readbytes != res.ContentLength {
      // ... seems bad
    }

    if err != nil {
       // definitely bad
       return err
    }

    return nil
}
```

`io.MultiWriter` saves the day here. Every time a write occurs in
`io.Copy`, the timeout's timer can be reset.


## Bibliography

* [net/http docs](https://pkg.go.dev/net/http)
* [The complete guide to Go net/http timeouts - CloudFlare](https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/)
