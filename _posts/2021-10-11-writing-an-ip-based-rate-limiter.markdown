---
layout: post
title: "Writing an IP based rate limiter"
data: 2021-10-11
categories: [Tech]
---
# What is a Rate Limiter ?

I think the definition from Wikipedia is pretty accurate. In simple terms put, it is something that limits the rate of requests, either incoming or outgoing.[^1]

Why should you be using a rate limiter you may ask? In the first place in can be used as a mechanism to prevent certain attacks at your deployed service, such as 
distributed denial of service (DDOS in short) or if your service supports multiple client platforms (i.e mobile, PCs) you can have different rate limits for each platform. Furthermore, if you offer multiple tiers for you service, something like a free, silver or gold tier, you can also apply different rate limits for each of those tiers and if the customers depends on your service you can offer them a higher throughput with paid tiers or you just want to limit the number of incoming requests as your deployed service is experiencing some issues.[^2]<sup>,</sup>[^3] In addition, you may also use a rate limiter if you deployed your service on a rented VM, which calculates the cost based on the CPU usage or some other factors, and want to keep the price as low as possible.

Over the years different algorithms were developed for rate limiting like token bucket, leaky bucket, fixed window or sliding window to name a few[^4]. To keeps things simple we will only work
with the Token Bucket algorithm in this article, as this seems to be the more popular one.

# Token Bucket

The algorithm is very well described on wikipedia, to my surprise. The idea of the algorithm is 
as follows. Imagine you have a Bucket of some size and that you are filling up the bucket with something at some rate. The bucket size is referred to as the `burst size` and the rate as the `limit`.[^5] There are many ways on how to implement this algorithm, some start with a prefilled
bucket and other with an empty bucket. It is entirely up to the reader to recognize which fits the
best.

To give you an specific example. Let's say we have a bucket that can hold up to 30 stones and
that we are throwing stones into the bucket at a rate of 2 stones/sec. This means that our burst size will be 30 stones and our limit 2 stones/sec. The following figure may help you in
understanding the idea.

![alt text](/assets/img/rate-limiter/bucket.png)

After 3 seconds at the rate limit of 2 stones/sec we end up with a total of 6 stones out of 30. Since we have collected
6 stones the next "request" can take one of them or all 6 of them. If we reach the limit of 30 stones there are numerous
ways to deal with such situation. For example discarding any exceeding stones (as they wouldn't fit into the bucket anymore)
or simply waiting until space frees up and such.

# Implementing a Rate Limiter in Go

We'll be using the above described algorithm but we won't be implementing it. We'll use an package
that already has a concurent safe implementation https://pkg.go.dev/golang.org/x/time/rate. We'll be
building our IP based rate limiter upon this implementation.

To start with we'll be distinguishing each user by their IP address. Thus what we need to do is to
map an IP address to a rate limiter from the above mentioned package.

```go
package rate

type ipLimiter map[string] *rate.Limiter

type Limiter struct {
    // maps ip's to rate limiters.
    limiter ipLimiter

    // the rate at which events happen per second.
    l float64

    // the burst size (the bucket size in other words).
    b int

    // since this data structure will be accessed
    // concurrently by multiple go-routines we'll
    // protect the fields by a read-write lock.
    raw sync.RWMutex
}

// Returns an initialized instance of Limiter
func NewLimiter(l float64, b int) *Limiter {
    return &Limiter{
        limiter: make(ipLimiter),
        l:       l,
        b:       b,
        rw:      sync.RWMutex{},
    }
}
```

Next we'll need to add some methods to add/retrieve an limiter for
a ip address. First the method for identifying a new IP address.

```go
// Adds a new IP address to the rate limiter.
func(l *Limiter) AddFor(ip string) {
    l.rw.Lock()
    // the NewLimiter function is from the
    // golang.org/x/time/rate package
    l.limiter[ip] = rate.NewLimiter(l.l, l.b)
    l.rw.Unlock()
}
```

Secondly the method for retreiving the rate limiter for an IP address.
We can simply implement it by only fetching from the map but we decided
to add additional functionality to it by also inserting a new rate limiter
if there isn't one for that ip adddress.

```go
// Returns an existing limiter for an IP address.
// If there isn't such limiter, a new one is created.

// Note the *rate.Limiter is from the golang.org/x/time/rate package.
func (l *Limiter) GetFor(ip string) *rate.Limiter {
    l.rw.RLock()
    limiter, ok := l.limiter[ip]
    l.rw.RUnlock()

    if !ok {
        // the NewLimiter function is from the
        // golang.org/x/time/rate package
        limiter = rate.NewLimiter(l.l, l.b)

        l.rw.Lock()
        l.limiter[ip] = limiter
        l.rw.Unlock()
    }

    return limiter
}
```

This summarizes the whole rate limiter package with really basic functionality.
You can view the full code here: https://github.com/Despire/journal/tree/main/cmd/server/http/handlers/internal/rate

## Using the Rate Limiter

To use the above Rate limiter we'll wrap it in a middleware.

```go
func RateLimitMiddleware(limit float64, burst int, next http.Handler) http.Handler {
    // This is the NewLimiter method from our rate package that we created earlier.
    limiter := rate.NewLimiter(limit, burst)

    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        limiter := limiter.GetFor(r.RemoteAddr)
        if !limiter.Allow() {
            w.Header().Set("Content-Type", "text/html")
            if _, err := w.Write([]byte("Too many requests, try again later")); err != nil {
                logger.Errof("failed to write response for ip: %v, reason: %v", r.RemoteAddr, err)
            }
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

And then use the middleware to wrap a router for an http server like so
```go
http.ListenAndServe("127.0.0.1:8080", RateLimitMiddleware(limit, burst, router))
```

**Footnotes**

[^1]: [Rate limiting, https://en.wikipedia.org/wiki/Rate_limiting](https://en.wikipedia.org/wiki/Rate_limiting) [accessed 11.10.2021]
[^2]: [Rate limiting, https://en.wikipedia.org/wiki/Rate_limiting](https://en.wikipedia.org/wiki/Rate_limiting) [accessed 11.10.2021]
[^3]: [What are Rate Limiters?, https://redisson.org/glossary/rate-limiter.html](https://en.wikipedia.org/wiki/Rate_limiting) [accessed 11.10.2021]
[^4]: [Rate limiting, algorithms section, https://en.wikipedia.org/wiki/Rate_limiting](https://en.wikipedia.org/wiki/Rate_limiting) [accessed 11.10.2021]

[^5]: [Token Bucket algorithm, https://en.wikipedia.org/wiki/Token_bucket](https://en.wikipedia.org/wiki/Token_bucket) [accessed 11.10.2021]
