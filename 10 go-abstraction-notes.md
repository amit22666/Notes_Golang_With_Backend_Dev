# Go Abstraction — Master Golang with Abstraction

**Source:** https://www.youtube.com/watch?v=CRY4_-p5FgM&list=PL7g1jYj15RUMMCMDYPyZHN3CaWbt3Rl5y&index=5  
**Duration:** ~3 minutes  
**Playlist:** Golang in X Seconds

---

## Prerequisites

- Go interfaces (implicit implementation, method sets)
- Structs and methods
- Packages and import system

---

## Timeline

| Timestamp | Topic |
|-----------|-------|
| 0:00 | What is abstraction? Why it matters |
| 0:20 | Interfaces as Go's abstraction mechanism |
| 0:40 | Concrete types vs abstractions |
| 1:00 | "Accept interfaces, return structs" |
| 1:20 | Dependency injection via constructor |
| 1:45 | Layered architecture — domain/app/infra |
| 2:10 | Testing with interface mocks |
| 2:35 | When NOT to abstract (YAGNI) |

---

## 1. What Is Abstraction?

Abstraction means **hiding implementation details** behind a stable interface.  
Callers work with the interface; they don't care how it's implemented.

In Go there are no abstract classes or inheritance. Abstraction is achieved entirely through **interfaces**.

```
+------------------+        +--------------------+
|   High-level     |        |   Low-level        |
|   (business      |------->|   (infrastructure  |
|    logic)        | BAD!   |    concretions)    |
+------------------+        +--------------------+

+------------------+        +--------------------+
|   High-level     |        |   Low-level        |
|   (business      |<- I  ->|   (implementations)|
|    logic)        |  F     |                    |
+------------------+  A     +--------------------+
                       C
                       E  ← both depend on abstraction
```

---

## 2. Interfaces as the Abstraction Tool

```go
// The abstraction: defines behavior, no implementation
type Notifier interface {
    Send(to, message string) error
}

// Concrete implementation 1
type EmailNotifier struct {
    FromAddress string
}

func (e *EmailNotifier) Send(to, message string) error {
    fmt.Printf("Email → %s: %s\n", to, message)
    return nil
}

// Concrete implementation 2
type SMSNotifier struct {
    APIKey string
}

func (s *SMSNotifier) Send(to, message string) error {
    fmt.Printf("SMS → %s: %s\n", to, message)
    return nil
}
```

`NotificationService` doesn't know which notifier it uses — only that it can call `Send`.

---

## 3. Concrete vs Abstract Dependencies

**Without abstraction** — high-level code is glued to low-level details:

```go
// BAD: UserService directly depends on *PostgresDB
type UserService struct {
    db *PostgresDB // concrete type — hard to test, hard to swap
}

func NewUserService() *UserService {
    return &UserService{db: &PostgresDB{ConnStr: "postgres://..."}}
}
```

Changing the database requires rewriting `UserService`. Testing requires a real Postgres instance.

**With abstraction** — high-level code depends only on a contract:

```go
// GOOD: UserService depends on an interface
type UserRepository interface {
    GetByID(id int) (*User, error)
    Save(u *User) error
}

type UserService struct {
    repo UserRepository // abstraction — any implementation works
}

func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}
```

---

## 4. "Accept Interfaces, Return Structs"

This is Go's most important abstraction idiom — coined by Jack Lindamood.

**Accept interfaces** → function parameters are interfaces, maximising flexibility.  
**Return structs** → return concrete types so callers have access to all methods.

```go
// producer package (db)
type Store struct{ db *sql.DB }

func NewStore(connStr string) *Store { // returns concrete *Store
    db, _ := sql.Open("postgres", connStr)
    return &Store{db: db}
}

func (s *Store) Insert(u *User) error { /* ... */ return nil }
func (s *Store) GetByID(id int) (*User, error) { /* ... */ return nil, nil }
func (s *Store) Delete(id int) error { /* ... */ return nil }
```

```go
// consumer package (user service) — defines its OWN interface
// Only declares the methods IT needs, not everything Store has
type UserStore interface {
    Insert(u *User) error
    GetByID(id int) (*User, error)
}

type UserService struct {
    store UserStore
}

func NewUserService(s UserStore) *UserService {
    return &UserService{store: s}
}
```

Key insight: the **consumer** defines the interface in its own package.  
The **producer** (Store) doesn't know it implements `UserStore` — Go's implicit satisfaction handles that.

Benefits:
- Swap Postgres for MySQL without touching `UserService`
- Test with an in-memory mock, no database required
- Add methods to `Store` without breaking `UserStore`

---

## 5. Dependency Injection via Constructor

Don't create dependencies inside a struct — **inject them through the constructor**.

```go
// BAD: tight coupling, can't test or swap
type OrderService struct{}

func NewOrderService() *OrderService {
    return &OrderService{} // builds its own dependencies internally
}

func (s *OrderService) PlaceOrder(userID int) error {
    db := &PostgresDB{}   // hardcoded
    mailer := &SMTPMailer{} // hardcoded
    // ...
    return nil
}
```

```go
// GOOD: dependencies injected from outside
type OrderRepository interface {
    Save(order *Order) error
}

type Mailer interface {
    Send(to, subject, body string) error
}

type OrderService struct {
    repo   OrderRepository
    mailer Mailer
}

func NewOrderService(repo OrderRepository, mailer Mailer) *OrderService {
    return &OrderService{repo: repo, mailer: mailer}
}

func (s *OrderService) PlaceOrder(userID int) error {
    order := &Order{UserID: userID}
    if err := s.repo.Save(order); err != nil {
        return err
    }
    return s.mailer.Send("user@example.com", "Order placed", "Thanks!")
}
```

`main()` (or your DI framework) wires the real implementations together:

```go
func main() {
    repo := db.NewOrderRepository(pgConn)
    mailer := smtp.NewMailer(smtpConfig)
    svc := order.NewOrderService(repo, mailer)
    // ...
}
```

---

## 6. Layered Architecture

Abstraction enables clean layers. Each layer only knows the layer above via interfaces.

```
┌──────────────────────────┐
│       main.go            │  Wires everything together
├──────────────────────────┤
│    HTTP / gRPC handlers  │  Receives request, calls service
├──────────────────────────┤
│    Application / Service │  Business logic, depends on repo interface
├──────────────────────────┤
│    Domain (entities)     │  Pure data types, no external deps
├──────────────────────────┤
│    Infrastructure / DB   │  Implements repo interfaces, talks to DB
└──────────────────────────┘
```

```go
// domain/user.go — pure entity, no deps
package domain

type User struct {
    ID    int
    Email string
    Name  string
}

// Repository interface lives in the domain layer (where it's used)
type UserRepository interface {
    GetByID(id int) (*User, error)
    Save(u *User) error
}

// application/user_service.go — business logic
package application

type UserService struct {
    repo domain.UserRepository
}

func NewUserService(repo domain.UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) Register(email, name string) (*domain.User, error) {
    user := &domain.User{Email: email, Name: name}
    return user, s.repo.Save(user)
}

// infrastructure/postgres_user_repo.go — concrete implementation
package infrastructure

type PostgresUserRepo struct {
    db *sql.DB
}

func NewPostgresUserRepo(db *sql.DB) *PostgresUserRepo {
    return &PostgresUserRepo{db: db}
}

func (r *PostgresUserRepo) GetByID(id int) (*domain.User, error) {
    u := &domain.User{}
    err := r.db.QueryRow("SELECT id, email, name FROM users WHERE id=$1", id).
        Scan(&u.ID, &u.Email, &u.Name)
    return u, err
}

func (r *PostgresUserRepo) Save(u *domain.User) error {
    _, err := r.db.Exec("INSERT INTO users(email, name) VALUES($1,$2)", u.Email, u.Name)
    return err
}
```

The `application` layer imports `domain` (interface + entity). It never imports `infrastructure`.  
`main` imports all three and wires them:

```go
func main() {
    db, _ := sql.Open("postgres", os.Getenv("DB_URL"))
    repo := infrastructure.NewPostgresUserRepo(db)
    svc := application.NewUserService(repo)
    // pass svc to HTTP handlers
}
```

---

## 7. Testing with Interface Mocks

Abstraction makes unit tests fast and dependency-free.

```go
// Simple in-line mock — no mocking library needed
type mockUserRepo struct {
    users map[int]*domain.User
}

func (m *mockUserRepo) GetByID(id int) (*domain.User, error) {
    u, ok := m.users[id]
    if !ok {
        return nil, errors.New("not found")
    }
    return u, nil
}

func (m *mockUserRepo) Save(u *domain.User) error {
    u.ID = len(m.users) + 1
    m.users[u.ID] = u
    return nil
}

func TestRegister(t *testing.T) {
    repo := &mockUserRepo{users: make(map[int]*domain.User)}
    svc := application.NewUserService(repo)

    user, err := svc.Register("alice@example.com", "Alice")
    if err != nil {
        t.Fatal(err)
    }
    if user.Email != "alice@example.com" {
        t.Errorf("got %s, want alice@example.com", user.Email)
    }
}
```

No database. No network. Runs instantly.

---

## 8. Function Type as Interface (Lightweight Abstraction)

For a single-method interface, a function type is often cleaner:

```go
type Processor interface {
    Process(data []byte) error
}

// A function that matches the signature automatically satisfies Processor:
type ProcessorFunc func(data []byte) error

func (f ProcessorFunc) Process(data []byte) error {
    return f(data)
}

// Usage
func Run(p Processor, data []byte) error {
    return p.Process(data)
}

// Any function becomes a Processor with a simple cast:
Run(ProcessorFunc(func(d []byte) error {
    fmt.Println(string(d))
    return nil
}), []byte("hello"))
```

The standard library does this with `http.HandlerFunc`.

---

## 9. Compile-Time Interface Check

Verify a type satisfies an interface before it causes a runtime panic:

```go
// Place near the type definition
var _ domain.UserRepository = (*infrastructure.PostgresUserRepo)(nil)

// Fails at compile time if PostgresUserRepo is missing a method
```

---

## 10. SOLID Principles — Go Style

| Principle | Go Idiom |
|-----------|----------|
| **S**ingle Responsibility | One package, one clear purpose. Avoid `util`, `common`, `server` |
| **O**pen/Closed | Extend via embedding; don't modify existing types |
| **L**iskov Substitution | Any implementation satisfying the interface can be swapped |
| **I**nterface Segregation | Keep interfaces small (1–3 methods); `io.Reader` > `io.ReadWriteCloser` |
| **D**ependency Inversion | High-level code depends on interfaces, not concrete types |

```go
// ISP example: prefer the narrowest interface you need
func Save(w io.Writer, doc *Document) error { // NOT io.ReadWriteCloser
    return json.NewEncoder(w).Encode(doc)
}
// Now Save works with files, buffers, network conns, test writers — anything
```

---

## 11. When NOT to Abstract (YAGNI)

Over-engineering is a real risk. Don't add interfaces speculatively.

**Don't abstract** when:
- There is only one implementation and no test mock needed
- The abstraction adds no testability or flexibility
- You're abstracting for the sake of it

```go
// Overkill for a simple config loader with one implementation
type ConfigLoader interface {
    Load() (*Config, error)
}

// Just use the concrete type if there's only ever one
func LoadConfig(path string) (*Config, error) { /* ... */ return nil, nil }
```

**Rob Pike's rule:** "Don't design with interfaces, discover them."  
Start with concrete types. Extract an interface when you have a real second implementation or when you need a test double.

---

## 12. Quick Reference

```go
// Define abstraction (in consumer package)
type Repo interface {
    FindByID(id int) (*Entity, error)
    Save(e *Entity) error
}

// Concrete implementation (in producer package)
type PostgresRepo struct{ db *sql.DB }
func (r *PostgresRepo) FindByID(id int) (*Entity, error) { /* ... */ return nil, nil }
func (r *PostgresRepo) Save(e *Entity) error { /* ... */ return nil }

// Constructor injection (accept interface, return struct)
type Service struct{ repo Repo }
func NewService(r Repo) *Service { return &Service{repo: r} }

// Compile-time check
var _ Repo = (*PostgresRepo)(nil)

// Test mock (in-line, no library)
type fakeRepo struct{ saved []*Entity }
func (f *fakeRepo) FindByID(id int) (*Entity, error) { return nil, nil }
func (f *fakeRepo) Save(e *Entity) error { f.saved = append(f.saved, e); return nil }

// Wire in main
func main() {
    db, _ := sql.Open("postgres", dsn)
    svc := NewService(&PostgresRepo{db: db})
    _ = svc
}
```

---

## 13. Summary

| Concept | Key Point |
|---------|-----------|
| Abstraction in Go | Done entirely with interfaces — no abstract classes |
| Implicit satisfaction | No `implements` keyword; any type with the methods qualifies |
| Accept interfaces | Parameters as interfaces → flexible, testable |
| Return structs | Concrete return types → full API accessible to callers |
| Constructor injection | Pass dependencies in; don't build them inside |
| Interface in consumer | Consumer defines the interface it needs, not the producer |
| Layered architecture | Domain → Application → Infrastructure; each layer sees only interfaces |
| Mocking | Implement the interface in a test struct; no library needed |
| YAGNI | Only abstract when you have a real second use case |

---

## Further Resources

- [Dave Cheney — SOLID Go Design](https://dave.cheney.net/2016/08/20/solid-go-design)
- [Effective Go — Interfaces](https://go.dev/doc/effective_go#interfaces)
- ["Accept Interfaces, Return Structs" — Bryan F Tan](https://bryanftan.medium.com/accept-interfaces-return-structs-in-go-d4cab29a301b)
- [Three Dots Labs — Repository Pattern in Go](https://threedots.tech/post/repository-pattern-in-go/)
- [Go Blog — Errors are values](https://go.dev/blog/errors-are-values)
