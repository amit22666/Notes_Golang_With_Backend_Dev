# Golang Roadmap That Gets You Hired

**Source:** https://www.youtube.com/watch?v=ZzC3PNeCk6w  
**Type:** Career / Learning Roadmap

---

## Why Go in 2025?

- Compiled, statically typed, garbage-collected — **fast and safe**
- First-class concurrency (goroutines, channels) — built for the cloud era
- Simple syntax, fast compile times, single binary output
- #11 Tiobe Index 2024, consistently top-3 "most loved" in StackOverflow surveys
- Powers: **Docker, Kubernetes, Terraform, Prometheus, CockroachDB, Caddy**
- Adopted by: Google, Uber, Dropbox, Netflix, PayPal, Twitch, Cloudflare
- Salary: **$117k–$170k/yr** (US, 2024–2025 range)

---

## The Full Roadmap at a Glance

```
Phase 1  → Language Fundamentals
Phase 2  → Go Internals (types, interfaces, memory)
Phase 3  → Concurrency ← most important differentiator
Phase 4  → Standard Library
Phase 5  → Web & REST APIs
Phase 6  → Databases & Caching
Phase 7  → Testing & Debugging
Phase 8  → Design Patterns
Phase 9  → Microservices & gRPC
Phase 10 → DevOps & Cloud
Phase 11 → CLI Development
        → Portfolio Projects
        → Interview Prep
```

---

## Phase 1 — Language Fundamentals

**Goal:** Read and write idiomatic Go code confidently.

### Core Syntax
```go
// Variables
var x int = 10
y := 20          // short declaration (most common)
const Pi = 3.14

// Multiple return values (idiomatic Go)
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// Named return values
func minMax(a, b int) (min, max int) {
    if a < b { return a, b }
    return b, a
}
```

### Data Types
| Category | Types |
|----------|-------|
| Basic | `bool`, `int`, `int8/16/32/64`, `uint*`, `float32/64`, `complex64/128`, `string`, `byte`, `rune` |
| Composite | `array`, `slice`, `map`, `struct` |
| Reference | `pointer`, `slice`, `map`, `channel`, `function`, `interface` |

### Slices, Maps, Arrays
```go
// Slice — dynamic array (use this, not arrays)
s := []int{1, 2, 3}
s = append(s, 4)
s2 := s[1:3]         // slice of slice — shares backing array

// Make — allocate slice/map/channel
nums := make([]int, 0, 10) // len=0, cap=10
m := make(map[string]int)

// Map
m["key"] = 42
val, ok := m["key"] // comma-ok idiom
```

### Key Keywords
```go
defer  // runs at function return (LIFO order)
panic  // runtime error — unwind stack
recover// catch panic inside deferred func
make   // allocate slice/map/chan
new    // allocate zero value, return pointer
range  // iterate slice/map/string/channel
go     // launch goroutine
select // multiplex channels
```

### Error Handling
```go
// Go has no exceptions — errors are values
result, err := doSomething()
if err != nil {
    return fmt.Errorf("doSomething failed: %w", err) // wrap with %w
}

// Sentinel errors
var ErrNotFound = errors.New("not found")

// Custom error type
type ValidationError struct {
    Field   string
    Message string
}
func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

// Unwrap
if errors.Is(err, ErrNotFound) { ... }
var ve *ValidationError
if errors.As(err, &ve) { ... }
```

### Toolchain
```bash
go run main.go         # compile + run
go build ./...         # compile
go test ./...          # run tests
go mod init <module>   # init module
go get <package>       # add dependency
go mod tidy            # clean up go.mod/go.sum
go vet ./...           # static analysis
gofmt -w .             # format code
golangci-lint run      # lint (install separately)
```

---

## Phase 2 — Go Internals

**Goal:** Understand Go's type system and memory model deeply.

### Structs, Methods, Embedding
```go
type User struct {
    ID    int
    Name  string
    Email string
}

// Value receiver — read-only, works on copy
func (u User) String() string {
    return fmt.Sprintf("%s <%s>", u.Name, u.Email)
}

// Pointer receiver — mutates, use for large structs
func (u *User) SetEmail(email string) {
    u.Email = email
}

// Embedding = composition
type Admin struct {
    User           // promotes User fields and methods
    Permissions []string
}
```

### Interfaces (Implicit)
```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
// Any type with Write() automatically implements Writer
// No "implements" keyword needed
```

### Pointers
```go
x := 42
p := &x    // address of x
*p = 99    // dereference — modifies x

// Escape analysis: compiler decides stack vs heap
func newUser() *User {
    u := User{Name: "Alice"} // escapes to heap (pointer returned)
    return &u
}
```

### Slices Internals
```go
// Slice header: {pointer, length, capacity}
// Shares backing array until append triggers reallocation
a := []int{1, 2, 3}
b := a[0:2]   // b shares a's array
b[0] = 99     // modifies a[0] too!

// Safe copy
c := make([]int, len(a))
copy(c, a)
```

---

## Phase 3 — Concurrency ★ Most Important

**This is what separates Go from other languages. Interviewers test this heavily.**

### Goroutines & Channels
```go
// Goroutine — lightweight thread (2KB initial stack)
go func() {
    fmt.Println("concurrent")
}()

// Unbuffered channel — synchronises sender and receiver
ch := make(chan int)
go func() { ch <- 42 }()
val := <-ch

// Buffered channel — sender doesn't block until full
bch := make(chan int, 5)
```

### Sync Primitives
```go
var wg sync.WaitGroup
var mu sync.Mutex

wg.Add(1)
go func() {
    defer wg.Done()
    mu.Lock()
    defer mu.Unlock()
    // critical section
}()
wg.Wait()
```

### Key Concurrency Patterns
```go
// Worker pool
jobs := make(chan int, 100)
for w := 0; w < runtime.GOMAXPROCS(0); w++ {
    go func() {
        for j := range jobs { process(j) }
    }()
}

// Fan-out / Fan-in
// Select — multiplex channels
select {
case msg := <-ch1:
case msg := <-ch2:
case <-time.After(1 * time.Second): // timeout
case <-ctx.Done():                  // cancellation
}

// Context — cancellation, deadlines, values
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
```

### Must-Know Concurrency Topics for Interviews
- Goroutine scheduler (M:N threading, GMP model)
- Race conditions → `go test -race`
- Deadlock detection
- `sync.Once`, `sync.Pool`, `sync.Map`
- `atomic` operations (`sync/atomic`)
- `errgroup` for goroutine error propagation
- Channel direction (`chan<- T` send-only, `<-chan T` receive-only)

---

## Phase 4 — Standard Library Must-Knows

| Package | Purpose |
|---------|---------|
| `fmt` | Formatted I/O |
| `errors` | Error creation and wrapping |
| `log` / `log/slog` | Logging (slog = structured, Go 1.21+) |
| `net/http` | HTTP client + server |
| `encoding/json` | JSON marshal/unmarshal |
| `context` | Cancellation and deadlines |
| `sync` | Mutex, WaitGroup, Once, Pool |
| `sync/atomic` | Lock-free operations |
| `time` | Time, Duration, Timer, Ticker |
| `os` | File I/O, env vars, process |
| `io` | Reader, Writer, Closer interfaces |
| `strings` / `bytes` | String/byte manipulation |
| `strconv` | String ↔ numeric conversion |
| `path/filepath` | File path handling |
| `testing` | Unit tests, benchmarks, fuzzing |
| `database/sql` | SQL database interface |
| `crypto/tls` | TLS configuration |
| `regexp` | Regular expressions |
| `sort` | Sorting algorithms |
| `math/rand` | Random numbers |

---

## Phase 5 — Web & REST APIs

**Start with `net/http` (no framework) — understand the primitives first.**

### net/http (Standard Library)
```go
// Handler interface
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

// Simple server
http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
})
log.Fatal(http.ListenAndServe(":8080", nil))
```

### Popular Frameworks (pick one)

| Framework | Style | When to Use |
|-----------|-------|-------------|
| **Gin** | Fast, Rails-like | Most popular; good for REST APIs |
| **Echo** | Minimalist, high-perf | Clean API, good middleware |
| **Chi** | Stdlib-compatible | Prefers net/http compatibility |
| **Fiber** | Express.js-inspired | Very high throughput |

```go
// Gin example
r := gin.Default()
r.Use(gin.Logger(), gin.Recovery())

r.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id")
    c.JSON(http.StatusOK, gin.H{"id": id})
})

r.POST("/users", func(c *gin.Context) {
    var user User
    if err := c.ShouldBindJSON(&user); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusCreated, user)
})

r.Run(":8080")
```

### Middleware Pattern
```go
func AuthMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        if token == "" {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

---

## Phase 6 — Databases & Caching

### SQL (database/sql + sqlx/pgx)
```go
// database/sql — standard interface
db, err := sql.Open("postgres", connStr)
db.SetMaxOpenConns(25)
db.SetMaxIdleConns(25)
db.SetConnMaxLifetime(5 * time.Minute)

// Query
rows, err := db.QueryContext(ctx, "SELECT id, name FROM users WHERE active=$1", true)
defer rows.Close()
for rows.Next() {
    var u User
    rows.Scan(&u.ID, &u.Name)
}

// Transaction
tx, err := db.BeginTx(ctx, nil)
defer tx.Rollback()
tx.ExecContext(ctx, "INSERT INTO ...")
tx.Commit()
```

### ORM — GORM
```go
import "gorm.io/gorm"

type User struct {
    gorm.Model        // adds ID, CreatedAt, UpdatedAt, DeletedAt
    Name  string
    Email string `gorm:"uniqueIndex"`
}

db.AutoMigrate(&User{})
db.Create(&User{Name: "Alice", Email: "alice@example.com"})
db.Where("name = ?", "Alice").First(&user)
db.Save(&user)
db.Delete(&user)
```

### Key Database Concepts
- Connection pooling (`SetMaxOpenConns`, `SetMaxIdleConns`)
- Transactions and rollbacks
- N+1 query problem → use joins / eager loading
- Database migrations (golang-migrate, goose)
- Prepared statements for security

### Caching — Redis
```go
import "github.com/redis/go-redis/v9"

rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})

rdb.Set(ctx, "user:1", jsonBytes, 10*time.Minute)
val, err := rdb.Get(ctx, "user:1").Result()
if errors.Is(err, redis.Nil) {
    // cache miss — fetch from DB, then set
}
```

---

## Phase 7 — Testing & Debugging

### Unit Tests
```go
// user_test.go
func TestCreateUser(t *testing.T) {
    user := NewUser("Alice", "alice@example.com")
    if user.Name != "Alice" {
        t.Errorf("got %q, want %q", user.Name, "Alice")
    }
}

// Table-driven tests (idiomatic Go)
func TestDivide(t *testing.T) {
    tests := []struct {
        name    string
        a, b    float64
        want    float64
        wantErr bool
    }{
        {"normal", 10, 2, 5, false},
        {"divide by zero", 10, 0, 0, true},
    }
    for _, tc := range tests {
        t.Run(tc.name, func(t *testing.T) {
            got, err := divide(tc.a, tc.b)
            if (err != nil) != tc.wantErr {
                t.Fatalf("err = %v, wantErr %v", err, tc.wantErr)
            }
            if got != tc.want {
                t.Errorf("got %v, want %v", got, tc.want)
            }
        })
    }
}
```

### Benchmarks & Race Detection
```go
func BenchmarkFibonacci(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Fibonacci(20)
    }
}

// Run:
go test -bench=. -benchmem       // benchmark
go test -race ./...               // race detector
go test -cover ./...              // coverage
go test -coverprofile=cov.out
go tool cover -html=cov.out
```

### Key Libraries
| Library | Purpose |
|---------|---------|
| `testify` | Assertions, mocks (`assert`, `require`, `mock`) |
| `gomock` | Interface mock generation |
| `httptest` | HTTP handler testing without real server |
| `Delve (dlv)` | Go debugger |
| `pprof` | CPU and memory profiling |

---

## Phase 8 — Design Patterns

### Creational
- **Factory Method** — `NewX()` returning interface
- **Abstract Factory** — family of related objects
- **Builder** — step-by-step construction + `Build()`
- **Singleton** — `sync.Once` for thread-safe init
- **Functional Options** — Go's idiomatic config pattern

### Structural
- **Adapter** — wrap incompatible interface
- **Decorator** — embed interface, override one method
- **Proxy** — same interface, different behaviour (caching, auth)
- **Facade** — simple interface over complex subsystem

### Behavioural
- **Strategy** — swap algorithm at runtime via interface
- **Observer** — pub/sub with channels
- **Command** — encapsulate request as object
- **Iterator** — `range` + channel-based iterators

### Go-Specific Patterns
- **Pipeline** — chain of goroutine stages connected by channels
- **Worker Pool** — fixed goroutines consuming from a jobs channel
- **Fan-out / Fan-in** — distribute + collect work
- **Nil channel trick** — disable select cases dynamically

---

## Phase 9 — Microservices & gRPC

### gRPC
```protobuf
// user.proto
service UserService {
    rpc GetUser(GetUserRequest) returns (User);
    rpc ListUsers(ListUsersRequest) returns (stream User);
}

message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
}
```

```go
// Server implementation
type userServer struct {
    pb.UnimplementedUserServiceServer
    repo UserRepository
}

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    user, err := s.repo.FindByID(ctx, int(req.Id))
    if err != nil {
        return nil, status.Errorf(codes.NotFound, "user %d not found", req.Id)
    }
    return &pb.User{Id: int32(user.ID), Name: user.Name, Email: user.Email}, nil
}
```

### Message Queues
- **Apache Kafka** (`segmentio/kafka-go`) — high throughput event streaming
- **RabbitMQ** (`amqp091-go`) — traditional message queue
- **NATS** — lightweight pub/sub

### Service Communication Patterns
| Pattern | Tool |
|---------|------|
| REST | net/http, Gin, Echo |
| gRPC | google.golang.org/grpc |
| Event-driven | Kafka, NATS, RabbitMQ |
| WebSocket | gorilla/websocket, nhooyr.io/websocket |

---

## Phase 10 — DevOps & Cloud

### Docker
```dockerfile
# Multi-stage build — small final image
FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./cmd/server

FROM scratch
COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

### Kubernetes — Key Concepts
- Pods, Deployments, Services, Ingress
- ConfigMaps, Secrets
- Health checks: `livenessProbe`, `readinessProbe`
- Horizontal Pod Autoscaler
- Go client: `k8s.io/client-go`

### Observability
```go
// Structured logging (slog — Go 1.21+)
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
logger.Info("request processed",
    "method", r.Method,
    "path", r.URL.Path,
    "duration", time.Since(start),
    "status", statusCode,
)

// Metrics — Prometheus
import "github.com/prometheus/client_golang/prometheus"
httpRequests := prometheus.NewCounterVec(
    prometheus.CounterOpts{Name: "http_requests_total"},
    []string{"method", "path", "status"},
)
```

### Cloud Platforms
| Platform | Key Services |
|----------|-------------|
| AWS | ECS/EKS, Lambda, RDS, SQS/SNS, S3 |
| GCP | GKE, Cloud Run, Cloud SQL, Pub/Sub |
| Azure | AKS, Container Apps, Cosmos DB |

---

## Phase 11 — CLI Development

```go
// Cobra — used by kubectl, hugo, GitHub CLI
import "github.com/spf13/cobra"

var rootCmd = &cobra.Command{
    Use:   "myapp",
    Short: "My CLI application",
}

var serveCmd = &cobra.Command{
    Use:   "serve",
    Short: "Start the server",
    RunE: func(cmd *cobra.Command, args []string) error {
        port, _ := cmd.Flags().GetInt("port")
        return startServer(port)
    },
}

func init() {
    serveCmd.Flags().IntP("port", "p", 8080, "port to listen on")
    rootCmd.AddCommand(serveCmd)
}

func main() {
    rootCmd.Execute()
}
```

---

## Portfolio Projects to Build

| Project | Skills Demonstrated |
|---------|-------------------|
| **REST API** (users, auth, CRUD) | net/http or Gin, Postgres, JWT, testing |
| **CLI tool** (task manager, notes app) | Cobra, file I/O, config |
| **Chat application** | WebSockets, goroutines, channels |
| **Rate limiter** | token bucket, sync primitives |
| **Web scraper** | net/http, goquery, concurrency |
| **Microservice pair** (gRPC) | protobuf, gRPC, Docker |
| **Distributed job queue** | Redis/Kafka, worker pool |
| **Kubernetes operator** | client-go, controller-runtime |

**Tip:** Each project should have: README, tests (`>80% coverage`), Dockerfile, CI config (GitHub Actions).

---

## Common Interview Questions

### Language Fundamentals
- What's the difference between `make` and `new`?
- How does `defer` work? What order do multiple defers execute?
- When does a goroutine panic propagate?
- Difference between value receiver and pointer receiver?
- What is interface satisfaction in Go? How is it checked?
- How does Go handle multiple return values / error handling?

### Concurrency
- What is a goroutine? How is it different from an OS thread?
- What is a channel? Buffered vs unbuffered?
- What causes a deadlock in Go? How do you detect it?
- Explain the `select` statement with example.
- How do you cancel goroutines? (context.WithCancel)
- What is a race condition? How do you find them? (`go test -race`)
- What is `sync.WaitGroup`? When would you use `errgroup`?

### Internals
- How does Go's garbage collector work? (tricolor mark-and-sweep, concurrent)
- What is escape analysis? (`go build -gcflags="-m"`)
- How does Go's scheduler work? (GMP model: G=goroutine, M=OS thread, P=processor)
- What is `GOMAXPROCS`?

### Design & Architecture
- How do you structure a large Go application?
- Explain "accept interfaces, return structs."
- What is the difference between Factory Method and Abstract Factory?
- How do you implement dependency injection in Go?
- What patterns do you use for error handling across layers?

### System Design
- Design a rate limiter in Go.
- Design a URL shortener with Go + Redis + Postgres.
- How would you handle graceful shutdown?

```go
// Graceful shutdown — must know
srv := &http.Server{Addr: ":8080", Handler: router}

go func() {
    if err := srv.ListenAndServe(); err != http.ErrServerClosed {
        log.Fatal(err)
    }
}()

quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
srv.Shutdown(ctx)
```

---

## Recommended Learning Resources

| Resource | Type | Level |
|----------|------|-------|
| [go.dev/tour](https://go.dev/tour) | Interactive | Beginner |
| [gobyexample.com](https://gobyexample.com) | Examples | Beginner |
| [Effective Go](https://go.dev/doc/effective_go) | Official guide | Intermediate |
| [roadmap.sh/golang](https://roadmap.sh/golang) | Visual roadmap | All levels |
| [The Go Programming Language (book)](https://www.gopl.io) | Book | Intermediate |
| [Go with Tests](https://quii.gitbook.io/learn-go-with-tests) | TDD approach | Intermediate |
| [refactoring.guru/go](https://refactoring.guru/design-patterns/go) | Design patterns | Intermediate |
| [ardanlabs.com/blog](https://www.ardanlabs.com/blog/) | Deep dives | Advanced |
| [Dave Cheney's blog](https://dave.cheney.net) | Best practices | Advanced |

---

## Rough Timeline

| Phase | Time | Focus |
|-------|------|-------|
| Fundamentals | 2–4 weeks | Syntax, types, error handling, toolchain |
| Internals + Concurrency | 3–5 weeks | Goroutines, channels, sync, context |
| Standard library + Web | 2–3 weeks | net/http, JSON, DB, one framework |
| Testing + Patterns | 2 weeks | Table tests, mocks, common patterns |
| First project | 2–3 weeks | REST API + DB + auth + Docker |
| Microservices / gRPC | 3–4 weeks | protobuf, gRPC, Kafka basics |
| **Job-ready** | ~4–6 months total | With 2–3 solid portfolio projects |

---

## Quick Checklist Before Applying

- [ ] Understand goroutines, channels, select, context deeply
- [ ] Can explain interface implicit satisfaction
- [ ] Know the difference between value and pointer receivers
- [ ] Comfortable with table-driven tests and `-race` flag
- [ ] Built at least one REST API with auth, Postgres, Docker
- [ ] Know graceful shutdown pattern
- [ ] Can describe GMP scheduler at a high level
- [ ] Know `errors.Is` / `errors.As` / `%w` wrapping
- [ ] Understand `sync.Once`, `sync.Pool`, `sync.WaitGroup`
- [ ] Have 2+ projects on GitHub with READMEs and tests
