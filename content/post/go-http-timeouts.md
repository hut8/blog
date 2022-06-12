---
title: "HTTP Client Timeouts with Context in Go"
date: 2022-06-10T21:23:09Z
draft: true
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

#### Idle Connection Timeout

When a connection is idle (keep-alive is enabled and the last reshas been received, how long should the idle connection stay open?

``` go
tr := &http.Transport{
    IdleConnTimeout:    30 * time.Second,
}
client := &http.Client{Transport: tr}
```

#### `client.Timeout`
This field specifies a time limit for requests made by this
Client. The timeout includes connection time, any redirects, and
reading the response body. The timer remains running after Get, Read,
Post, or Do return and will interrupt reading of the Response.Body. A
Timeout of zero means no timeout. (quoting the docs)

``` go
client := &http.Client{Timeout: 50 * time.Second}
```

There are other timeouts that are configurable on `http.Server`, but
this post is limited to `http.Client`.
