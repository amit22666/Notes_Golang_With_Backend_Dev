# Go Concurrency Patterns — Part 2 Study Notes

**Source:** [Master Go Programming With These Concurrency Patterns | Part 2 (in 40 minutes)](https://www.youtube.com/watch?v=wELNUHb3kuA&list=PL7g1jYj15RUNqJStuwE9SCmeOKpgxC0HP&index=2)

> **Prerequisites:** Part 1 — goroutines, channels, WaitGroups, mutex, select, worker pool, pipelines, context.
> Part 2 covers the advanced/extended `sync` and `golang.org/x/sync` ecosystem.

---

## Timeline Overview

| Timestamp | Topic |
|-----------|-------|
| 0:00 | Recap & what Part 2 covers |
| 2:00 | `sync.Once` — run exactly once |
| 8:00 | `sync/atomic` — lock-free operations |
| 14:00 | `errgroup` — goroutines with error propagation |
| 20:00 | `errgroup.WithContext` — cancel on first error |
| 25:00 | `singleflight` — deduplicating concurrent calls |
| 30:00 | Bounded concurrency — channel semaphore |
| 34:00 | Weighted semaphore — variable-cost tasks |
| 37:00 | Rate limiting — token/leaky bucket |
| 39:00 | Pub-Sub pattern |

---

## 1. `sync.Once` — Run Exactly Once (2:00)

Use `sync.Once` when you need something initialized only once, no matter how many goroutines hit it simultaneously. Classic use: database connections, loggers, config loading.

### Basic usage

```go
package main

import (
    "fmt"
    "sync"
)

var (
    once   sync.Once
    config Config
)

func GetConfig() Config {
    once.Do(func() {
        fmt.Println("Loading config (only printed once)...")
        config = fetchConfig()
    })
    return config
}

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            cfg := GetConfig()
            _ = cfg
        }()
    }
    wg.Wait()
    // "Loading config..." printed exactly once despite 10 goroutines
}
```

### Behaviour rules

```go
var once sync.Once

once.Do(func() { fmt.Println("first") })   // prints "first"
once.Do(func() { fmt.Println("second") })  // does nothing — Once is spent
```

- `sync.Once` cannot be reused; create a new one if you need to re-run.
- Error state is also cached — if the function panics, future calls also panic.

### Go 1.21+ helpers

```go
// OnceFunc — wraps any func() for single execution
var initLogger = sync.OnceFunc(func() {
    // expensive logger setup
})

// OnceValue — caches the return value
var getDB = sync.OnceValue(func() *sql.DB {
    db, _ := sql.Open("postgres", dsn)
    return db
})
db := getDB() // executes once, subsequent calls return cached *sql.DB

// OnceValues — caches two return values (great for error-returning inits)
var loadCerts = sync.OnceValues(func() (tls.Certificate, error) {
    return tls.LoadX509KeyPair("cert.pem", "key.pem")
})
cert, err := loadCerts() // safe to call from many goroutines
```

### How `sync.Once` works internally

```go
type Once struct {
    done atomic.Uint32 // checked first (fast path, stays in CPU cache)
    m    Mutex
}

func (o *Once) Do(f func()) {
    if o.done.Load() == 0 { // fast path — no lock needed if already done
        o.doSlow(f)
    }
}

func (o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done.Load() == 0 {
        defer o.done.Store(1) // AFTER f() completes — prevents partial init
        f()
    }
}
```

**Why `defer o.done.Store(1)` not before `f()`?** Other goroutines would see `done=1` while `f()` is still running, potentially reading a partially initialized value.

---

## 2. `sync/atomic` — Lock-Free Operations (8:00)

Atomic operations read/modify shared integers without a mutex. Cheaper than a lock for simple counters and flags.

### Counter (the most common case)

```go
package main

import (
    "fmt"
    "sync"
    "sync/atomic"
)

func main() {
    var counter int64
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            atomic.AddInt64(&counter, 1) // safe — no mutex needed
        }()
    }

    wg.Wait()
    fmt.Println("Counter:", counter) // always 1000
}
```

### All atomic operations

```go
var val int64

// Read
n := atomic.LoadInt64(&val)

// Write
atomic.StoreInt64(&val, 42)

// Add (returns new value)
newVal := atomic.AddInt64(&val, 1)   // increment
newVal  = atomic.AddInt64(&val, -1)  // decrement

// Swap — stores new, returns old
old := atomic.SwapInt64(&val, 100)

// Compare-and-Swap (CAS) — only stores if current == expected
swapped := atomic.CompareAndSwapInt64(&val, 100, 200)
// swapped == true if val was 100 and is now 200
```

### `atomic.Value` — for any type

```go
var cache atomic.Value // stores any comparable type

// Writer goroutine
cache.Store(map[string]string{"key": "value"})

// Reader goroutines — always get a consistent snapshot
if v := cache.Load(); v != nil {
    m := v.(map[string]string)
    fmt.Println(m["key"])
}

// Swap: store new, get old atomically
old := cache.Swap(map[string]string{"key": "new"})
```

### When to use atomic vs mutex

| | `sync/atomic` | `sync.Mutex` |
|--|--|--|
| Best for | Simple counters, flags, pointers | Protecting struct/multi-field state |
| Overhead | Lowest | Low |
| Composability | Single variable only | Multiple fields together |
| Readability | Lower | Higher |

---

## 3. `errgroup` — Goroutines with Error Propagation (14:00)

`errgroup.Group` is `sync.WaitGroup` + error collection. Use it when you launch multiple goroutines and need to know if any failed.

```bash
go get golang.org/x/sync
```

### Basic errgroup

```go
package main

import (
    "fmt"
    "golang.org/x/sync/errgroup"
)

func fetchUser(id int) error {
    if id == 3 {
        return fmt.Errorf("user %d not found", id)
    }
    fmt.Printf("fetched user %d\n", id)
    return nil
}

func main() {
    var g errgroup.Group

    for _, id := range []int{1, 2, 3, 4, 5} {
        id := id // capture loop variable
        g.Go(func() error {
            return fetchUser(id)
        })
    }

    if err := g.Wait(); err != nil {
        fmt.Println("Error:", err) // first error returned
    } else {
        fmt.Println("All succeeded")
    }
}
```

### errgroup with shared result slice

```go
func fetchCities(cities []string) ([]*WeatherInfo, error) {
    var g errgroup.Group
    var mu sync.Mutex
    results := make([]*WeatherInfo, len(cities))

    for i, city := range cities {
        i, city := i, city
        g.Go(func() error {
            info, err := fetchWeather(city)
            if err != nil {
                return fmt.Errorf("city %s: %w", city, err)
            }
            mu.Lock()
            results[i] = info
            mu.Unlock()
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

**Key difference from WaitGroup:**

| | `sync.WaitGroup` | `errgroup.Group` |
|--|--|--|
| Error handling | Manual — need separate channel | Built-in — `Wait()` returns error |
| Context cancel | Manual | Via `errgroup.WithContext` |
| Pattern | `Add/Done/Wait` | `Go/Wait` |

---

## 4. `errgroup.WithContext` — Cancel on First Error (20:00)

When one goroutine fails, cancel all others automatically.

```go
package main

import (
    "context"
    "fmt"
    "golang.org/x/sync/errgroup"
)

func processItem(ctx context.Context, id int) error {
    select {
    case <-ctx.Done():
        return ctx.Err() // cancelled by another goroutine's failure
    default:
    }
    if id == 3 {
        return fmt.Errorf("item %d failed", id)
    }
    fmt.Printf("processed item %d\n", id)
    return nil
}

func main() {
    g, ctx := errgroup.WithContext(context.Background())

    for i := 1; i <= 5; i++ {
        i := i
        g.Go(func() error {
            return processItem(ctx, i)
        })
    }

    if err := g.Wait(); err != nil {
        fmt.Println("Pipeline failed:", err)
    }
}
```

### errgroup + pipeline stages

```go
func run(ctx context.Context) error {
    g, ctx := errgroup.WithContext(ctx)

    jobs := make(chan Job)

    // Stage 1: produce
    g.Go(func() error {
        defer close(jobs)
        return produce(ctx, jobs)
    })

    // Stage 2: consume (multiple workers)
    for w := 0; w < 5; w++ {
        g.Go(func() error {
            return consume(ctx, jobs)
        })
    }

    return g.Wait() // waits for ALL stages; returns first error
}
```

---

## 5. `singleflight` — Deduplicating Concurrent Calls (25:00)

When multiple goroutines request the same expensive resource simultaneously, `singleflight` makes only ONE real call and shares the result with all callers.

```bash
go get golang.org/x/sync/singleflight
```

**Problem it solves:** Cache stampede / thundering herd.

```go
package main

import (
    "fmt"
    "golang.org/x/sync/singleflight"
    "time"
)

var group singleflight.Group

type WeatherInfo struct {
    City string
    Temp int
}

func fetchWeatherFromDB(city string) (*WeatherInfo, error) {
    time.Sleep(100 * time.Millisecond) // simulate slow DB call
    fmt.Printf("  DB HIT for %s\n", city) // only printed once per burst
    return &WeatherInfo{City: city, Temp: 25}, nil
}

func GetWeather(city string) (*WeatherInfo, error) {
    result, err, shared := group.Do(city, func() (interface{}, error) {
        return fetchWeatherFromDB(city)
    })
    if shared {
        fmt.Println("  result was shared!")
    }
    if err != nil {
        return nil, err
    }
    return result.(*WeatherInfo), nil
}

func main() {
    var wg sync.WaitGroup

    // 10 goroutines all requesting "London" at the same time
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            info, _ := GetWeather("London")
            fmt.Printf("Got: %+v\n", info)
        }()
    }
    wg.Wait()
    // "DB HIT for London" printed only once
}
```

### `group.Do` vs `group.DoChan`

```go
// Do — blocks the calling goroutine until result is ready
result, err, shared := group.Do(key, fn)

// DoChan — returns a channel immediately (non-blocking)
ch := group.DoChan(key, fn)
select {
case res := <-ch:
    // res.Val, res.Err, res.Shared
case <-ctx.Done():
    // cancelled
}

// Forget — drop in-flight result, next call starts fresh
group.Forget(key)
```

---

## 6. Bounded Concurrency — Channel Semaphore (30:00)

Limit how many goroutines run concurrently using a buffered channel as a semaphore.

```go
package main

import (
    "fmt"
    "sync"
)

func processWithLimit(items []string, maxConcurrent int) {
    sem := make(chan struct{}, maxConcurrent) // the semaphore
    var wg sync.WaitGroup

    for _, item := range items {
        item := item
        sem <- struct{}{} // acquire a slot (blocks when full)
        wg.Add(1)

        go func() {
            defer wg.Done()
            defer func() { <-sem }() // release slot when done
            process(item)
        }()
    }

    wg.Wait()
}

// Or combined with errgroup
func processWithErrgroup(ctx context.Context, items []string) error {
    g, ctx := errgroup.WithContext(ctx)
    sem := make(chan struct{}, 10)

    for _, item := range items {
        item := item
        sem <- struct{}{}
        g.Go(func() error {
            defer func() { <-sem }()
            return process(ctx, item)
        })
    }
    return g.Wait()
}
```

**Why this works:**
- Buffered channel acts as a counting semaphore
- `sem <- struct{}{}` blocks when `maxConcurrent` goroutines are running
- `<-sem` frees a slot when a goroutine finishes
- `struct{}{}` uses zero bytes of memory

---

## 7. Weighted Semaphore — Variable-Cost Tasks (34:00)

When tasks consume different amounts of resources (CPU cores, memory, file descriptors), use a weighted semaphore.

```go
package main

import (
    "context"
    "fmt"
    "golang.org/x/sync/semaphore"
    "golang.org/x/sync/errgroup"
    "sync"
)

func processCities(ctx context.Context, cities []string) ([]*WeatherInfo, error) {
    const maxWeight int64 = 100
    sem := semaphore.NewWeighted(maxWeight)

    var g errgroup.Group
    var mu sync.Mutex
    results := make([]*WeatherInfo, len(cities))

    for i, city := range cities {
        i, city := i, city
        cost := int64(len(city)) // longer city name = more "expensive"

        if err := sem.Acquire(ctx, cost); err != nil {
            break // context cancelled
        }

        g.Go(func() error {
            defer sem.Release(cost)
            info, err := fetchWeather(city)
            mu.Lock()
            results[i] = info
            mu.Unlock()
            return err
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

**When to use weighted vs simple:**

| | Channel Semaphore | Weighted Semaphore |
|--|--|--|
| Task cost | Uniform (each task = 1 slot) | Variable (tasks have different weights) |
| API | Built-in channels | `golang.org/x/sync/semaphore` |
| Context support | Manual | Built-in (`Acquire` accepts `ctx`) |
| Use case | Simple rate limiting | Memory/CPU-aware concurrency |

---

## 8. Rate Limiting — Token Bucket (37:00)

Control how fast operations execute to protect downstream services.

### Simple rate limiter with `time.Ticker`

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    requests := make(chan int, 10)
    for i := 1; i <= 10; i++ {
        requests <- i
    }
    close(requests)

    // Allow 2 requests per second
    limiter := time.NewTicker(500 * time.Millisecond)
    defer limiter.Stop()

    for req := range requests {
        <-limiter.C // block until next tick
        fmt.Println("Request", req, "processed at", time.Now().Format("15:04:05.000"))
    }
}
```

### Token bucket (burst-capable rate limiter)

```go
package main

import (
    "math"
    "sync"
    "time"
)

type RateLimiter struct {
    rate       float64 // tokens added per second
    bucketSize float64 // max tokens in bucket
    tokens     float64 // current tokens
    lastRefill time.Time
    mu         sync.Mutex
}

func NewRateLimiter(rate, burst float64) *RateLimiter {
    return &RateLimiter{
        rate:       rate,
        bucketSize: burst,
        tokens:     burst,
        lastRefill: time.Now(),
    }
}

func (l *RateLimiter) Allow() bool {
    l.mu.Lock()
    defer l.mu.Unlock()

    now := time.Now()
    elapsed := now.Sub(l.lastRefill).Seconds()
    l.tokens = math.Min(l.bucketSize, l.tokens+elapsed*l.rate)
    l.lastRefill = now

    if l.tokens >= 1 {
        l.tokens--
        return true
    }
    return false
}

// Usage
func main() {
    limiter := NewRateLimiter(5, 10) // 5 req/sec, burst of 10

    for i := 0; i < 20; i++ {
        if limiter.Allow() {
            fmt.Printf("Request %d: allowed\n", i)
        } else {
            fmt.Printf("Request %d: rate limited\n", i)
        }
    }
}
```

### Using `golang.org/x/time/rate` (production-ready)

```go
import "golang.org/x/time/rate"

limiter := rate.NewLimiter(rate.Limit(10), 20) // 10/sec, burst 20

// Block until allowed
if err := limiter.Wait(ctx); err != nil {
    return err
}

// Non-blocking check
if !limiter.Allow() {
    return errors.New("rate limited")
}

// Reserve tokens in advance
r := limiter.Reserve()
time.Sleep(r.Delay())
```

---

## 9. Pub-Sub Pattern (39:00)

Publishers broadcast events to multiple subscribers over channels. Useful for event-driven architectures.

```go
package main

import (
    "fmt"
    "sync"
)

type PubSub struct {
    mu          sync.RWMutex
    subscribers map[string][]chan interface{}
}

func NewPubSub() *PubSub {
    return &PubSub{
        subscribers: make(map[string][]chan interface{}),
    }
}

func (ps *PubSub) Subscribe(topic string, bufSize int) <-chan interface{} {
    ps.mu.Lock()
    defer ps.mu.Unlock()

    ch := make(chan interface{}, bufSize)
    ps.subscribers[topic] = append(ps.subscribers[topic], ch)
    return ch
}

func (ps *PubSub) Publish(topic string, msg interface{}) {
    ps.mu.RLock()
    defer ps.mu.RUnlock()

    for _, ch := range ps.subscribers[topic] {
        select {
        case ch <- msg:
        default:
            // subscriber too slow — drop or block depending on policy
        }
    }
}

func (ps *PubSub) Close(topic string) {
    ps.mu.Lock()
    defer ps.mu.Unlock()

    for _, ch := range ps.subscribers[topic] {
        close(ch)
    }
    delete(ps.subscribers, topic)
}

func main() {
    ps := NewPubSub()

    sub1 := ps.Subscribe("orders", 10)
    sub2 := ps.Subscribe("orders", 10)

    var wg sync.WaitGroup
    wg.Add(2)

    go func() {
        defer wg.Done()
        for msg := range sub1 {
            fmt.Println("Subscriber 1:", msg)
        }
    }()

    go func() {
        defer wg.Done()
        for msg := range sub2 {
            fmt.Println("Subscriber 2:", msg)
        }
    }()

    ps.Publish("orders", "order-001")
    ps.Publish("orders", "order-002")
    ps.Publish("orders", "order-003")

    ps.Close("orders")
    wg.Wait()
}
```

---

## Pattern Comparison — Which to Use?

| Scenario | Pattern |
|----------|---------|
| Init a singleton (DB conn, logger) | `sync.Once` / `sync.OnceValue` |
| Simple shared counter / flag | `sync/atomic` |
| Multiple goroutines, need any error | `errgroup.Group` |
| Multiple goroutines, cancel all on first error | `errgroup.WithContext` |
| Cache stampede / thundering herd | `singleflight` |
| Limit concurrency uniformly | Channel semaphore |
| Limit concurrency by resource cost | Weighted semaphore |
| Control request rate | Token bucket / `x/time/rate` |
| Broadcast events to multiple consumers | Pub-Sub |
| Process data in stages concurrently | Pipeline |

---

## Real-World Combined Example

A production-grade fetch service combining errgroup + singleflight + weighted semaphore:

```go
package main

import (
    "context"
    "fmt"
    "sync"
    "golang.org/x/sync/errgroup"
    "golang.org/x/sync/semaphore"
    "golang.org/x/sync/singleflight"
)

type Service struct {
    group singleflight.Group
    sem   *semaphore.Weighted
    cache sync.Map
}

func NewService() *Service {
    return &Service{
        sem: semaphore.NewWeighted(50),
    }
}

func (s *Service) FetchMany(ctx context.Context, keys []string) ([]string, error) {
    g, ctx := errgroup.WithContext(ctx)
    var mu sync.Mutex
    results := make([]string, len(keys))

    for i, key := range keys {
        i, key := i, key

        // Acquire semaphore slot (weighted by key length)
        cost := int64(len(key))
        if err := s.sem.Acquire(ctx, cost); err != nil {
            return nil, err
        }

        g.Go(func() error {
            defer s.sem.Release(cost)

            // Deduplicate concurrent identical requests
            val, err, _ := s.group.Do(key, func() (interface{}, error) {
                // Check cache first
                if cached, ok := s.cache.Load(key); ok {
                    return cached, nil
                }
                result := expensiveFetch(key)
                s.cache.Store(key, result)
                return result, nil
            })
            if err != nil {
                return fmt.Errorf("fetch %s: %w", key, err)
            }

            mu.Lock()
            results[i] = val.(string)
            mu.Unlock()
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

---

## Common Mistakes in Part 2 Patterns

| Mistake | Problem | Fix |
|---------|---------|-----|
| Reusing a spent `sync.Once` | Second `Do()` call does nothing | Create a new `sync.Once` |
| Reading partial state from `atomic.Value` | `Store` and `Load` on different types | Always store/load the same concrete type |
| Ignoring `shared` return from `singleflight.Do` | Miss knowing if result was shared | Check `shared` bool for observability |
| Not releasing weighted semaphore on error | Semaphore leaks, future acquires block | Always `defer sem.Release(cost)` |
| Closing PubSub channel while publishing | Panic: send on closed channel | Lock before checking closed state |
| Loop variable capture in `g.Go` closures | All goroutines use last value | Always shadow: `item := item` inside loop |

---

## Quick Reference — New Packages

```go
// sync.Once
var once sync.Once
once.Do(func() { /* runs once */ })
var fn = sync.OnceFunc(func() { /* runs once */ })         // Go 1.21+
var getVal = sync.OnceValue(func() T { return val })       // Go 1.21+
var getBoth = sync.OnceValues(func() (T, error) { ... })   // Go 1.21+

// sync/atomic
atomic.AddInt64(&n, 1)
atomic.LoadInt64(&n)
atomic.StoreInt64(&n, v)
atomic.SwapInt64(&n, v)
atomic.CompareAndSwapInt64(&n, old, new)
var val atomic.Value; val.Store(v); val.Load()

// errgroup (golang.org/x/sync/errgroup)
var g errgroup.Group
g.Go(func() error { return nil })
err := g.Wait()

g, ctx := errgroup.WithContext(ctx)    // cancel-on-error

// singleflight (golang.org/x/sync/singleflight)
var sf singleflight.Group
result, err, shared := sf.Do(key, fn)
ch := sf.DoChan(key, fn)
sf.Forget(key)

// semaphore (golang.org/x/sync/semaphore)
sem := semaphore.NewWeighted(maxWeight)
sem.Acquire(ctx, cost)
sem.Release(cost)

// x/time/rate
lim := rate.NewLimiter(rate.Limit(10), burst)
lim.Wait(ctx)
lim.Allow()
```

---

## Further Resources

- [Advanced Go Concurrency — Encore Blog](https://encore.dev/blog/advanced-go-concurrency)
- [sync package docs — pkg.go.dev](https://pkg.go.dev/sync)
- [golang.org/x/sync docs](https://pkg.go.dev/golang.org/x/sync)
- [Go sync.Once deep dive — VictoriaMetrics](https://victoriametrics.com/blog/go-sync-once/)
- [Advanced Go Concurrency Patterns — Rob Pike (2013)](https://go.dev/talks/2013/advconc.slide)
- [Part 1 Notes](./go-concurrency-patterns-notes.md)
