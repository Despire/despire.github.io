---
layout: post
title: "Fearless Concurrency in Go: Atomics, Mutexes, Channels"
date: 2024-10-11
categories: [Tech]
---

# Pararrelism and Concurrency in Programs

Nowadays it is rare that the programs we created are single threaded where we do not need to consider the effects of parallel access
for a shared resource at multiple points in a program, especially for server-side programs, although this has not always been the case.

Rob Pike in his talk[^1] mentions that in his earlier years at Google, key people were arguing against writing code with threading, seeing it as a bad idea, leading to more difficult to understand programs resulting in harder to write correct code and reason about it. 

As we live in world were many individual processes exists, possibly independently of eachother, interacting with these processes happens in a concurrent way and thus the idea of concurrency in programs won and is here to stay.

# Concurrency vs Parallelism

Go is a concurrent programming language with great tools to support concurrency models for structuring your program such as goroutines and channels, while also providing other lower level primitives, namely mutexes and atomics, conditions.

Concurrency and parallelism are two different concepts that can sometimes be misunderstood and used in the wrong context.

In very simple terms, parallelism describes how multiple processes execute actions simultaneously, 
whereas concurrency involves managing multiple actions at the same time, but not necessarily executing them at the same time.
Concurrency is more about how you structure your program to consider multiple actions at the same time and possibly also taking advantage of parallelism, while parallelism is the process of executing actions simultaneously. 

Consider this example, if you had a CPU with only one core, parallelism would not be possible because there would be no way to run two actions/tasks/processes at the same time because there can only be one instruction running at any given time, whereas concurrency would be possible because you could structure your program to handle multiple actions at the same time, they just would not be executed at the same time. If you were to increase the number of cores on your CPU, parallelism would be achievable by scheduling two actions/tasks/processes on two different cores that execute instructions independently of each other, but this would also benefit the concurrency model because it can now take advantage of the multiple cores available and actually handle multiple actions at the same time. There is a great talk about this topic from Rob Pike which is highly recommended to watch[^2].

If you are still confused about this, to bring this more into the context of Go, below is a simple code example demonstrating concurrency vs parralelism, which you can also run yourself using the [Go playground](https://go.dev/play/).

```go
// go version: 1.23
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

func main() {
    // These changes how many CPUs will be used
    // to execute the scheduled goroutines.
    runtime.GOMAXPROCS(1)

    wg := sync.WaitGroup{}

    wg.Add(10)
    for i := range 10 {
        go func() {
            defer wg.Done()
            fmt.Printf("goroutine %v start\n", i)
            time.Sleep(2 * time.Second)
            fmt.Printf("goroutine %v end\n", i)
        }()
    }

    wg.Wait()
}
```

Depending on which computer you run the program on, you will get different output, but if you run it multiple times on the same computer, you will get the same output repeatedly. On my computer the output is:

```
go run ./main.go
goroutine 9 start
goroutine 0 start
goroutine 1 start
goroutine 2 start
goroutine 3 start
goroutine 4 start
goroutine 5 start
goroutine 6 start
goroutine 7 start
goroutine 8 start
goroutine 8 end
goroutine 9 end
goroutine 0 end
goroutine 1 end
goroutine 2 end
goroutine 3 end
goroutine 4 end
goroutine 5 end
goroutine 6 end
goroutine 7 end
```

Since we instruct Go to use only a single CPU (`runtime.GOMAXPROCS(1)`) to schedule all 10 spawned goroutines, they will only be executed on a single core, which, as mentioned above, cannot take advantage of parallelism. If we were to increase the number of CPU cores by setting `runtime.GOMAXPROCS(2)`, the output would be different each time, as we could schedule up to 2 goroutines at the same time, i.e. take advantage of parallelism without any changes to the program as it was built using concurrent primitives.

```
go run ./main.go
goroutine 1 start
goroutine 5 start
goroutine 9 start
goroutine 7 start
goroutine 0 start
goroutine 2 start
goroutine 6 start
goroutine 3 start
goroutine 4 start
goroutine 8 start
goroutine 8 end
goroutine 1 end
goroutine 5 end
goroutine 3 end
goroutine 9 end
goroutine 6 end
goroutine 2 end
goroutine 0 end
goroutine 4 end
goroutine 7 end

go run ./main.go
goroutine 9 start
goroutine 3 start
goroutine 1 start
goroutine 2 start
goroutine 5 start
goroutine 6 start
goroutine 7 start
goroutine 8 start
goroutine 4 start
goroutine 0 start
goroutine 4 end
goroutine 0 end
goroutine 3 end
goroutine 1 end
goroutine 7 end
goroutine 2 end
goroutine 6 end
goroutine 9 end
goroutine 5 end
goroutine 8 end
```

# Channels, Goroutines

# Mutexes, Atomics, Conditions

# Conclusion
Go allows you to structure your programs to make effective use of concurrency models and to facilitate the scalability of your code by abstracting away the complexity of dealing with the lower-level instructions. In the case of synchronisation primitives for dealing with limited access to a shared resource, it is important to remember that locking is not expensive, contention is. Structuring your programs to allow high contention for shared resources will be the biggest bottleneck of your program, and you should aim to structure your program to minimise contention and maximise performance and scalability.

**Footnotes**

[^1]: [Rob Pike - From Parallel to Concurrent](https://www.youtube.com/watch?v=iTrP_EmGNmw)
[^2]: [Rob Pike - Concurrency is not Parallelism](https://www.youtube.com/watch?v=oV9rvDllKEg)
