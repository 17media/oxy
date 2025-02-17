Oxy [![Build Status](https://travis-ci.org/vulcand/oxy.svg?branch=master)](https://travis-ci.org/vulcand/oxy)
=====

Oxy is a Go library with HTTP handlers that enhance HTTP standard library:

* [Buffer](https://pkg.go.dev/github.com/17media/oxy/buffer) retries and buffers requests and responses 
* [Stream](https://pkg.go.dev/github.com/17media/oxy/stream) passes-through requests, supports chunked encoding with configurable flush interval 
* [Forward](https://pkg.go.dev/github.com/17media/oxy/forward) forwards requests to remote location and rewrites headers 
* [Roundrobin](https://pkg.go.dev/github.com/17media/oxy/roundrobin) is a round-robin load balancer 
* [Circuit Breaker](https://pkg.go.dev/github.com/17media/oxy/cbreaker) Hystrix-style circuit breaker
* [Connlimit](https://pkg.go.dev/github.com/17media/oxy/connlimit) Simultaneous connections limiter
* [Ratelimit](https://pkg.go.dev/github.com/17media/oxy/ratelimit) Rate limiter (based on tokenbucket algo)
* [Trace](https://pkg.go.dev/github.com/17media/oxy/trace) Structured request and response logger

It is designed to be fully compatible with http standard library, easy to customize and reuse.

Status
------

* Initial design is completed
* Covered by tests
* Used as a reverse proxy engine in [Vulcand](https://github.com/vulcand/vulcand)

Quickstart
-----------

Every handler is ``http.Handler``, so writing and plugging in a middleware is easy. Let us write a simple reverse proxy as an example:

Simple reverse proxy
====================

```go

import (
  "net/http"
  "github.com/17media/oxy/forward"
  "github.com/17media/oxy/testutils"
  )

// Forwards incoming requests to whatever location URL points to, adds proper forwarding headers
fwd, _ := forward.New()

redirect := http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
    // let us forward this request to another server
		req.URL = testutils.ParseURI("http://localhost:63450")
		fwd.ServeHTTP(w, req)
})
	
// that's it! our reverse proxy is ready!
s := &http.Server{
	Addr:           ":8080",
	Handler:        redirect,
}
s.ListenAndServe()
```

As a next step, let us add a round robin load-balancer:


```go

import (
  "net/http"
  "github.com/17media/oxy/forward"
  "github.com/17media/oxy/roundrobin"
  )

// Forwards incoming requests to whatever location URL points to, adds proper forwarding headers
fwd, _ := forward.New()
lb, _ := roundrobin.New(fwd)

lb.UpsertServer(url1)
lb.UpsertServer(url2)

s := &http.Server{
	Addr:           ":8080",
	Handler:        lb,
}
s.ListenAndServe()
```

What if we want to handle retries and replay the request in case of errors? `buffer` handler will help:


```go

import (
  "net/http"
  "github.com/17media/oxy/forward"
  "github.com/17media/oxy/buffer"
  "github.com/17media/oxy/roundrobin"
  )

// Forwards incoming requests to whatever location URL points to, adds proper forwarding headers

fwd, _ := forward.New()
lb, _ := roundrobin.New(fwd)

// buffer will read the request body and will replay the request again in case if forward returned status
// corresponding to nework error (e.g. Gateway Timeout)
buffer, _ := buffer.New(lb, buffer.Retry(`IsNetworkError() && Attempts() < 2`))

lb.UpsertServer(url1)
lb.UpsertServer(url2)

// that's it! our reverse proxy is ready!
s := &http.Server{
	Addr:           ":8080",
	Handler:        buffer,
}
s.ListenAndServe()
```
