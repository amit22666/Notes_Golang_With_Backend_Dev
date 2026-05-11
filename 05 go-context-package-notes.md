# golang context package explained — Study Notes

**Source:** [golang context package explained: the package that changed concurrency forever](https://www.youtube.com/watch?v=8omcakb31xQ&list=PL7g1jYj15RUNqJStuwE9SCmeOKpgxC0HP&index=5)

> **Prerequisites:** Parts 1–4 — goroutines, channels, select, done-channel pattern.
> The `context` package is the **official, standard-library answer** to the done-channel pattern from Part 4. It changed how all Go programs handle cancellation, timeouts, and request-scoped data.

---

## Timeline Overview

| Timestamp | Topic |
|-----------|-------|
| 0:00 | Why context exists — the problem it solves |
| 3:00 | The `Context` interface |
| 6:00 | `context.Background()` and `context.TODO()` |
| 9:00 | `context.WithCancel` |
| 15:00 | `context.WithTimeout` and `context.WithDeadline` |
| 22:00 | `context.WithValue` — passing request-scoped data |
| 28:00 | Context propagation — the cancellation tree |
| 33:00 | Real-world: HTTP server with context |
| 37:00 | Real-world: Database queries with context |
| 40:00 | New features: `WithCancelCause`, `AfterFunc`, `WithoutCancel` |
| 44:00 | Internals: `cancelCtx`, `valueCtx`, `timerCtx` |
| 47:00 | Common mistakes and anti-patterns |

---

## 1. Why Context Exists (0:00)

Before `context` (pre Go 1.7), stopping in-progress goroutines required hand-rolling a `done` channel and threading it through every function:

```go
// The old way — fragile, no standard contract
func oldWorker(done <-chan struct{}, in <-chan int) {
    for {
        select {
        case <-done:
            return
        case v := <-in:
            process(v)
        }
    }
}
```

Problems with the manual approach:
- No standard: every package invents its own cancellation mechanism
- No deadlines: you'd need a separate `time.After` channel
- No values: no way to pass request-scoped data (trace IDs, auth tokens)
- No propagation: you have to manually thread `done` through every function call

The `context` package (introduced in **Go 1.7**) standardises all three into one interface that every package in the ecosystem speaks.

```
context.Context solves:
  1. Cancellation   → WithCancel
  2. Deadlines      → WithDeadline / WithTimeout
  3. Request values → WithValue
```

---

## 2. The `Context` Interface (3:00)

```go
type Context interface {
    // When will this context be cancelled? ok=false if no deadline set.
    Deadline() (deadline time.Time, ok bool)

    // Closed when work should stop. Use in select statements.
    // Returns nil for contexts that can never be cancelled.
    Done() <-chan struct{}

    // nil if Done not yet closed.
    // context.Canceled     — if cancelled via CancelFunc
    // context.DeadlineExceeded — if deadline passed
    Err() error

    // Returns the value associated with key, or nil.
    Value(key any) any
}
```

**Two sentinel errors:**
```go
context.Canceled        // = errors.New("context canceled")
context.DeadlineExceeded // implements net.Error with Timeout() == true
```

---

## 3. Root Contexts — `Background` and `TODO` (6:00)

Every context tree starts from one of these two roots.

```go
// Use Background as the root in main, init, and at the top of request handlers
ctx := context.Background()

// Use TODO as a placeholder when you haven't decided which context to pass yet
ctx := context.TODO()
```

Both are **empty contexts**: never cancelled, no deadline, no values.
The difference is semantic — it communicates intent to the reader.

---

## 4. `context.WithCancel` — Manual Cancellation (9:00)

Creates a child context and a `CancelFunc`. Call `cancel()` to stop all work using this context.

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func worker(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("worker %d stopped: %v\n", id, ctx.Err())
            return
        default:
            fmt.Printf("worker %d working...\n", id)
            time.Sleep(300 * time.Millisecond)
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel() // always defer — releases resources even if you cancel early

    for i := 1; i <= 3; i++ {
        go worker(ctx, i)
    }

    time.Sleep(1 * time.Second)
    cancel() // signals all 3 workers simultaneously
    time.Sleep(100 * time.Millisecond) // let workers print their messages
}
```

### Cancelling on first error

```go
func runAll(ctx context.Context, tasks []Task) error {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    errc := make(chan error, len(tasks))
    for _, t := range tasks {
        t := t
        go func() {
            errc <- t.Run(ctx)
        }()
    }

    for range tasks {
        if err := <-errc; err != nil {
            cancel() // cancel remaining tasks on first failure
            return err
        }
    }
    return nil
}
```

**Key rules:**
- `cancel()` is safe to call multiple times — only the first call has effect
- `cancel()` is safe to call from multiple goroutines simultaneously
- **Always `defer cancel()`** — even if you call `cancel()` explicitly, the defer is a safety net
- Not calling `cancel()` leaks child contexts and their goroutines until the parent cancels

---

## 5. `context.WithTimeout` and `context.WithDeadline` (15:00)

Automatically cancel after a duration or at an absolute time.

### `WithTimeout` — relative duration

```go
func fetchData(url string) ([]byte, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel() // cancel early if fetch completes before 5s

    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return nil, err
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        // If timeout: err wraps context.DeadlineExceeded
        return nil, fmt.Errorf("fetch %s: %w", url, err)
    }
    defer resp.Body.Close()
    return io.ReadAll(resp.Body)
}
```

### `WithDeadline` — absolute time

```go
func processBeforeMidnight(ctx context.Context, data []byte) error {
    // Must finish by end of business today
    eod := time.Date(time.Now().Year(), time.Now().Month(), time.Now().Day(),
        23, 59, 59, 0, time.Local)

    ctx, cancel := context.WithDeadline(ctx, eod)
    defer cancel()

    return processData(ctx, data)
}
```

### `WithTimeout` is just sugar over `WithDeadline`

```go
// These two are equivalent
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
ctx, cancel = context.WithDeadline(parent, time.Now().Add(5*time.Second))
```

### Checking if a deadline is set

```go
if deadline, ok := ctx.Deadline(); ok {
    remaining := time.Until(deadline)
    fmt.Printf("%.1fs remaining\n", remaining.Seconds())
}
```

### Which error did we get?

```go
select {
case <-ctx.Done():
    err := ctx.Err()
    if errors.Is(err, context.DeadlineExceeded) {
        fmt.Println("timed out")
    } else if errors.Is(err, context.Canceled) {
        fmt.Println("cancelled by caller")
    }
}
```

---

## 6. `context.WithValue` — Request-Scoped Data (22:00)

Attach values that flow with the request through the call stack — trace IDs, auth tokens, user info.

### WRONG — using a plain string key

```go
// Collision risk: any package can overwrite "userID"
ctx = context.WithValue(ctx, "userID", 42)
```

### RIGHT — use an unexported custom type as the key

```go
package middleware

// Unexported type — only this package can create keys of this type
type contextKey string

const (
    keyUserID   contextKey = "userID"
    keyTraceID  contextKey = "traceID"
    keyTenantID contextKey = "tenantID"
)

// Typed setter/getter — the only way to put/get these values
func WithUserID(ctx context.Context, userID int) context.Context {
    return context.WithValue(ctx, keyUserID, userID)
}

func UserID(ctx context.Context) (int, bool) {
    v, ok := ctx.Value(keyUserID).(int)
    return v, ok
}
```

```go
// HTTP middleware sets the value once
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        userID := parseToken(r.Header.Get("Authorization"))
        ctx := middleware.WithUserID(r.Context(), userID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Handler reads it without knowing how it was stored
func MyHandler(w http.ResponseWriter, r *http.Request) {
    userID, ok := middleware.UserID(r.Context())
    if !ok {
        http.Error(w, "unauthorized", http.StatusUnauthorized)
        return
    }
    fmt.Fprintf(w, "Hello user %d", userID)
}
```

**WithValue rules:**
- Key must be **comparable**
- Use **unexported custom types** as keys to prevent collisions
- Values must be **safe for concurrent use** (treat them as read-only)
- Only for **request-scoped data** — don't pass function parameters via context
- `Value()` searches up the chain linearly — don't nest hundreds of values

---

## 7. Context Propagation — The Cancellation Tree (28:00)

Every derived context forms a **tree**. Cancelling a parent cancels **all descendants**.

```
context.Background()           ← root (never cancels)
    │
    ├── WithCancel             ← request context
    │       │
    │       ├── WithTimeout    ← DB query (5s)
    │       │
    │       └── WithTimeout    ← HTTP call (2s)
    │               │
    │               └── WithValue  ← traceID added
    │
    └── WithCancel             ← another request
```

```go
func handleRequest(w http.ResponseWriter, r *http.Request) {
    // r.Context() is already cancelled if client disconnects
    ctx := r.Context()

    // DB query gets its own tighter timeout
    dbCtx, dbCancel := context.WithTimeout(ctx, 2*time.Second)
    defer dbCancel()

    // HTTP call gets its own timeout
    httpCtx, httpCancel := context.WithTimeout(ctx, 500*time.Millisecond)
    defer httpCancel()

    var (
        dbResult  Result
        httpResult Result
        wg        sync.WaitGroup
        mu        sync.Mutex
    )

    wg.Add(2)
    go func() {
        defer wg.Done()
        res, err := queryDB(dbCtx)
        if err == nil {
            mu.Lock()
            dbResult = res
            mu.Unlock()
        }
    }()

    go func() {
        defer wg.Done()
        res, err := callAPI(httpCtx)
        if err == nil {
            mu.Lock()
            httpResult = res
            mu.Unlock()
        }
    }()

    wg.Wait()
    // If the HTTP request was cancelled (client left), both sub-contexts
    // are ALSO cancelled — no wasted DB/API work
}
```

**Propagation rules:**
1. Child is cancelled when **parent** is cancelled
2. Child is cancelled when **its own** deadline/cancel fires
3. Parent is **NOT** affected when a child is cancelled
4. `cancel()` removes the child from the parent's children map — releases memory

---

## 8. Real-World: HTTP Server with Context (33:00)

`net/http` automatically provides a context per request that cancels if the client disconnects.

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "time"
)

func slowHandler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context() // already cancelled if client closes connection

    resultCh := make(chan string, 1)
    go func() {
        // Simulate slow work
        time.Sleep(3 * time.Second)
        resultCh <- "done"
    }()

    select {
    case result := <-resultCh:
        fmt.Fprintln(w, result)
    case <-ctx.Done():
        // Client disconnected — stop processing, don't write response
        fmt.Println("client left:", ctx.Err())
        // Note: do NOT write to w here, the connection is gone
    }
}

func main() {
    http.HandleFunc("/slow", slowHandler)
    http.ListenAndServe(":8080", nil)
}
```

### Propagating context through the entire request chain

```go
// Every function in the call chain accepts ctx as the first parameter
func handleOrder(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    user, err := getUser(ctx, r.Header.Get("Authorization"))
    if err != nil { http.Error(w, err.Error(), 401); return }

    inventory, err := checkInventory(ctx, r.URL.Query().Get("item"))
    if err != nil { http.Error(w, err.Error(), 500); return }

    order, err := createOrder(ctx, user, inventory)
    if err != nil { http.Error(w, err.Error(), 500); return }

    json.NewEncoder(w).Encode(order)
}

func getUser(ctx context.Context, token string) (*User, error) {
    // passes ctx to DB/cache — will cancel if request is cancelled
    return db.QueryRowContext(ctx, "SELECT * FROM users WHERE token=$1", token).Scan(...)
}
```

---

## 9. Real-World: Database Queries with Context (37:00)

All `database/sql` methods have a `Context` variant — always use them.

```go
import "database/sql"

// QueryContext — stops the query if ctx is cancelled
func getAlbums(ctx context.Context, db *sql.DB) ([]Album, error) {
    // If HTTP request times out or client leaves, this query is cancelled
    rows, err := db.QueryContext(ctx, "SELECT id, title, artist FROM albums")
    if err != nil {
        return nil, fmt.Errorf("getAlbums: %w", err)
    }
    defer rows.Close()

    var albums []Album
    for rows.Next() {
        var a Album
        if err := rows.Scan(&a.ID, &a.Title, &a.Artist); err != nil {
            return nil, err
        }
        albums = append(albums, a)
    }
    return albums, rows.Err()
}

// ExecContext — stops the write if ctx is cancelled
func updateAlbum(ctx context.Context, db *sql.DB, id int, title string) error {
    _, err := db.ExecContext(ctx,
        "UPDATE albums SET title=$1 WHERE id=$2", title, id)
    return err
}

// Query with its own tighter timeout, nested inside request context
func queryWithTimeout(ctx context.Context, db *sql.DB) ([]Album, error) {
    // This cancels after 5s OR when the parent HTTP request cancels,
    // whichever comes FIRST
    queryCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    return getAlbums(queryCtx, db)
}
```

---

## 10. New Context Features (Go 1.20–1.21) (40:00)

### `WithCancelCause` + `Cause()` — Go 1.20

Distinguish *why* a context was cancelled.

```go
var ErrUserLoggedOut = errors.New("user logged out")

ctx, cancel := context.WithCancelCause(context.Background())

// In another goroutine:
go func() {
    if sessionExpired() {
        cancel(ErrUserLoggedOut) // attach a specific reason
    }
}()

<-ctx.Done()
fmt.Println(ctx.Err())         // context canceled
fmt.Println(context.Cause(ctx)) // user logged out  ← the actual reason
```

### `WithoutCancel` — Go 1.21

Derive a context that carries values but **breaks the cancellation chain**.

```go
func processRequest(ctx context.Context, data []byte) {
    // Even if ctx is cancelled, we still want to record metrics
    metricsCtx := context.WithoutCancel(ctx)
    defer recordMetrics(metricsCtx, time.Now())  // always runs

    // This respects cancellation
    result, err := doWork(ctx, data)
    if err != nil {
        return
    }
    sendResult(ctx, result)
}

// Use cases for WithoutCancel:
// - Audit logging that must complete
// - Transaction rollbacks
// - Cleanup operations
// - Caching results after the request ended
```

### `AfterFunc` — Go 1.21

Register a callback to run in a new goroutine when a context is cancelled.

```go
// Automatically rollback a DB transaction when context is cancelled
func withAutoRollback(ctx context.Context, tx *sql.Tx) (stop func() bool) {
    return context.AfterFunc(ctx, func() {
        // Runs in new goroutine when ctx is cancelled
        tx.Rollback()
    })
}

func doTransaction(ctx context.Context, db *sql.DB) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }

    stop := withAutoRollback(ctx, tx)

    if err := doWork(ctx, tx); err != nil {
        return err // rollback already triggered by AfterFunc
    }

    if stop() { // stop AfterFunc if we're committing
        return tx.Commit()
    }
    return ctx.Err() // context was cancelled before we could commit
}
```

### `WithDeadlineCause` / `WithTimeoutCause` — Go 1.21

Set a specific cause when the deadline/timeout fires.

```go
var ErrSlowService = errors.New("downstream service too slow")

ctx, cancel := context.WithTimeoutCause(
    context.Background(),
    2*time.Second,
    ErrSlowService, // this becomes Cause() when timeout fires
)
defer cancel()

<-ctx.Done()
fmt.Println(context.Cause(ctx)) // downstream service too slow
```

---

## 11. Internals — How Context Works (44:00)

### The context struct hierarchy

```
interface Context
    ├── emptyCtx       ← Background() and TODO()
    ├── cancelCtx      ← WithCancel()
    │     └── timerCtx ← WithDeadline() / WithTimeout()
    └── valueCtx       ← WithValue()
```

### `cancelCtx` — the core structure

```go
type cancelCtx struct {
    Context                       // parent

    mu       sync.Mutex
    done     atomic.Value         // stores the Done channel (lazy init)
    children map[canceler]struct{} // all children watching this ctx
    err      error                // set on first cancel
    cause    error                // set by WithCancelCause
}
```

When `cancel()` is called:
1. Sets `err` and `cause`
2. Closes the `done` channel (all `<-ctx.Done()` receivers wake up)
3. Iterates `children` map, calls `cancel()` on each child
4. Removes itself from parent's `children` map

### `valueCtx` — a linked list

```go
type valueCtx struct {
    Context        // parent — forms a chain
    key, val any
}

// Value() walks UP the chain until it finds the key or hits a root
func (c *valueCtx) Value(key any) any {
    if c.key == key {
        return c.val
    }
    return value(c.Context, key) // recurse to parent
}
```

This is why deeply nesting `WithValue` is slow — `O(n)` lookup where n is nesting depth.

### `timerCtx` — extends `cancelCtx`

```go
type timerCtx struct {
    cancelCtx
    timer    *time.Timer  // fires cancel() at deadline
    deadline time.Time
}
```

`WithDeadline` computes remaining time, creates a `time.AfterFunc` timer that calls `cancel()` at the deadline. That's all the magic.

### Propagation

When you create a child with `WithCancel(parent)`, Go does:
1. If parent already cancelled → cancel child immediately
2. If parent is a `cancelCtx` → add child to `parent.children` map
3. Otherwise → spawn a goroutine watching `parent.Done()` (for custom Context implementations)

---

## 12. Common Mistakes and Anti-Patterns (47:00)

### 1. Storing Context in a struct

```go
// WRONG — context is request-scoped, not object-scoped
type Server struct {
    ctx context.Context // don't do this
}

// RIGHT — pass as parameter
func (s *Server) HandleRequest(ctx context.Context, req *Request) error {
    return s.doWork(ctx, req)
}
```

### 2. Passing nil context

```go
// WRONG — panics or causes subtle bugs
doWork(nil)

// RIGHT — use TODO() if you don't have a context yet
doWork(context.TODO())
```

### 3. Not deferring `cancel()`

```go
// WRONG — resource leak if function returns early
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
result, err := doWork(ctx)
cancel() // might not be reached on error paths

// RIGHT
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
defer cancel() // always runs, no matter how function returns
result, err := doWork(ctx)
```

### 4. Ignoring `cancel()` on `WithTimeout`

```go
// WRONG — timer and goroutine leak until timeout fires
ctx, _ = context.WithTimeout(parent, 5*time.Second)

// RIGHT — always capture and defer cancel
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
defer cancel()
```

### 5. Cancelling parent before goroutine finishes

```go
// WRONG — goroutine's context is cancelled when doSomething returns
func doSomething(parent context.Context) {
    ctx, cancel := context.WithCancel(parent)
    defer cancel() // cancels ctx — but goroutine is still using it!
    go func(ctx context.Context) {
        doLongWork(ctx) // may be cancelled prematurely
    }(ctx)
}

// RIGHT — if goroutine must outlive the function, use a fresh context
func doSomething(parent context.Context) {
    go func() {
        // Use WithoutCancel to inherit values but not cancellation,
        // or use context.Background() if fully independent
        bgCtx := context.WithoutCancel(parent)
        doLongWork(bgCtx)
    }()
}
```

### 6. Using string keys with `WithValue`

```go
// WRONG — any package can accidentally overwrite "userID"
ctx = context.WithValue(ctx, "userID", 42)

// RIGHT — unexported custom type, impossible to collide
type ctxKey string
const userIDKey ctxKey = "userID"
ctx = context.WithValue(ctx, userIDKey, 42)
```

### 7. Passing business logic through context values

```go
// WRONG — makes function behaviour implicit and untestable
func processOrder(ctx context.Context) {
    db := ctx.Value("db").(*sql.DB) // hidden dependency
}

// RIGHT — pass dependencies as explicit parameters
func processOrder(ctx context.Context, db *sql.DB) {
    // explicit, testable
}
```

---

## Complete Real-World Example

HTTP server with authentication middleware, request tracing, DB query, and timeout:

```go
package main

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
)

// --- Context keys ---
type ctxKey string

const (
    keyUserID  ctxKey = "userID"
    keyTraceID ctxKey = "traceID"
)

func withUserID(ctx context.Context, id int) context.Context {
    return context.WithValue(ctx, keyUserID, id)
}
func userID(ctx context.Context) (int, bool) {
    v, ok := ctx.Value(keyUserID).(int)
    return v, ok
}

// --- Middleware ---
func traceMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        traceID := r.Header.Get("X-Trace-ID")
        ctx := context.WithValue(r.Context(), keyTraceID, traceID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        id := parseToken(r.Header.Get("Authorization"))
        if id == 0 {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }
        ctx := withUserID(r.Context(), id)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// --- Handler ---
func profileHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context() // cancelled if client disconnects

        id, _ := userID(ctx)

        // DB query with its own tight timeout, nested inside request ctx
        queryCtx, cancel := context.WithTimeout(ctx, 2*time.Second)
        defer cancel()

        var name string
        err := db.QueryRowContext(queryCtx,
            "SELECT name FROM users WHERE id=$1", id,
        ).Scan(&name)

        if err != nil {
            if ctx.Err() != nil {
                // Request was cancelled (client left)
                return
            }
            http.Error(w, "db error", 500)
            return
        }

        json.NewEncoder(w).Encode(map[string]string{"name": name})
    }
}

func main() {
    db, _ := sql.Open("postgres", "...")

    mux := http.NewServeMux()
    mux.Handle("/profile", traceMiddleware(authMiddleware(profileHandler(db))))

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
    }
    srv.ListenAndServe()
}
```

---

## Quick Reference — Full API

```go
// Root contexts
ctx := context.Background()   // use in main, tests, top of request chain
ctx := context.TODO()         // placeholder when ctx not yet determined

// Derived contexts — all return (Context, CancelFunc)
ctx, cancel := context.WithCancel(parent)                   // manual cancel
ctx, cancel := context.WithTimeout(parent, 5*time.Second)   // relative deadline
ctx, cancel := context.WithDeadline(parent, time.Time{})    // absolute deadline
ctx          = context.WithValue(parent, key, val)           // attach value

// Go 1.20+
ctx, cancel := context.WithCancelCause(parent)              // cancel with reason
cancel(err)                                                  // CancelCauseFunc
cause        := context.Cause(ctx)                           // retrieve reason

// Go 1.21+
ctx          = context.WithoutCancel(parent)                 // break cancel chain
stop         := context.AfterFunc(ctx, fn)                  // run fn when cancelled
ctx, cancel  := context.WithTimeoutCause(parent, d, err)    // timeout + cause
ctx, cancel  := context.WithDeadlineCause(parent, t, err)   // deadline + cause

// Context methods
deadline, ok := ctx.Deadline()
<-ctx.Done()                  // blocks until cancelled
err          := ctx.Err()     // context.Canceled or context.DeadlineExceeded
val          := ctx.Value(key)

// Convention
func myFunc(ctx context.Context, arg1 T, ...) error { ... }
// ctx is ALWAYS the first parameter, named ctx
```

---

## The 6 Rules of Context

| # | Rule |
|---|------|
| 1 | **First parameter**, always named `ctx` |
| 2 | **Never store** a Context in a struct |
| 3 | **Never pass nil** — use `context.TODO()` |
| 4 | **Always `defer cancel()`** immediately after `WithCancel`/`WithTimeout`/`WithDeadline` |
| 5 | **Values only for request-scoped data** — not optional function parameters |
| 6 | **Custom unexported key types** — never plain strings for `WithValue` |

---

## Further Resources

- [context package — pkg.go.dev](https://pkg.go.dev/context)
- [Go Pipelines & Cancellation — go.dev](https://go.dev/blog/pipelines)
- [Canceling Database Operations — go.dev](https://go.dev/doc/database/cancel-operations)
- [How To Use Contexts in Go — DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-use-contexts-in-go)
- [Cascading Context Cancellation Internals — dev.to](https://dev.to/flew1x/cascading-context-cancellation-in-go-from-source-code-to-production-patterns-177j)
- [Context in Golang — sohamkamani.com](https://www.sohamkamani.com/golang/context/)
- [Part 4 Notes — done channel pattern](./go-concurrency-mind-blowing-pattern-notes.md)
