# Go Concurrency Patterns — Study Notes

**Source:** [Master Go Programming With These Concurrency Patterns (in 40 minutes)](https://www.youtube.com/watch?v=qyM8Pi1KiiM&list=PL7g1jYj15RUNqJStuwE9SCmeOKpgxC0HP&index=1)

> **Note:** These notes are reconstructed from the video topic and supporting official Go resources. The video covers the most useful Go concurrency patterns in ~40 minutes.

---

## Timeline Overview

| Timestamp | Topic |
|-----------|-------|
| 0:00 | Concurrency vs Parallelism |
| 2:00 | Goroutines |
| 7:00 | Channels (unbuffered) |
| 14:00 | Channels (buffered) |
| 18:00 | WaitGroups |
| 22:00 | Mutex |
| 26:00 | Select Statement |
| 30:00 | Worker Pool Pattern |
| 35:00 | Fan-Out / Fan-In Pattern |
| 38:00 | Pipeline Pattern |
| 39:20 | (t=2360s — video link starts here) Context & Cancellation |

---

## 1. Concurrency vs Parallelism (0:00)

- **Concurrency**: Composing independently executing processes (can run on 1 core)
- **Parallelism**: Multiple processes executing simultaneously (requires multiple cores)
- Concurrency enables parallelism but is NOT the same thing
- Go's mantra: **"Don't communicate by sharing memory; share memory by communicating."**

---

## 2. Goroutines (2:00)

A goroutine is a lightweight thread managed by the Go runtime, not the OS.

```go
package main

import (
    "fmt"
    "time"
)

func say(msg string) {
    for i := 0; i < 3; i++ {
        fmt.Println(msg, i)
        time.Sleep(100 * time.Millisecond)
    }
}

func main() {
    go say("world") // runs concurrently
    say("hello")    // runs in main goroutine
}
```

**Key facts:**
- Created with the `go` keyword
- Very cheap — you can have thousands running simultaneously
- Has its own call stack (grows/shrinks dynamically)
- NOT OS threads — Go runtime multiplexes goroutines onto threads

---

## 3. Channels — Unbuffered (7:00)

Channels are typed conduits for communication between goroutines.

```go
package main

import "fmt"

func sum(s []int, ch chan int) {
    total := 0
    for _, v := range s {
        total += v
    }
    ch <- total // send result to channel
}

func main() {
    s := []int{1, 2, 3, 4, 5, 6, 7, 8}
    ch := make(chan int) // unbuffered channel

    go sum(s[:len(s)/2], ch)
    go sum(s[len(s)/2:], ch)

    x, y := <-ch, <-ch // receive from channel (blocks until ready)
    fmt.Println(x, y, x+y)
}
```

**Key facts:**
- `make(chan Type)` creates unbuffered channel
- Send (`ch <- value`) blocks until receiver is ready
- Receive (`value := <-ch`) blocks until sender is ready
- This synchronization is automatic — no locks needed
- Always `close(ch)` when done sending so receivers know to stop

### Generator Pattern (function returning channel)

```go
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

func main() {
    ch := generate(2, 3, 4, 5)
    for v := range ch { // range on channel reads until closed
        fmt.Println(v)
    }
}
```

---

## 4. Channels — Buffered (14:00)

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 3) // buffered — capacity of 3

    ch <- 1 // does NOT block (buffer has space)
    ch <- 2
    ch <- 3
    // ch <- 4 // would BLOCK — buffer full

    fmt.Println(<-ch) // 1
    fmt.Println(<-ch) // 2
    fmt.Println(<-ch) // 3
}
```

**Buffered vs Unbuffered:**
| | Unbuffered | Buffered |
|--|--|--|
| Blocks on send | Until receiver ready | Until buffer full |
| Blocks on receive | Until sender ready | Until buffer non-empty |
| Use case | Tight synchronization | Decouple producer/consumer speed |

---

## 5. WaitGroups (18:00)

Use `sync.WaitGroup` when you launch multiple goroutines and need to wait for ALL of them to finish.

```go
package main

import (
    "fmt"
    "sync"
)

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done() // decrements counter when goroutine finishes
    fmt.Printf("Worker %d starting\n", id)
    // ... do work ...
    fmt.Printf("Worker %d done\n", id)
}

func main() {
    var wg sync.WaitGroup

    for i := 1; i <= 5; i++ {
        wg.Add(1) // increment counter before launching goroutine
        go worker(i, &wg)
    }

    wg.Wait() // blocks until counter reaches 0
    fmt.Println("All workers done")
}
```

**Rules:**
- Call `wg.Add(1)` BEFORE launching the goroutine
- Call `wg.Done()` at the END of the goroutine (use `defer`)
- Always pass `*sync.WaitGroup` by pointer, never by value

---

## 6. Mutex — Protecting Shared State (22:00)

Use `sync.Mutex` when multiple goroutines need to read/write shared data.

```go
package main

import (
    "fmt"
    "sync"
)

type SafeCounter struct {
    mu sync.Mutex
    count int
}

func (c *SafeCounter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

func main() {
    counter := SafeCounter{}
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Inc()
        }()
    }

    wg.Wait()
    fmt.Println("Final count:", counter.Value()) // always 1000
}
```

**Detect race conditions:**
```bash
go run -race main.go
go test -race ./...
```

**RWMutex — Multiple readers, exclusive writer:**
```go
var rw sync.RWMutex

// Multiple goroutines can hold a read lock simultaneously
rw.RLock()
defer rw.RUnlock()
// ... read shared data ...

// Only one goroutine can hold a write lock at a time
rw.Lock()
defer rw.Unlock()
// ... write shared data ...
```

---

## 7. Select Statement (26:00)

`select` lets a goroutine wait on multiple channel operations — like a `switch` for channels.

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(1 * time.Second)
        ch1 <- "one"
    }()

    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "two"
    }()

    for i := 0; i < 2; i++ {
        select {
        case msg := <-ch1:
            fmt.Println("Received from ch1:", msg)
        case msg := <-ch2:
            fmt.Println("Received from ch2:", msg)
        }
    }
}
```

### Timeout Pattern

```go
func doWork(ch chan string) {
    select {
    case result := <-ch:
        fmt.Println("Got result:", result)
    case <-time.After(2 * time.Second):
        fmt.Println("Timed out!")
    }
}
```

### Non-blocking with Default

```go
select {
case msg := <-ch:
    fmt.Println("Received:", msg)
default:
    fmt.Println("No message ready, moving on")
}
```

---

## 8. Worker Pool Pattern (30:00)

Limits the number of concurrent goroutines processing jobs from a queue.

```go
package main

import (
    "fmt"
    "sync"
)

func workerPool(numWorkers, numJobs int) {
    jobs := make(chan int, numJobs)
    results := make(chan int, numJobs)

    // Launch fixed pool of workers
    var wg sync.WaitGroup
    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for job := range jobs { // workers pull jobs until channel closed
                result := job * job  // simulate work
                fmt.Printf("Worker %d processed job %d -> %d\n", id, job, result)
                results <- result
            }
        }(w)
    }

    // Send jobs
    for j := 1; j <= numJobs; j++ {
        jobs <- j
    }
    close(jobs) // signal workers no more jobs coming

    // Close results after all workers finish
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collect results
    for r := range results {
        fmt.Println("Result:", r)
    }
}

func main() {
    workerPool(3, 9) // 3 workers, 9 jobs
}
```

**Why use a pool?**
- Prevents spawning unlimited goroutines (resource exhaustion)
- Provides back-pressure — jobs queue up if workers are busy
- Controls max concurrency

**Semaphore variant (simpler for one-off limits):**
```go
const maxConcurrent = 5
sem := make(chan struct{}, maxConcurrent)

for _, item := range items {
    sem <- struct{}{}    // acquire slot (blocks when full)
    go func(item Item) {
        defer func() { <-sem }() // release slot
        process(item)
    }(item)
}
```

---

## 9. Fan-Out / Fan-In Pattern (35:00)

**Fan-Out**: Distribute work across multiple goroutines from one channel.
**Fan-In**: Merge results from multiple channels into one.

```go
package main

import (
    "fmt"
    "sync"
)

// Fan-Out: distribute work from one input to multiple workers
func fanOut(input <-chan int, numWorkers int) []<-chan int {
    outputs := make([]<-chan int, numWorkers)
    for i := 0; i < numWorkers; i++ {
        outputs[i] = process(input)
    }
    return outputs
}

func process(input <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for v := range input {
            out <- v * v // do work
        }
    }()
    return out
}

// Fan-In: merge multiple channels into one
func fanIn(inputs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    merged := make(chan int)

    output := func(ch <-chan int) {
        defer wg.Done()
        for v := range ch {
            merged <- v
        }
    }

    wg.Add(len(inputs))
    for _, ch := range inputs {
        go output(ch)
    }

    go func() {
        wg.Wait()
        close(merged)
    }()

    return merged
}

func main() {
    input := make(chan int)

    // Send data
    go func() {
        for i := 1; i <= 9; i++ {
            input <- i
        }
        close(input)
    }()

    // Fan-out to 3 workers, fan-in results
    workerChans := fanOut(input, 3)
    for result := range fanIn(workerChans...) {
        fmt.Println(result)
    }
}
```

---

## 10. Pipeline Pattern (38:00)

Chain stages together where each stage's output is the next stage's input. Each stage runs concurrently.

```go
package main

import "fmt"

// Stage 1: generate numbers
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

// Stage 2: square numbers
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

// Stage 3: filter (only even results)
func filterEven(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            if n%2 == 0 {
                out <- n
            }
        }
    }()
    return out
}

func main() {
    // Pipeline: generate -> square -> filterEven -> print
    nums := generate(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    squared := square(nums)
    evens := filterEven(squared)

    for v := range evens {
        fmt.Println(v) // 4, 16, 36, 64, 100
    }
}
```

---

## 11. Context — Cancellation & Timeout (39:20 / t=2360s)

`context.Context` propagates cancellation, deadlines, and values across goroutines.

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func doWork(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("Worker %d cancelled: %v\n", id, ctx.Err())
            return
        default:
            fmt.Printf("Worker %d working...\n", id)
            time.Sleep(500 * time.Millisecond)
        }
    }
}

func main() {
    // Cancel after 2 seconds
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel() // always defer cancel to avoid goroutine leak

    for i := 1; i <= 3; i++ {
        go doWork(ctx, i)
    }

    time.Sleep(3 * time.Second) // wait to observe cancellation
}
```

### Manual Cancellation

```go
ctx, cancel := context.WithCancel(context.Background())

go func() {
    // Cancel when user presses Ctrl+C or on error
    cancel()
}()

// Pass ctx to goroutines; they check ctx.Done()
```

### Context in Pipelines

```go
func generateWithCtx(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case out <- n:
            case <-ctx.Done(): // stop if cancelled
                return
            }
        }
    }()
    return out
}
```

---

## Common Pitfalls

| Pitfall | Problem | Fix |
|---------|---------|-----|
| Goroutine leak | Goroutine blocks forever, never exits | Use context cancellation or quit channels |
| Race condition | Multiple goroutines read/write shared data | Use mutex or communicate via channels |
| Deadlock | All goroutines waiting, none making progress | Ensure every send has a receiver |
| Closing closed channel | Panic at runtime | Use `sync.Once` or careful ownership |
| Sending on closed channel | Panic at runtime | Only sender should close; use `defer` |
| Copying a mutex | `sync.Mutex` must not be copied | Always pass by pointer |

---

## Quick Reference

```go
// Goroutine
go func() { /* work */ }()

// Channel
ch := make(chan T)           // unbuffered
ch := make(chan T, n)        // buffered, capacity n
ch <- value                  // send (blocks if full/no receiver)
value := <-ch                // receive (blocks if empty/no sender)
close(ch)                    // close (sender only)
for v := range ch { }        // receive until closed

// WaitGroup
var wg sync.WaitGroup
wg.Add(1); go func() { defer wg.Done(); /* work */ }()
wg.Wait()

// Mutex
var mu sync.Mutex
mu.Lock(); defer mu.Unlock()

// Select
select {
case v := <-ch1:
case ch2 <- v:
case <-time.After(d):    // timeout
default:                  // non-blocking
}

// Context
ctx, cancel := context.WithTimeout(context.Background(), d)
defer cancel()
select { case <-ctx.Done(): return }
```

---

## Further Resources

- [Go Concurrency Patterns — Rob Pike (Google I/O 2012)](https://go.dev/talks/2012/concurrency.slide)
- [Advanced Go Concurrency Patterns (2013)](https://go.dev/talks/2013/advconc.slide)
- [The Go Blog: Pipelines and Cancellation](https://go.dev/blog/pipelines)
- [Part 2 of this video series](https://www.youtube.com/watch?v=wELNUHb3kuA)
