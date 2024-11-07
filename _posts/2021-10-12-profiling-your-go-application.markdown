---
layout: post
title: "Profiling your Go application"
date: 2021-10-12
categories: [Tech]
---
If you don't know what profiling means/is you can read about it on
[Wikipedia](https://en.wikipedia.org/wiki/Profiling_(computer_programming)), as the concept is described there very well.

# Profiling in Go

The Go programming language has a rich standard library, some argue that
it's a positive thing while other seem to think the opposite. In Go if you want to profile your application you can use the `net/http/pprof` package from
the standard library. In your Go program you would just import the package like so:

```go
import(
    _ "net/http/pprof"
)
```

This simple import is a rather dangerous part of a program. What's wrong with it you may ask? Well the first red sign is that it's a blank import. Meaning we're importing it for the side effects of an init function (I'd like to note that you should always be cautious about what you're importing into your program and you always should check the code for what it does as some packages could be using init functions that you may not know of). If we look inside the package and look for the init function we can find this:[^2]

```go
func init() {
	http.HandleFunc("/debug/pprof/", Index)
	http.HandleFunc("/debug/pprof/cmdline", Cmdline)
	http.HandleFunc("/debug/pprof/profile", Profile)
	http.HandleFunc("/debug/pprof/symbol", Symbol)
	http.HandleFunc("/debug/pprof/trace", Trace)
}
```

So by default after importing the package it will register the all the pprof endpoints on the default server mux in the http package. What this means is that if you would to deploy your application and your application uses the default http mux anyone would have access to your
profiling data (assuming that you don't authorize all endpoints by default) by simply accessing the `/debug/pprof/` endpoint, yikes!

So how would be go about not exposing the endpoint? Well, as already mentioned you could make the endpoint require authorization of some kind but I'm more of a fan of simply separating the endpoints for you application and the pprof webserver into two different webservers on separate ports and making the pprof server accessible only from localhost.[^3]<sup>,</sup>[^4]

Your code should then look something like this:
```go
import (
    ...

    "net/http/pprof"
    
    ...
)

...

go func() {
    pprofMux := mux.NewRouter()
    
    pprofMux.HandleFunc("/debug/pprof/", pprof.Index).Methods(http.MethodGet)
    pprofMux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline).Methods(http.MethodGet)
    pprofMux.HandleFunc("/debug/pprof/profile", pprof.Profile).Methods(http.MethodGet)
    pprofMux.HandleFunc("/debug/pprof/symbol", pprof.Symbol).Methods(http.MethodGet)
    pprofMux.HandleFunc("/debug/pprof/trace", pprof.Trace).Methods(http.MethodGet)
    pprofMux.Handle("/debug/pprof/goroutine", pprof.Handler("goroutine")).Methods(http.MethodGet)
    pprofMux.Handle("/debug/pprof/heap", pprof.Handler("heap")).Methods(http.MethodGet)
    pprofMux.Handle("/debug/pprof/threadcreate", pprof.Handler("threadcreate")).Methods(http.MethodGet)
    pprofMux.Handle("/debug/pprof/block", pprof.Handler("block")).Methods(http.MethodGet)
    pprofMux.Handle("/debug/pprof/allocs", pprof.Handler("allocs")).Methods(http.MethodGet)
    pprofMux.Handle("/debug/pprof/mutex", pprof.Handler("mutex")).Methods(http.MethodGet)

    http.ListenAndServe("127.0.0.1:"+someOtherPort, pprofMux)
}()

...


http.ListenAndServe(mainAppAddress, mainAppMux)
```

voilà ![^1] Your pprof logic is now separated from your app logic and only accessible from your localhost thus if you deploy your app onto cloud hosting services or on a rented VM you can access it by remotly accessing that machine and just inspecting the pprof data on a live production/dev deployment of your app
with the `go tool pprof` command.

# Profiling in a production environment

You may be asking yourself if it's okay to enable profiling in a production environment. Well yes it is totally okay and pprof is safe to use in production. Profiling in production has many benefits as certain problems will be visible only in production and are hard to catch in dev environments. Furthermore you it may help you understand your program better so you can reduce your CPU usage bill.[^5]

According to Jaana dogan there is an additional 5% overhead for CPU and heap allocation profiling. [^6]

**Footnotes**

[^1]: [Merriam-Webster Dictionary](https://www.merriam-webster.com/dictionary/voil%C3%A0) [accessed 12.10.2021]

[^2]: [Go stdlib pprof package](https://cs.opensource.google/go/go/+/refs/tags/go1.17.2:src/net/http/pprof/pprof.go;l=80) [accessed 12.10.2021]

[^3]: [Remote Profiling of Go programs, Access Control section, Farsight security](https://www.farsightsecurity.com/blog/txt-record/go-remote-profiling-20161028/) [accessed 12.10.2021]

[^4]: [Your pprof is showing, Prevention section, Michael McLoughlin](https://mmcloughlin.com/posts/your-pprof-is-showing) [accessed 12.10.2021]

[^5]: [Continuous Profiling of Go programs, Jaana Dogan](https://medium.com/google-cloud/continuous-profiling-of-go-programs-96d4416af77b) [accessed 12.10.2021]

[^6]: [Continuous Profiling of Go programs, Profiling in production section, Jaana Dogan](https://medium.com/google-cloud/continuous-profiling-of-go-programs-96d4416af77b) [accessed 12.10.2021]
