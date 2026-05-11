# Learn the Go Concurrency Pattern That Blew My Mind — Study Notes

**Source:** [learn the Go concurrency pattern that blew my mind](https://www.youtube.com/watch?v=bnbEULxcX3o&list=PL7g1jYj15RUNqJStuwE9SCmeOKpgxC0HP&index=4)

> **Prerequisites:** Parts 1–3 — goroutines, channels, select, WaitGroups, mutex, sync.Pool.
> This video covers advanced channel orchestration patterns that go beyond the basics.

---

## Timeline Overview

| Timestamp | Topic |
|-----------|-------|
| 0:00 | The problem with naive concurrent code |
| 3:00 | The `for-select` loop idiom |
| 7:00 | **The nil channel trick** — the mind-blowing pattern |
| 13:00 | Or-Done channel — wrapping cancellation |
| 19:00 | Tee channel — splitting a stream |
| 24:00 | Bridge channel — flattening a channel of channels |
| 29:00 | Confinement — ownership without locks |
| 33:00 | Heartbeat pattern |
| 37:00 | Service / reply channel pattern |
| 40:00 | Putting it all together — subscription service |

---

## 1. The Problem with Naive Concurrent Code (0:00)

Naive goroutine code has subtle bugs that compound at scale:

```go
// BUG 1: No cancellation — goroutine leaks forever
func fetchForever() {
    go func() {
        for {
            result := slowFetch() // what if caller stops caring?
            process(result)
        }
    }()
}

// BUG 2: Sleep blocks — can't cancel mid-sleep
func retry() {
    for {
        if err := doWork(); err != nil {
            time.Sleep(10 * time.Second) // stuck here — can't cancel
        }
    }
}

// BUG 3: Blocking send — goroutine hangs if receiver exits
func produce(out chan<- int) {
    for i := 0; ; i++ {
        out <- i // blocks forever if nobody receives
    }
}
```

All three bugs share one root cause: **goroutines that cannot be signalled to stop**.

The patterns in this video solve this systematically.

---

## 2. The `for-select` Loop (3:00)

The foundation of all advanced channel patterns. A goroutine runs a loop, and each iteration uses `select` to react to whichever channel is ready.

```go
func loop(done <-chan struct{}, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for {
            select {
            case <-done:
                return        // cancelled — exit cleanly
            case v, ok := <-in:
                if !ok {
                    return    // input closed — exit cleanly
                }
                select {
                case out <- v * 2:
                case <-done:
                    return
                }
            }
        }
    }()
    return out
}
```

**Rules of the `for-select` loop:**
1. Always have a `<-done` case — never block without a way out
2. Check `ok` when ranging over input channels — closed channel returns zero value + false
3. Both receive AND send need `<-done` protection — either can block

---

## 3. The Nil Channel Trick — The Mind-Blowing Pattern (7:00)

> **The insight:** A nil channel **blocks forever**. A `select` case on a nil channel is **never selected**. You can use this to dynamically enable and disable `select` cases.

### The key rule

```go
var ch chan int // nil channel

ch <- 1        // blocks forever — never completes
v := <-ch      // blocks forever — never completes

select {
case v := <-ch: // NEVER selected when ch == nil
    fmt.Println(v)
case <-time.After(time.Second):
    fmt.Println("timeout") // this fires instead
}
```

### Using nil to enable/disable cases

```go
func dedupeStream(done <-chan struct{}, in <-chan Item) <-chan Item {
    out := make(chan Item)
    go func() {
        defer close(out)
        var pending Item
        var outEnabled chan Item // starts nil — send case DISABLED

        for {
            select {
            case <-done:
                return

            case v, ok := <-in: // receive case — always enabled
                if !ok {
                    return
                }
                if !alreadySeen(v) {
                    pending = v
                    in = nil           // DISABLE receive (don't read more until sent)
                    outEnabled = out   // ENABLE send
                }

            case outEnabled <- pending: // send case — nil until we have data
                in = in                // RE-ENABLE receive (restore original)
                outEnabled = nil       // DISABLE send
            }
        }
    }()
    return out
}
```

**What just happened:**
- `in = nil` → receive case is disabled (blocks forever → never selected)
- `outEnabled = out` → send case is now enabled
- After sending: `in` restored, `outEnabled` set to nil again
- This creates a **toggle** between receive-mode and send-mode with ZERO extra synchronization

### Real-world example: rate-limited fetcher

```go
type fetchResult struct {
    items []Item
    next  time.Time
    err   error
}

func (s *sub) loop() {
    var fetchDone chan fetchResult // nil = no fetch running
    var pending []Item
    var next time.Time

    for {
        // Enable fetch timer only if no fetch is in-flight AND queue isn't full
        var startFetch <-chan time.Time
        if fetchDone == nil && len(pending) < maxPending {
            delay := time.Until(next)
            if delay < 0 {
                delay = 0
            }
            startFetch = time.After(delay) // enable fetch case
        }
        // startFetch == nil if we don't want to fetch → case never fires

        // Enable send only if we have items to send
        var first Item
        var updates chan Item // nil = send disabled
        if len(pending) > 0 {
            first = pending[0]
            updates = s.updates // enable send case
        }

        select {
        case errc := <-s.closing:
            errc <- nil
            close(s.updates)
            return

        case <-startFetch: // only fires when startFetch != nil
            fetchDone = make(chan fetchResult, 1)
            go func() {
                items, next, err := s.fetcher.Fetch()
                fetchDone <- fetchResult{items, next, err}
            }()

        case result := <-fetchDone: // only fires when fetchDone != nil
            fetchDone = nil
            if result.err == nil {
                pending = append(pending, result.items...)
            }
            next = result.next

        case updates <- first: // only fires when updates != nil (pending > 0)
            pending = pending[1:]
        }
    }
}
```

**This single loop handles:**
- Rate-limited fetching
- Backpressure (don't fetch if queue is full)
- Non-blocking sends
- Graceful shutdown
- Zero mutexes, zero race conditions

---

## 4. Or-Done Channel — Wrapping Cancellation (13:00)

When you `range` over a channel and need it to stop when `done` fires, you need a select inside every loop body — that's verbose. **Or-Done wraps this cleanly.**

### The problem

```go
// Verbose — need this pattern everywhere
for {
    select {
    case <-done:
        return
    case v, ok := <-ch:
        if !ok {
            return
        }
        doSomething(v)
    }
}
```

### Or-Done wraps it once

```go
func orDone(done <-chan struct{}, in <-chan interface{}) <-chan interface{} {
    out := make(chan interface{})
    go func() {
        defer close(out)
        for {
            select {
            case <-done:
                return
            case v, ok := <-in:
                if !ok {
                    return
                }
                select {
                case out <- v:
                case <-done:
                    return
                }
            }
        }
    }()
    return out
}
```

**Now you can `range` cleanly:**

```go
// BEFORE (verbose)
for {
    select {
    case <-done:
        return
    case v, ok := <-ch:
        if !ok { return }
        process(v)
    }
}

// AFTER (clean)
for v := range orDone(done, ch) {
    process(v)
}
```

### Or-Channel — any signal closes the output

Combine N done channels — output closes when ANY input closes:

```go
func orChannel(channels ...<-chan struct{}) <-chan struct{} {
    switch len(channels) {
    case 0:
        return nil
    case 1:
        return channels[0]
    }

    orDone := make(chan struct{})
    go func() {
        defer close(orDone)
        switch len(channels) {
        case 2:
            select {
            case <-channels[0]:
            case <-channels[1]:
            }
        default:
            select {
            case <-channels[0]:
            case <-channels[1]:
            case <-channels[2]:
            // Recurse for remaining channels, passing orDone so
            // the recursive goroutine exits when we do
            case <-orChannel(append(channels[3:], orDone)...):
            }
        }
    }()
    return orDone
}
```

**Usage:**

```go
sig := func(after time.Duration) <-chan struct{} {
    c := make(chan struct{})
    go func() {
        defer close(c)
        time.Sleep(after)
    }()
    return c
}

// Fires after the FIRST of these durations
start := time.Now()
<-orChannel(
    sig(2*time.Hour),
    sig(5*time.Minute),
    sig(1*time.Second), // ← this fires first
    sig(1*time.Hour),
)
fmt.Printf("done after %v\n", time.Since(start)) // ~1 second
```

---

## 5. Tee Channel — Splitting a Stream (19:00)

Like the Unix `tee` command — one input, two outputs. Both outputs receive every value.

```go
func tee(done <-chan struct{}, in <-chan interface{}) (<-chan interface{}, <-chan interface{}) {
    out1 := make(chan interface{})
    out2 := make(chan interface{})
    go func() {
        defer close(out1)
        defer close(out2)
        for v := range orDone(done, in) {
            // Use local vars so we can nil them out after sending
            var c1, c2 = out1, out2
            // Send to BOTH before moving to next value
            // Select ensures we don't deadlock if one is slow
            for i := 0; i < 2; i++ {
                select {
                case c1 <- v:
                    c1 = nil // sent to out1, disable this case
                case c2 <- v:
                    c2 = nil // sent to out2, disable this case
                }
            }
        }
    }()
    return out1, out2
}
```

**Usage:**

```go
done := make(chan struct{})
defer close(done)

in := generate(done, 1, 2, 3, 4, 5)
log, process := tee(done, in)

go func() {
    for v := range log {
        fmt.Println("LOG:", v) // receives every value
    }
}()

for v := range process {
    fmt.Println("PROCESS:", v) // also receives every value
}
```

**The nil trick again:** `c1 = nil` and `c2 = nil` disable cases after they've been satisfied for the current value, so the loop runs exactly twice per value.

---

## 6. Bridge Channel — Flattening a Channel of Channels (24:00)

When a pipeline stage produces a *sequence of channels* (e.g., one channel per batch), bridge flattens them into a single stream.

```go
// chanStream is a channel that produces channels
func bridge(done <-chan struct{}, chanStream <-chan (<-chan interface{})) <-chan interface{} {
    out := make(chan interface{})
    go func() {
        defer close(out)
        for {
            var stream <-chan interface{}
            select {
            case maybeStream, ok := <-chanStream:
                if !ok {
                    return // no more channels coming
                }
                stream = maybeStream
            case <-done:
                return
            }
            // Drain this channel before pulling the next one
            for v := range orDone(done, stream) {
                select {
                case out <- v:
                case <-done:
                    return
                }
            }
        }
    }()
    return out
}
```

**Usage:**

```go
// A function that returns a sequence of channels (e.g., paginated API)
func genChannels(done <-chan struct{}) <-chan (<-chan interface{}) {
    chanStream := make(chan (<-chan interface{}))
    go func() {
        defer close(chanStream)
        for i := 0; i < 3; i++ {
            stream := make(chan interface{}, 3)
            stream <- fmt.Sprintf("batch-%d-item-1", i)
            stream <- fmt.Sprintf("batch-%d-item-2", i)
            stream <- fmt.Sprintf("batch-%d-item-3", i)
            close(stream)
            chanStream <- stream
        }
    }()
    return chanStream
}

done := make(chan struct{})
defer close(done)

for v := range bridge(done, genChannels(done)) {
    fmt.Println(v)
}
// batch-0-item-1, batch-0-item-2, ..., batch-2-item-3
```

---

## 7. Confinement — Ownership Without Locks (29:00)

The safest concurrent code is code where **only one goroutine ever touches the data**. No sharing = no synchronization needed.

### Lexical confinement — compiler enforces it

```go
// The slice is created inside, passed as read-only channel — confined by design
func producer() <-chan int {
    data := []int{1, 2, 3, 4, 5} // only this goroutine writes to data
    out := make(chan int, len(data))
    go func() {
        defer close(out)
        for _, v := range data {
            out <- v
        }
        // data goes out of scope here — nobody else can touch it
    }()
    return out // caller only gets the channel, not the slice
}

func main() {
    for v := range producer() {
        fmt.Println(v) // read-only access via channel
    }
}
```

### Confinement with a worker pool — each worker owns its own buffer

```go
func process(done <-chan struct{}, jobs <-chan []byte) <-chan []byte {
    out := make(chan []byte)
    go func() {
        defer close(out)
        // Each goroutine has its OWN buffer — no sharing, no mutex
        buf := make([]byte, 0, 4096)
        for job := range orDone(done, jobs) {
            buf = transform(buf[:0], job) // reuse buf — only this goroutine touches it
            result := make([]byte, len(buf))
            copy(result, buf)
            select {
            case out <- result:
            case <-done:
                return
            }
        }
    }()
    return out
}
```

**Why this is better than mutex:** The compiler and ownership model make concurrent access impossible by construction, not by convention.

---

## 8. Heartbeat Pattern (33:00)

A goroutine periodically sends a signal proving it's alive. Useful for:
- Health monitoring
- Testing that long-running goroutines haven't deadlocked
- Detecting slow goroutines in test suites

### Interval-based heartbeat

```go
func withHeartbeat(done <-chan struct{}, interval time.Duration) (<-chan struct{}, <-chan interface{}) {
    heartbeat := make(chan struct{}, 1)
    results := make(chan interface{})

    go func() {
        defer close(heartbeat)
        defer close(results)

        pulse := time.NewTicker(interval)
        defer pulse.Stop()

        sendPulse := func() {
            select {
            case heartbeat <- struct{}{}:
            default: // non-blocking — skip if nobody is listening
            }
        }

        for {
            select {
            case <-done:
                return
            case <-pulse.C:
                sendPulse()
            case results <- doWork():
                sendPulse() // also pulse on every result
            }
        }
    }()

    return heartbeat, results
}
```

### Using heartbeat in tests to detect hangs

```go
func TestLongRunningProcess(t *testing.T) {
    done := make(chan struct{})
    defer close(done)

    heartbeat, results := withHeartbeat(done, 10*time.Millisecond)
    timeout := time.After(2 * time.Second)

    for {
        select {
        case <-timeout:
            t.Fatal("goroutine appears to be deadlocked")
        case _, ok := <-heartbeat:
            if !ok {
                t.Log("goroutine finished")
                return
            }
            t.Log("goroutine still alive") // visible in -v mode
        case result, ok := <-results:
            if !ok {
                return
            }
            t.Log("got result:", result)
        }
    }
}
```

---

## 9. Service / Reply Channel Pattern (37:00)

When you need **request-response** semantics between goroutines — the caller sends a request and blocks until the goroutine responds.

```go
type closeRequest struct {
    reply chan error // caller-provided channel for the response
}

type Service struct {
    in      chan int
    closing chan closeRequest
}

func NewService() *Service {
    s := &Service{
        in:      make(chan int),
        closing: make(chan closeRequest),
    }
    go s.run()
    return s
}

func (s *Service) run() {
    var last int
    for {
        select {
        case v := <-s.in:
            last = v
            fmt.Println("processed:", v)

        case req := <-s.closing:
            // Clean up, then respond
            fmt.Println("shutting down, last value:", last)
            req.reply <- nil // send response back to caller
            return
        }
    }
}

func (s *Service) Send(v int) {
    s.in <- v
}

func (s *Service) Close() error {
    req := closeRequest{reply: make(chan error)}
    s.closing <- req  // send request
    return <-req.reply // wait for acknowledgement
}

func main() {
    svc := NewService()
    svc.Send(1)
    svc.Send(2)
    svc.Send(3)
    fmt.Println("close error:", svc.Close()) // blocks until goroutine confirms close
}
```

**Why this pattern:**
- `Close()` doesn't return until the goroutine has actually cleaned up
- No mutex needed — only the goroutine touches state
- Caller gets the actual error from cleanup (not just a "close was called" signal)

---

## 10. Full Example — Subscription Service (40:00)

Combining `for-select`, nil channels, done channels, and the service pattern into a real-world feed aggregator:

```go
package main

import (
    "fmt"
    "time"
)

type Item struct {
    GUID    string
    Title   string
    Channel string
}

type Fetcher interface {
    Fetch() (items []Item, next time.Time, err error)
}

type Subscription interface {
    Updates() <-chan Item
    Close() error
}

type sub struct {
    fetcher Fetcher
    updates chan Item
    closing chan chan error
}

func Subscribe(fetcher Fetcher) Subscription {
    s := &sub{
        fetcher: fetcher,
        updates: make(chan Item),
        closing: make(chan chan error),
    }
    go s.loop()
    return s
}

func (s *sub) Updates() <-chan Item { return s.updates }

func (s *sub) Close() error {
    errc := make(chan error)
    s.closing <- errc // request shutdown
    return <-errc     // wait for confirmation
}

func (s *sub) loop() {
    const maxPending = 10

    type fetchResult struct {
        items []Item
        next  time.Time
        err   error
    }

    var (
        fetchDone chan fetchResult // nil = no fetch running
        pending   []Item
        next      time.Time
        seen      = make(map[string]bool)
    )

    for {
        // --- nil channel trick: dynamically enable/disable cases ---

        // Enable fetch timer only if not already fetching and queue has room
        var startFetch <-chan time.Time
        if fetchDone == nil && len(pending) < maxPending {
            delay := time.Until(next)
            if delay < 0 {
                delay = 0
            }
            startFetch = time.After(delay) // non-nil → case enabled
        }
        // startFetch == nil → fetch case never fires

        // Enable send only if we have items queued
        var updates chan Item
        var first Item
        if len(pending) > 0 {
            updates = s.updates // non-nil → send case enabled
            first = pending[0]
        }
        // updates == nil → send case never fires

        select {
        case errc := <-s.closing:
            errc <- nil
            close(s.updates)
            return

        case <-startFetch:
            fetchDone = make(chan fetchResult, 1)
            go func() {
                items, next, err := s.fetcher.Fetch()
                fetchDone <- fetchResult{items, next, err}
            }()

        case result := <-fetchDone:
            fetchDone = nil // re-enable startFetch next iteration
            next = result.next
            if result.err == nil {
                for _, item := range result.items {
                    if !seen[item.GUID] {
                        pending = append(pending, item)
                        seen[item.GUID] = true
                    }
                }
            }

        case updates <- first:
            pending = pending[1:] // shift queue
        }
    }
}
```

---

## Pattern Summary

| Pattern | The trick | Solves |
|---------|-----------|--------|
| `for-select` loop | `select` inside infinite loop | Single event-loop goroutine |
| **Nil channel** | `nil` case never fires in `select` | Dynamically enable/disable cases |
| Or-Done | Wrap `for { select { done/ch } }` | Clean `range` over cancellable channels |
| Or-Channel | Recursive `select` across N channels | "Any of these signals fires → done" |
| Tee | `c1 = nil` after sending | Duplicate stream to two consumers |
| Bridge | Drain inner channels sequentially | Flatten `chan (<-chan T)` to `chan T` |
| Confinement | Single owner, read-only channel out | Safe concurrency without any lock |
| Heartbeat | `Ticker` + non-blocking send | Goroutine liveness monitoring |
| Service/Reply | Embedded reply channel in request | Request-response between goroutines |

---

## Common Bugs Fixed by These Patterns

```go
// BUG: goroutine leaks — no way to stop
go func() {
    for v := range ch { process(v) }
}()
// FIX: use orDone or for-select with done channel

// BUG: time.Sleep blocks cancellation
time.Sleep(10 * time.Second)
// FIX: use time.After in select
select {
case <-time.After(10 * time.Second):
case <-done:
    return
}

// BUG: blocking send on closed receiver
ch <- value // blocks if receiver exited
// FIX: always pair send with done
select {
case ch <- value:
case <-done:
    return
}

// BUG: multiple goroutines read same data = race
data := sharedSlice
go func() { data[0] = 1 }()
go func() { data[0] = 2 }()
// FIX: confinement — each goroutine owns its own data
```

---

## Quick Reference — The Nil Channel Trick

```go
// RULE: nil channel blocks forever → never selected in select

var ch chan int  // nil

// Enable a case:   ch = realChannel
// Disable a case:  ch = nil

// Pattern: toggle between receive and send
var out chan T  // nil = send disabled

case v := <-in:
    pending = v
    in = nil   // disable receive
    out = outChan // enable send

case out <- pending:
    in = inChan // re-enable receive
    out = nil   // disable send
```

---

## Further Resources

- [Go Concurrency Patterns: Pipelines and Cancellation — go.dev](https://go.dev/blog/pipelines)
- [Advanced Go Concurrency Patterns (2013) — Rob Pike](https://go.dev/talks/2013/advconc.slide)
- [Go Concurrency Patterns: or-done, tee, bridge — opcito.com](https://www.opcito.com/blogs/practical-concurrency-patterns-in-go)
- [Part 1 Notes](./go-concurrency-patterns-notes.md)
- [Part 2 Notes](./go-concurrency-patterns-part2-notes.md)
- [Part 3 Notes](./go-concurrency-performance-notes.md)
