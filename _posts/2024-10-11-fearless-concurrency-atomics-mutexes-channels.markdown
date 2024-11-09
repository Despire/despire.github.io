---
layout: post
title: "Fearless Concurrency in Go: Atomics, Mutexes, Channels"
date: 2024-10-11
categories: [Tech]
---

# Pararrelism and Concurrency in Programs

Rob Pike in his talk[^1] mentions that in his earlier years at Google, key people were arguing against writing code with threading, seeing it as a bad idea, leading to more difficult to understand programs resulting in harder to write correct code and reason about it. 

Nowadays it is rare that the programs we created are single threaded where we do not need to consider the effects of parallel access
for a shared resource at multiple points in a program, especially for server-side programs, although this has not always been the case.
As we live in world were many individual processes exists, possibly independently of eachother, interacting with these processes happens in a concurrent way and thus the idea of concurrency in programs is here stay.

# Concurrency vs Parallelism

Go is a concurrent programming language with great tools to support concurrency models for structuring your program such as goroutines, channels and select, while also providing other lower level primitives, namely mutexes and atomics, conditions.

Concurrency and parallelism are two different concepts that can sometimes be misunderstood and used in the wrong context.

In very simple terms, parallelism describes how multiple processes execute actions simultaneously, 
whereas concurrency involves managing multiple actions at the same time, but not necessarily executing them at the same time.
Concurrency is more about how you structure your program to consider multiple actions at the same time and possibly also taking advantage of parallelism, while parallelism is the process of executing actions simultaneously. 

Consider this example, if you had a CPU with only one core, parallelism would not be possible because there would be no way to run two actions/tasks/processes at the same time because there can only be one instruction running at any given time, whereas concurrency would be possible because you could structure your program to handle multiple actions at the same time, they just would not be executed at the same time. If you were to increase the number of cores on your CPU, parallelism would be achievable by scheduling two actions/tasks/processes on two different cores that execute instructions independently of each other, but this would also benefit the concurrency model because it can now take advantage of the multiple cores available and actually handle multiple actions at the same time. There is a great talk about this topic from Rob Pike which is highly recommended to watch[^2].

If you are still confused about this, to bring this more into the context of Go, below is a simple code example demonstrating concurrency vs parallelism, which you can also run yourself using the [Go playground](https://go.dev/play/).

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

Depending on which computer you run the program on, you will get different output, but if you run it multiple times on the same computer, you should get the same output repeatedly. On my computer the output is:

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

Since we instruct Go to use only a single CPU (`runtime.GOMAXPROCS(1)`) to schedule all 10 spawned goroutines, they will be executed concurrently but only on a single core, which, as mentioned above, cannot take advantage of parallelism. If we were to increase the number of CPU cores by setting `runtime.GOMAXPROCS(2)`, the output would be different each time, as the order of execution would be determined by the Go scheduler. We can now schedule up to 2 goroutines at the same time, i.e. take advantage of parallelism without any changes to the program as it was built using concurrent primitives.

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

# Channels, Goroutines, Select
> Don't communicate by sharing memory, share memory by communicating

A go [proverb](https://go-proverbs.github.io/) everyone who writes Go should know. Channels and goroutines are the primitives
for concurrency in Go. Goroutines are lightweight threads that are scheduled for execution by the runtime scheduler. Goroutines are
are not the same as OS threads, and in fact the runtime operates with a number of OS threads on which these goroutines are scheduled for 
execution, thus a goroutine is scheduled for execution based on the number of available OS Threads and a set of goroutines can be executed in parallel if the hardware is capable of parallel execution. Some of the key points to know about goroutines are:

- Cheap to create: A program can spawn thousands of goroutines that are scheduled by the runtime on the number of available OS threads, thus having an M:N model.
- Better performance compared to OS Threads: Creating and destroying hundreds to thousands of goroutines is faster and more performant than compared to OS Threads.
- Growable stack: Goroutines start with a small stack size of a few KBs and the stack grows according to the memory required, thus avoiding the possibility of stack overflows that occur with fixed-size stacks.
- Shared address space: Goroutines share the same address space as the program they're spawned in, thus synchronization to shared resources is needed to avoid race conditions.

Spawning a goroutine is as simple as using the `go` keyword.
```go
go handleWork()
```


By spawning goroutines we need a way to signal/communicate between them. This is where Channels come in. Channels are a key feature of the Go programming language that makes Go the concurrent programming language that it is. Channels are a communication medium for multiple goroutines to share memory by communicating with each other, rather than sharing memory directly between each of them to communicate, a model of communicating sequential processes[^3]. Creating channels in go is straighforward:

```go
chan := make(chan struct{})
```

We can perform 3 operations on channels, read, write, close:

```go
chan <- struct{}{} // write to a channel

// val will have the value that was send by the other goroutine.
val := <-chan // read from a channel

close(chan) // close a channel
```

Some of the key properties to know about channels:

- Blocking: Channels are blocking by default for both writes and reads.
- Buffered/unbuffered: Channels can be buffered by specifying the size at create time `make(chan struct{}, 5)`, a buffered channel will not block until the channel is full. 
- Close channels: Closing a channel will unblock other blocked goroutines, reading from a closed channel is valid and returns the null value for the type. It is possible to check if a channel has been closed by using double assignment.
```go
val, ok := <-chan // ok is true if channel is not closed and false if it is closed.
```
- Directionality: Channels can be directional (send-only or receive-only), which can be enforced in function signatures to improve code clarity and safety.
```go
func f1() <-chan struct{} {} // returns a read-only channel
func f2() chan<- struct{} {} // returns a write-only channel
```

To streamline working with multiple goroutines at once, go also provides the `select{}` statement as part of the language. Select behaves like a switch statement, but only works with channels. It blocks until one of the used channels is ready, or if none are ready and the default case is present it executes the default case.

```go
select{
    case v1 := <-c1 // channel c1 send a value.
    case v2 := <-c2 // channel c2 send a value.
    default:
        // no channel is ready to be read from
}
```

With just these 3 simple primitives, we can build powerful concurrent programs that are easy to write and reason about. Below is an example of using these primitives in a small program demonstrating multiplexing data from multiple channels.

```go
package main

import "fmt"

func main() {
    for val := range merge(worker(), worker()) {
        fmt.Printf("val: %v\n", val)
    }
}

func merge(c ...<-chan int) <-chan int {
    merged := make(chan int)
    go func() {
        defer close(merged)
        for {
            closed := 0
            for _, c := range c {
                select {
                case v, ok := <-c:
                    if !ok {
                        closed++
                        continue
                    }
                    merged <- v
                default:
                    // not ready skip.
                }
            }
            if closed == len(c) {
                return
            }
        }
    }()
    return merged
}

func worker() <-chan int {
    work := make(chan int)
    go func() {
        for i := 0; i < 10; i++ {
            work <- i
        }
        close(work)
    }()
    return work
}
```

```
val: 0
val: 0
val: 1
val: 1
val: 2
val: 2
val: 3
val: 3
val: 4
val: 4
val: 5
val: 5
val: 6
val: 6
val: 7
val: 7
val: 8
val: 8
val: 9
val: 9
```

As useful as these concurrency primitives are, you should not overuse them just because the language offers them to you. You should be
able to identify the problem you face and choose the right tool for the job. Therefore, the next two sections describes other synchronization primitives that can be used in conjunction.

# Mutexes, Atomics

If you are accessing a single resource from multiple goroutines that can run in parallel, and there is at least 1 modifier and reader of the resource, you need to synchronize access to that resource. If we don't need communication between the goroutines, using channels, depending on what needs to be done to the resource before another goroutine gets access to it, you can either use a mutex (short for mutual exclusion) or an atomic variable.

Looking at it a bit differently, this shared resource can be thought of as just memory that you want to synchronize access to. For example, the highlighted area in the next figure would be the memory we want to guard or synchronize access to so that no race condition occurs.

![alt text](/assets/img/concurrency/memory.png)

Whether to use a mutex or an atomic variable depends on how exactly you want to protect this region of memory from multiple goroutines.

A mutex (`sync.Mutex` in go) has only two methods, `Lock()` and `Unlock()`. When a goroutine acquires a lock, it now has exclusive access to that region of memory and is free to perform whatever transformations are needed on it; this is also referred to as the critical section that the mutex guards. Once a lock is acquired, it prevents any other goroutine from acquiring it until the lock is released. 

```go
m := &sync.Mutex{}

...

m.Lock() // start of critical section.

// perform operations needed to modify the guarded area.
...

m.Unlock() // end of critical section.
```

The two methods provided by a mutex define the span of the critical section. Depending on how large the span of the critical section is, and how many goroutines occupy that path, contention can occur and affect the performance of the program. It is important to keep the critical section as small as possible and to design the program to avoid high contention for a shared resource. Locking itself is not expensive in terms of CPU cycles, contention is.

Atomics, on the other hand, do not have the ability to define a critical section that mutexes provide, though it is possible to implement them using just these atomic variables. When a variable is atomic, it guarantees that writes and reads are performed atomically, so that multiple goroutines always work with the most recent version of the variable enabling lock-free programming techniques that do not have lock contention. Atomics in go have the following 4 methods by default, with some additional ones depending on what the atomic type is, all of which are executed atomically and are the building blocks for lock-free data structures and programming.

```go
Store(new) // atomically stores value into the memory.
Load() // atomically loads value from the memery.
CompareAndSwap(old, new) // compares if the memory still holds the old value, if so swaps it with the new value.
Swap(new) // swaps the old value with the new value.
```

Using `CompareAndSwap()` can be a powerful instruction in lock-free programming. For example, below it is used to implement a SpinLock[^4] that mimics the properties of a mutex.

```go
type SpinLock struct {
	l int32
}

func (l *SpinLock) Lock() {
	for !atomic.CompareAndSwapInt32(&l.l, 0, 1) {
	}
}

func (l *SpinLock) Unlock() {
	atomic.StoreInt32(&l.l, 0)
}
```

When to use a mutex?

- You need to perform multiple operations that need to be treated as a single "atomic" operation before another goroutine is granted access.
- Ensure consistency for data structures accessed by multiple goroutines.

When to use an atomic?

- When using primitive data types.
- Minimize lock contention, resulting in better performance, lock-free programming.
- Useful for counters or flags.


# Conditions

The `sync.Cond` type in Go is a bit controversial, as it's place in the program would have to be really justified to be used correctly,
with all the previous primitives available, especially channels. There has even been a proposal to the Go programming language to get rid of [`sync.Cond`](https://github.com/golang/go/issues/21165), which was recently categorized as a likely denial, so the type is here to stay.

Condition are something like channels but on a way lower level and are used in conjuction with a Mutex. They're primarly used for signaling the occurence of an event for the coordination of multiple goroutines. The following methods are available:

- `Wait()`: Suspends the calling goroutine until another goroutine signals the condition variable.
- `Signal()`: Wakes up one waiting goroutine.
- `Broadcast()`: Wakes up all waiting goroutines.

Below is a simple example that demonstrates how to unblock a goroutine that is waiting for a particular event.

```go
func main() {
    m := &sync.Mutex{}
    c := sync.NewCond(m)

    var event bool

    go func() {
        c.L.Lock()
        for !event {
            c.Wait()
        }
        fmt.Printf("Condition met processing data...\n")
        time.Sleep(3 * time.Second)
        c.L.Unlock()
    }()

    c.L.Lock()
    fmt.Printf("processing data...\n")
    time.Sleep(2 * time.Second)
    fmt.Printf("finished processing data, sending finish event\n")
    event = true
    c.L.Unlock()

    c.Signal()

    time.Sleep(1 * time.Second)
}
```

The `sync.Cond` type should really be used in scenarios where fine-grained control over goroutine execution is required and the overhead of channels is a performance hit, but extra care should be taken as subtle bugs can be introduced if not used carefully.

# Conclusion

Go allows you to structure your programs to make effective use of concurrency models and to facilitate the scalability of your code by abstracting away the complexity of dealing with the lower-level instructions. In the case of synchronisation primitives for dealing with limited access to a shared resource, it is important to remember that locking is not expensive, contention is. Structuring your programs to allow high contention for shared resources will be the biggest bottleneck of your program, and you should aim to structure your program to minimise contention and maximise performance and scalability.

# Further Reading

Go's memory model
- https://go.dev/ref/mem
- https://www.cs.cmu.edu/afs/cs.cmu.edu/academic/class/15440-f11/go/doc/go_mem.html

- [Communicating Sequential Processes](https://www.cs.cmu.edu/~crary/819-f09/Hoare78.pdf)

**Footnotes**

[^1]: [Rob Pike - From Parallel to Concurrent (Accessed 06 Nov 2024)](https://www.youtube.com/watch?v=iTrP_EmGNmw)
[^2]: [Rob Pike - Concurrency is not Parallelism (Accessed 06 Nov 2024)](https://www.youtube.com/watch?v=oV9rvDllKEg)
[^3]: [C.A.R. Hoare - Communicating Sequential Processes (Accessed 07 Nov 2024)](https://www.cs.cmu.edu/~crary/819-f09/Hoare78.pdf)
[^4]: [Spin Lock (Accessed 09 Nov 2024)](https://en.wikipedia.org/wiki/Spinlock)
