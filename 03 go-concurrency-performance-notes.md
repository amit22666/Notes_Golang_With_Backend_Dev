# Improve Go Concurrency Performance With This Pattern — Study Notes

**Source:** [Improve Go Concurrency Performance With This Pattern](https://www.youtube.com/watch?v=Bk1c30avsuU&list=PL7g1jYj15RUNqJStuwE9SCmeOKpgxC0HP&index=3)

> **Prerequisites:** Parts 1 & 2 — goroutines, channels, WaitGroups, mutex, worker pool, errgroup.
> This video focuses on **reducing allocation overhead and GC pressure** in high-throughput concurrent programs, primarily via `sync.Pool`.

---

## Timeline Overview

| Timestamp | Topic |
|-----------|-------|
| 0:00 | The problem — allocations in concurrent programs |
| 3:00 | Benchmarking with `go test -bench` |
| 7:00 | `sync.Pool` — concept and basic usage |
| 14:00 | The allocation trap (values vs pointers) |
| 18:00 | Reset pattern — safe object reuse |
| 22:00 | Real-world example — buffer pooling in HTTP handlers |
| 27:00 | `sync.Pool` internals — per-P architecture |
| 32:00 | False sharing and cache-line padding |
| 35:00 | GC interaction — victim pool mechanism |
| 38:00 | Worker pool sizing — GOMAXPROCS |
| 40:00 | Profiling with pprof and execution tracer |

---

## 1. The Problem — Allocations in Concurrent Programs (0:00)

In high-throughput concurrent code, **frequent heap allocations are the silent killer of performance**:

- Every `make()` / `new()` / struct literal allocates on the heap
- Garbage collector has to track and free all those objects
- GC pauses hurt latency; GC CPU usage hurts throughput
- At scale: millions of short-lived objects per second = constant GC pressure

**The pattern:** instead of allocating → use → discard, we **pool and reuse**.

```
Without pool:           With pool:
alloc → use → GC        get → reset → use → return to pool
alloc → use → GC        get (reuse) → reset → use → return
alloc → use → GC        get (reuse) → ...
```

---

## 2. Benchmarking with `go test -bench` (3:00)

Always measure before and after — don't guess.

```go
// handler_test.go
package main

import (
    "bytes"
    "testing"
)

func BenchmarkWithoutPool(b *testing.B) {
    b.ReportAllocs() // show allocations per op
    for i := 0; i < b.N; i++ {
        buf := new(bytes.Buffer)
        buf.Write([]byte("hello world"))
        _ = buf.Bytes()
    }
}

func BenchmarkWithPool(b *testing.B) {
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        buf := bufPool.Get().(*bytes.Buffer)
        buf.Reset()
        buf.Write([]byte("hello world"))
        _ = buf.Bytes()
        bufPool.Put(buf)
    }
}
```

**Run benchmarks:**
```bash
go test -bench=. -benchmem ./...
go test -bench=. -benchmem -count=5 ./...  # run 5 times for stability
go test -bench=BenchmarkWithPool -benchtime=10s ./...
```

**Reading the output:**
```
BenchmarkWithoutPool-10    1328937    864 ns/op    4096 B/op    1 allocs/op
BenchmarkWithPool-10      28021245     42 ns/op       0 B/op    0 allocs/op
```
- `ns/op` — nanoseconds per operation (lower = faster)
- `B/op` — bytes allocated per operation (0 = pool hit)
- `allocs/op` — heap allocations per operation (0 = perfect)

> **20× throughput improvement** and **zero allocations** after warm-up.

---

## 3. `sync.Pool` — Concept and Basic Usage (7:00)

`sync.Pool` is a **thread-safe cache of temporary objects** that can be retrieved and returned for reuse.

```go
package main

import (
    "bytes"
    "fmt"
    "sync"
)

var bufPool = sync.Pool{
    New: func() any {
        // called only when pool is empty
        return new(bytes.Buffer)
    },
}

func process(data []byte) []byte {
    buf := bufPool.Get().(*bytes.Buffer) // get from pool (or call New)
    defer bufPool.Put(buf)               // always return to pool
    buf.Reset()                          // clear state before use

    buf.Write(data)
    buf.WriteString(" processed")
    result := make([]byte, buf.Len())
    copy(result, buf.Bytes())
    return result
}

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(i int) {
            defer wg.Done()
            result := process([]byte(fmt.Sprintf("item-%d", i)))
            _ = result
        }(i)
    }
    wg.Wait()
}
```

**Key rules:**
1. `Get()` — retrieves an object from the pool; calls `New()` if pool is empty
2. `Put()` — returns an object to the pool for reuse
3. `New` — factory function, called only when the pool is empty
4. Objects in the pool **may be removed by the GC at any time** — always handle nil / use `New`

---

## 4. The Allocation Trap — Values vs Pointers (14:00)

**Critical:** always store **pointers** in the pool, never values.

```go
// WRONG — value type causes heap escape on Put()
var pool = sync.Pool{
    New: func() any { return []byte{} },  // returns a value
}
b := pool.Get().([]byte)
pool.Put(b) // []byte value escapes to heap — allocates!

// CORRECT — pointer type stays on stack path
var pool = sync.Pool{
    New: func() any { return new([]byte) }, // returns a pointer
}
b := pool.Get().(*[]byte)
*b = (*b)[:0]  // reset slice length
pool.Put(b)    // pointer returned — no heap escape
```

**Why this matters:** when you pass a value into `Put()` as `interface{}`, Go wraps it in an interface, which forces the value to escape to the heap — exactly what you're trying to avoid.

**Use escape analysis to verify:**
```bash
go build -gcflags="-m" ./...
# Look for: "does not escape" — good
# Avoid: "escapes to heap" — allocation happening
```

---

## 5. Reset Pattern — Safe Object Reuse (18:00)

Before returning an object to the pool or using a pooled object, **always reset its state**. Failing to reset causes data leaks between requests.

```go
// bytes.Buffer
buf.Reset() // repositions read/write offsets, keeps allocated memory

// strings.Builder
var sb strings.Builder
sb.Reset()

// slices — reset length but keep capacity
s = s[:0]

// custom struct
type Request struct {
    UserID int
    Data   []byte
    Tags   []string
}

func (r *Request) Reset() {
    r.UserID = 0
    r.Data = r.Data[:0]   // keep backing array
    r.Tags = r.Tags[:0]
}

var reqPool = sync.Pool{
    New: func() any { return &Request{} },
}

func handleReq(userID int, data []byte) {
    req := reqPool.Get().(*Request)
    defer func() {
        req.Reset()
        reqPool.Put(req)
    }()

    req.UserID = userID
    req.Data = append(req.Data, data...)
    // process req...
}
```

---

## 6. Real-World Example — Buffer Pooling in HTTP Handlers (22:00)

This is the most common production use of `sync.Pool`.

```go
package main

import (
    "bytes"
    "encoding/json"
    "net/http"
    "sync"
)

var jsonBufPool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

type Response struct {
    Status  string `json:"status"`
    Message string `json:"message"`
}

func handleAPI(w http.ResponseWriter, r *http.Request) {
    buf := jsonBufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        jsonBufPool.Put(buf)
    }()

    resp := Response{Status: "ok", Message: "hello"}
    if err := json.NewEncoder(buf).Encode(resp); err != nil {
        http.Error(w, "encode error", 500)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.Write(buf.Bytes())
}
```

**Without pool:** every request allocates a new `bytes.Buffer` backing array.
**With pool:** buffer backing array is reused — zero allocations after warm-up.

**Standard library examples using sync.Pool:**
- `encoding/json` — pools `encodeState` structs
- `net/http` — pools `bufio.Reader` and `bufio.Writer` (2KB and 4KB variants separately)
- `fmt` — pools `pp` printer objects

```go
// How net/http does it
var (
    bufioReaderPool   sync.Pool
    bufioWriter2kPool sync.Pool  // separate pool per size
    bufioWriter4kPool sync.Pool
)
```

---

## 7. `sync.Pool` Internals — Per-P Architecture (27:00)

Understanding the internals explains why `sync.Pool` is so fast.

### Go's PMG Scheduler Model

```
P (logical processor) — schedules goroutines
M (machine thread)    — OS thread
G (goroutine)         — lightweight coroutine

Only one G runs on one P at a time.
```

### Per-P Local Pool Structure

Instead of one global pool with a lock, each P has its own `poolLocal`:

```go
type poolLocalInternal struct {
    private any       // only THIS P's goroutine can access — zero contention
    shared  poolChain // other Ps can steal from here (lock-free CAS)
}

type poolLocal struct {
    poolLocalInternal
    // padding to fill a full cache line — prevents false sharing
    pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
```

### `Get()` search sequence

```
1. Check current P's private slot  → fastest, zero contention
2. Pop from current P's shared chain head
3. Steal from OTHER Ps' shared chain tails  (work stealing)
4. Check victim pool (objects from previous GC cycle)
5. Call New()  → slowest, allocates
```

### `Put()` sequence

```
1. Store in current P's private slot (if empty)
2. Push to current P's shared chain head
```

```go
// Simplified Put() and Get() logic
func (p *Pool) Put(x any) {
    l, _ := p.pin()           // disable preemption, get current P's pool
    if l.private == nil {
        l.private = x         // fast path: store in private slot
    } else {
        l.shared.pushHead(x)  // slow path: push to shared chain
    }
    runtime_procUnpin()
}

func (p *Pool) Get() any {
    l, pid := p.pin()
    x := l.private            // fast path: take from private slot
    l.private = nil
    if x == nil {
        x, _ = l.shared.popHead()
        if x == nil {
            x = p.getSlow(pid) // steal from other Ps or victim pool
        }
    }
    runtime_procUnpin()
    if x == nil && p.New != nil {
        x = p.New()           // last resort: allocate
    }
    return x
}
```

---

## 8. False Sharing and Cache-Line Padding (32:00)

**False sharing** is a hidden performance killer in concurrent code.

### What is false sharing?

Modern CPUs load memory in **cache lines** (typically 64–128 bytes). If two goroutines on different cores write to different variables that happen to share the same cache line, every write **invalidates the other core's cache** — even though they're touching different data.

```
Cache line (64 bytes):
[CounterA (8 bytes)][CounterB (8 bytes)][..padding..48 bytes..]

Core 1 writes CounterA → invalidates Core 2's cache line
Core 2 writes CounterB → invalidates Core 1's cache line
Result: constant cache invalidation, terrible performance
```

### Solution — Padding

```go
// BEFORE — false sharing: A and B share a cache line
type Counters struct {
    A int64
    B int64
}

// AFTER — padded: A and B each occupy their own cache line
type PaddedCounter struct {
    value int64
    _     [56]byte // pad to 64 bytes total
}

type Counters struct {
    A PaddedCounter
    B PaddedCounter
}
```

**How `sync.Pool` prevents this:**
```go
type poolLocal struct {
    poolLocalInternal
    pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte // 128-byte aligned
}
// Each P's poolLocal is on its own cache line — no false sharing between Ps
```

### Detecting false sharing

```bash
# Linux perf
perf stat -e cache-misses,cache-references ./myapp

# Or use pprof to see high contention
go tool pprof -alloc_objects mem.prof
```

---

## 9. GC Interaction — Victim Pool Mechanism (35:00)

`sync.Pool` objects are **not permanent**. The GC can clear them.

### Two-cycle cleanup

```
GC Cycle 1:
  allPools (active) → moved to oldPools (victims)
  oldPools (old victims) → discarded

GC Cycle 2:
  Objects that survived GC 1 are now "victims"
  Available for Get() before calling New()
```

```go
// Simplified poolCleanup() called by GC
func poolCleanup() {
    for _, p := range oldPools {
        p.victim = nil       // discard last GC cycle's objects
    }
    for _, p := range allPools {
        p.victim = p.local   // demote current objects to victims
        p.local = nil        // clear active pools
    }
    oldPools, allPools = allPools, nil
}
```

**Why this matters for your code:**
- Objects survive **at least 2 GC cycles** — provides a grace period
- After every GC, expect a brief spike in `New()` calls — the pool is "warming up" again
- For latency-sensitive code, **pre-warm the pool**:

```go
func init() {
    // Pre-populate pool at startup to avoid cold-start allocations
    for i := 0; i < 100; i++ {
        bufPool.Put(new(bytes.Buffer))
    }
}
```

---

## 10. Worker Pool Sizing — GOMAXPROCS (38:00)

For the **worker pool pattern** from Part 1, getting the pool size right is critical.

```go
import "runtime"

// CPU-bound tasks — match to CPU count
numWorkers := runtime.GOMAXPROCS(0) // 0 = query current value
// or
numWorkers := runtime.NumCPU()

// I/O-bound tasks — can exceed CPU count (workers spend time waiting)
numWorkers := runtime.GOMAXPROCS(0) * 4
```

### Why over-sizing hurts CPU-bound work

```
With 10 CPUs and 100 workers (CPU-bound):
- 10 workers run, 90 are runnable
- Scheduler constantly context-switches
- Cache locality destroyed (goroutines migrate between cores)
- More goroutines = more memory even when idle

With 10 CPUs and 10 workers (CPU-bound):
- All workers run, no context switching needed
- Each worker "owns" a core → cache stays hot
- 28% faster, 50% less memory (benchmark result)
```

```go
// Adaptive pool sizing
func optimalWorkers(taskType string) int {
    cpus := runtime.GOMAXPROCS(0)
    switch taskType {
    case "cpu":
        return cpus
    case "io":
        return cpus * 4
    case "mixed":
        return cpus * 2
    default:
        return cpus
    }
}
```

### Setting GOMAXPROCS

```go
// In main() or init() for containerized environments
import "runtime"

func main() {
    // By default Go uses all available CPUs
    // In containers, may need to set manually based on CPU quota
    runtime.GOMAXPROCS(4) // limit to 4 logical processors
    // ...
}
```

---

## 11. Profiling — pprof and Execution Tracer (40:00)

### CPU and memory profiling with pprof

```go
package main

import (
    "net/http"
    _ "net/http/pprof" // registers /debug/pprof endpoints
    "runtime/pprof"
    "os"
)

func main() {
    // Option 1: HTTP endpoint (for long-running servers)
    go http.ListenAndServe(":6060", nil)

    // Option 2: File-based (for benchmarks/scripts)
    cpuFile, _ := os.Create("cpu.prof")
    pprof.StartCPUProfile(cpuFile)
    defer pprof.StopCPUProfile()

    memFile, _ := os.Create("mem.prof")
    defer pprof.WriteHeapProfile(memFile)

    runApp()
}
```

**Analyze profiles:**
```bash
# CPU profile
go tool pprof cpu.prof
(pprof) top10
(pprof) web          # open flame graph in browser

# Memory/allocation profile
go tool pprof -alloc_objects mem.prof
go tool pprof -inuse_space   mem.prof

# Via HTTP (server must be running)
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
go tool pprof http://localhost:6060/debug/pprof/heap
```

### Execution Tracer — see what goroutines are doing

```go
import (
    "os"
    "runtime/trace"
)

func main() {
    f, _ := os.Create("trace.out")
    trace.Start(f)
    defer trace.Stop()

    runApp()
}
```

```bash
go tool trace trace.out
# Opens browser with timeline view
```

**What to look for in the trace:**
| Color | Meaning |
|-------|---------|
| Gold | Goroutine executing |
| Blue | Network wait |
| Red | GC pause |
| Gray | Blocked / runnable but waiting for CPU |

**Annotate your code for the tracer:**
```go
import "runtime/trace"

func handleRequest(ctx context.Context) {
    ctx, task := trace.NewTask(ctx, "handleRequest")
    defer task.End()

    trace.WithRegion(ctx, "fetchData", func() {
        fetchData()
    })

    trace.WithRegion(ctx, "processData", func() {
        processData()
    })
}
```

### Race detector

```bash
go run -race main.go
go test -race ./...
```

---

## Full Example — Putting It Together

An HTTP server using `sync.Pool`, worker pool, and pprof:

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
    _ "net/http/pprof"
    "runtime"
    "sync"
)

// Pool for JSON response buffers
var respPool = sync.Pool{
    New: func() any { return new(bytes.Buffer) },
}

// Job queue and worker pool
type Job struct {
    ID   int
    Data string
}

type Result struct {
    ID     int
    Output string
}

func startWorkerPool(numWorkers int) (chan<- Job, <-chan Result) {
    jobs := make(chan Job, 100)
    results := make(chan Result, 100)

    for i := 0; i < numWorkers; i++ {
        go func() {
            for job := range jobs {
                // Get pooled buffer for intermediate work
                buf := respPool.Get().(*bytes.Buffer)
                buf.Reset()
                fmt.Fprintf(buf, "processed: %s", job.Data)
                output := buf.String()
                buf.Reset()
                respPool.Put(buf)

                results <- Result{ID: job.ID, Output: output}
            }
        }()
    }
    return jobs, results
}

func main() {
    numWorkers := runtime.GOMAXPROCS(0) // match CPU count for CPU-bound
    jobs, results := startWorkerPool(numWorkers)

    http.HandleFunc("/process", func(w http.ResponseWriter, r *http.Request) {
        jobs <- Job{ID: 1, Data: r.URL.Query().Get("data")}
        result := <-results

        // Get pooled buffer for response encoding
        buf := respPool.Get().(*bytes.Buffer)
        defer func() {
            buf.Reset()
            respPool.Put(buf)
        }()

        json.NewEncoder(buf).Encode(result)
        w.Header().Set("Content-Type", "application/json")
        w.Write(buf.Bytes())
    })

    // pprof available at http://localhost:6060/debug/pprof/
    http.ListenAndServe(":8080", nil)
}
```

---

## Performance Impact Summary

| Technique | What it fixes | Typical gain |
|-----------|--------------|-------------|
| `sync.Pool` for buffers | Heap allocations, GC pressure | 20× throughput, 0 allocs/op |
| Cache-line padding | False sharing between cores | Eliminates cache invalidation storms |
| Optimal worker count | Scheduler contention | ~28% faster, 50% less memory |
| Pointer-only pool storage | Heap escape on Put | Eliminates accidental allocs |
| Pre-warming pool | Cold-start latency | Eliminates initial allocation spikes |

---

## When to Use `sync.Pool` (Decision Guide)

**Use it when:**
- Objects are allocated > 1,000/sec
- Objects are short-lived (used within a request/goroutine lifetime)
- Object creation is non-trivial (large backing arrays, populated structs)
- You can safely reset state between uses
- You've measured and confirmed allocation overhead

**Don't use it for:**
- Database connections (use a dedicated connection pool)
- Long-lived objects (hours/days lifespan)
- Objects with complex lifecycle (references held after Put)
- Low-frequency allocations (< 10/sec — complexity not worth it)
- Goroutines themselves (Go runtime already manages goroutine reuse)

---

## Quick Reference

```bash
# Benchmark
go test -bench=. -benchmem ./...
go test -bench=BenchmarkFoo -benchmem -count=5 ./...

# Escape analysis
go build -gcflags="-m" ./...
go build -gcflags="-m -m" ./...   # more verbose

# Race detector
go run -race ./...
go test -race ./...

# CPU profile
go tool pprof cpu.prof

# Memory profile
go tool pprof -alloc_objects mem.prof
go tool pprof -inuse_space mem.prof

# Execution trace
go tool trace trace.out

# GOMAXPROCS
runtime.GOMAXPROCS(0)   // query
runtime.GOMAXPROCS(n)   // set
runtime.NumCPU()        // physical CPUs
```

```go
// sync.Pool template
var myPool = sync.Pool{
    New: func() any { return &MyObject{} },
}

obj := myPool.Get().(*MyObject)
defer func() {
    obj.Reset()
    myPool.Put(obj)
}()

// Generic wrapper (Go 1.18+)
type Pool[T any] struct{ p sync.Pool }
func (p *Pool[T]) Get() *T  { return p.p.Get().(*T) }
func (p *Pool[T]) Put(v *T) { p.p.Put(v) }
```

---

## Further Resources

- [Go sync.Pool Mechanics — VictoriaMetrics](https://victoriametrics.com/blog/go-sync-pool/)
- [Object Pooling — Go Optimization Guide](https://goperf.dev/01-common-patterns/object-pooling/)
- [Worker Pool Sizing — Go Optimization Guide](https://goperf.dev/01-common-patterns/worker-pool/)
- [Go Execution Tracer](https://pkg.go.dev/runtime/trace)
- [pprof profiling](https://pkg.go.dev/net/http/pprof)
- [Part 1 Notes](./go-concurrency-patterns-notes.md)
- [Part 2 Notes](./go-concurrency-patterns-part2-notes.md)
