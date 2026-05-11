# Factory Method Pattern — Visualized in Go

**Source:** https://www.youtube.com/watch?v=y0HRazQsvUY&list=PL7g1jYj15RUO-crQOgDV0_dVp2OaoSgye&index=3  
**Playlist:** Design Patterns (Visualized)  
**Category:** Creational Patterns

---

## Prerequisites

- Go interfaces and implicit implementation
- Structs, methods, embedding
- Basic dependency injection

---

## Timeline

| Timestamp | Topic |
|-----------|-------|
| 0:00 | Problem: tight coupling to concrete types |
| 0:30 | Factory Method — definition and intent |
| 1:00 | UML structure: Product, Creator, Concrete* |
| 1:30 | Go's take — no classes, use interfaces |
| 2:00 | Simple factory function (NewX pattern) |
| 2:30 | Interface factory — hide concrete type |
| 3:00 | Creator interface with factory method |
| 3:30 | Gun factory — full walkthrough |
| 4:30 | Payment gateway — real-world example |
| 5:30 | Factory generator / closure variant |
| 6:00 | Functional options factory |
| 6:30 | When to use vs when to avoid |

---

## 1. The Problem Factory Method Solves

Without a factory, client code is tightly coupled to concrete types:

```go
// BAD: client must know the concrete type
func processPayment(method string) {
    if method == "paypal" {
        gw := &PayPalGateway{ClientID: "...", Secret: "..."}
        gw.ProcessPayment(100)
    } else if method == "stripe" {
        gw := &StripeGateway{APIKey: "..."}
        gw.ProcessPayment(100)
    }
    // Adding a new gateway means editing THIS function
}
```

Problems:
- Adding a new type requires modifying existing code (violates Open/Closed)
- Client knows concrete struct names and their setup details
- Cannot swap implementations for testing

Factory Method fixes this by delegating creation to a dedicated function/method.

---

## 2. What Is the Factory Method Pattern?

> "Define an interface for creating an object, but let subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses."
> — Gang of Four

**Intent:** Create objects without specifying their exact concrete type.  
**Category:** Creational pattern.

### Classical OOP Structure

```
<<interface>>          <<interface>>
   Product    <------    Creator
      ^                    ^
      |                    |
ConcreteProduct    ConcreteCreator
                    factoryMethod() → ConcreteProduct
```

### Go's Reality

Go has no classes or inheritance, so the pattern maps to:

| OOP Term | Go Equivalent |
|----------|---------------|
| Product interface | `type Product interface { ... }` |
| ConcreteProduct | struct implementing the interface |
| Creator | factory function or Creator interface |
| ConcreteCreator | struct with `CreateProduct()` method |

---

## 3. Variant 1 — Simple Factory Function (NewX)

The most common Go idiom. A `New…` function ensures required fields are set.

```go
type Person struct {
    Name string
    Age  int
}

// Simple factory — enforces required fields, hides zero-value pitfalls
func NewPerson(name string, age int) *Person {
    return &Person{Name: name, Age: age}
}

p := NewPerson("Alice", 30)
```

**When zero values are sensible, skip the factory:**

```go
type Counter struct{ count int }

// No factory needed — Counter{} already works perfectly
c := Counter{}
c.count++ // fine
```

---

## 4. Variant 2 — Interface Factory (Hide the Concrete Type)

Return an **interface**, not a concrete struct. The caller only sees the contract.

```go
type Greeter interface {
    Greet() string
}

// unexported — caller can never reference this type directly
type englishGreeter struct{ name string }

func (e englishGreeter) Greet() string { return "Hello, " + e.name + "!" }

type spanishGreeter struct{ name string }

func (s spanishGreeter) Greet() string { return "¡Hola, " + s.name + "!" }

// Factory — client works with Greeter, never with the concrete structs
func NewGreeter(lang, name string) (Greeter, error) {
    switch lang {
    case "en":
        return englishGreeter{name: name}, nil
    case "es":
        return spanishGreeter{name: name}, nil
    default:
        return nil, fmt.Errorf("unsupported language: %s", lang)
    }
}

g, _ := NewGreeter("es", "Ana")
fmt.Println(g.Greet()) // ¡Hola, Ana!
```

---

## 5. Variant 3 — Creator Interface (Classic GoF Style)

Closest to the original GoF pattern. A `Creator` interface declares the factory method; concrete creators implement it.

```go
// Product interface
type Product interface {
    GetName() string
}

// Creator interface — the factory method
type Creator interface {
    CreateProduct() Product
}

// Concrete product A
type WidgetA struct{}
func (w *WidgetA) GetName() string { return "Widget A" }

// Concrete product B
type WidgetB struct{}
func (w *WidgetB) GetName() string { return "Widget B" }

// Concrete creator A
type CreatorA struct{}
func (c *CreatorA) CreateProduct() Product { return &WidgetA{} }

// Concrete creator B
type CreatorB struct{}
func (c *CreatorB) CreateProduct() Product { return &WidgetB{} }

func clientCode(creator Creator) {
    product := creator.CreateProduct()
    fmt.Println("Created:", product.GetName())
}

clientCode(&CreatorA{}) // Created: Widget A
clientCode(&CreatorB{}) // Created: Widget B
```

---

## 6. Full Example — Gun Factory (refactoring.guru)

A classic factory method walkthrough showing multiple concrete products.

**iGun.go — Product interface:**

```go
type IGun interface {
    setName(name string)
    setPower(power int)
    getName() string
    getPower() int
}
```

**gun.go — Base struct (shared behaviour via embedding):**

```go
type Gun struct {
    name  string
    power int
}

func (g *Gun) setName(name string)  { g.name = name }
func (g *Gun) getName() string      { return g.name }
func (g *Gun) setPower(power int)   { g.power = power }
func (g *Gun) getPower() int        { return g.power }
```

**ak47.go — Concrete product:**

```go
type Ak47 struct {
    Gun // embed base — gets all IGun methods for free
}

func newAk47() IGun {
    return &Ak47{
        Gun: Gun{name: "AK47 gun", power: 4},
    }
}
```

**musket.go — Concrete product:**

```go
type musket struct {
    Gun
}

func newMusket() IGun {
    return &musket{
        Gun: Gun{name: "Musket gun", power: 1},
    }
}
```

**gunFactory.go — The factory function:**

```go
func getGun(gunType string) (IGun, error) {
    switch gunType {
    case "ak47":
        return newAk47(), nil
    case "musket":
        return newMusket(), nil
    default:
        return nil, fmt.Errorf("unknown gun type: %s", gunType)
    }
}
```

**main.go — Client code:**

```go
func main() {
    ak47, err := getGun("ak47")
    if err != nil {
        log.Fatal(err)
    }
    musket, err := getGun("musket")
    if err != nil {
        log.Fatal(err)
    }

    printDetails(ak47)
    printDetails(musket)
}

func printDetails(g IGun) {
    fmt.Printf("Gun: %s\n", g.getName())
    fmt.Printf("Power: %d\n", g.getPower())
}
// Output:
// Gun: AK47 gun
// Power: 4
// Gun: Musket gun
// Power: 1
```

Adding a `sniper` gun requires only a new file with a `sniper` struct — zero changes to `getGun`'s callers.

---

## 7. Real-World Example — Payment Gateway

```go
type PaymentGateway interface {
    ProcessPayment(amount float64) error
}

// Concrete products
type PayPalGateway struct {
    ClientID string
    Secret   string
}

func (p *PayPalGateway) ProcessPayment(amount float64) error {
    fmt.Printf("PayPal: charging $%.2f (client: %s)\n", amount, p.ClientID)
    return nil
}

type StripeGateway struct {
    APIKey string
}

func (s *StripeGateway) ProcessPayment(amount float64) error {
    fmt.Printf("Stripe: charging $%.2f\n", amount)
    return nil
}

// Factory using iota constants for type safety
type GatewayType int

const (
    PayPal GatewayType = iota
    Stripe
)

func NewPaymentGateway(t GatewayType) (PaymentGateway, error) {
    switch t {
    case PayPal:
        return &PayPalGateway{
            ClientID: os.Getenv("PAYPAL_CLIENT_ID"),
            Secret:   os.Getenv("PAYPAL_SECRET"),
        }, nil
    case Stripe:
        return &StripeGateway{
            APIKey: os.Getenv("STRIPE_API_KEY"),
        }, nil
    default:
        return nil, fmt.Errorf("unsupported gateway: %d", t)
    }
}

// Client code — works with ANY PaymentGateway
func checkout(gw PaymentGateway, amount float64) error {
    return gw.ProcessPayment(amount)
}

func main() {
    gw, err := NewPaymentGateway(Stripe)
    if err != nil {
        log.Fatal(err)
    }
    checkout(gw, 99.99) // Stripe: charging $99.99
}
```

---

## 8. Testing with Interface Factory

Factory + interface makes swapping a real implementation for a test double trivial.

```go
type Doer interface {
    Do(req *http.Request) (*http.Response, error)
}

// Production factory
func NewHTTPClient() Doer {
    return &http.Client{Timeout: 10 * time.Second}
}

// Test factory — no real network calls
type mockHTTPClient struct {
    Response *http.Response
}

func (m *mockHTTPClient) Do(req *http.Request) (*http.Response, error) {
    return m.Response, nil
}

func NewMockHTTPClient(resp *http.Response) Doer {
    return &mockHTTPClient{Response: resp}
}

// Service depends on the interface
type APIService struct {
    client Doer
}

func NewAPIService(client Doer) *APIService {
    return &APIService{client: client}
}

// Test — inject mock, no network
func TestAPIService(t *testing.T) {
    mock := NewMockHTTPClient(&http.Response{StatusCode: 200, Body: http.NoBody})
    svc := NewAPIService(mock)
    _ = svc // test svc.FetchData() etc.
}
```

---

## 9. Variant 4 — Factory Generator (Factory of Factories)

A factory that produces other factories — useful for encoding shared configuration.

```go
type Animal struct {
    Species string
    Age     int
}

type AnimalFactory struct {
    species string
}

// The factory generator returns a configured factory
func NewAnimalFactory(species string) *AnimalFactory {
    return &AnimalFactory{species: species}
}

// The factory method on the factory
func (af *AnimalFactory) New(age int) Animal {
    return Animal{Species: af.species, Age: age}
}

// Usage
dogFactory := NewAnimalFactory("dog")
catFactory := NewAnimalFactory("cat")

dog1 := dogFactory.New(2)
dog2 := dogFactory.New(5)
cat1 := catFactory.New(3)
```

---

## 10. Variant 5 — Closure Factory (Functional Style)

Encode defaults in a closure; return a constructor function.

```go
type Person struct {
    Name string
    Age  int
}

// Returns a constructor specialised for a given age group
func NewPersonFactory(defaultAge int) func(name string) Person {
    return func(name string) Person {
        return Person{Name: name, Age: defaultAge}
    }
}

newBaby     := NewPersonFactory(0)
newTeenager := NewPersonFactory(16)
newAdult    := NewPersonFactory(30)

baby := newBaby("Sam")       // {Sam 0}
teen := newTeenager("Jill")  // {Jill 16}
```

---

## 11. Variant 6 — Functional Options Factory

Combine factory with the functional options pattern for flexible configuration.

```go
type Server struct {
    host    string
    port    int
    timeout time.Duration
}

type Option func(*Server)

func WithHost(host string) Option {
    return func(s *Server) { s.host = host }
}

func WithPort(port int) Option {
    return func(s *Server) { s.port = port }
}

func WithTimeout(d time.Duration) Option {
    return func(s *Server) { s.timeout = d }
}

func NewServer(opts ...Option) *Server {
    // sensible defaults
    s := &Server{
        host:    "localhost",
        port:    8080,
        timeout: 30 * time.Second,
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Usage — only supply what differs from defaults
s := NewServer(
    WithPort(9090),
    WithTimeout(60*time.Second),
)
```

This is Go's idiomatic answer to constructor overloading.

---

## 12. Notification System — Logger Factory

```go
type Logger interface {
    Log(msg string)
}

type ConsoleLogger struct{}
func (l *ConsoleLogger) Log(msg string) { fmt.Println("[CONSOLE]", msg) }

type FileLogger struct{ path string }
func (l *FileLogger) Log(msg string) {
    // append to file
    fmt.Printf("[FILE:%s] %s\n", l.path, msg)
}

type JSONLogger struct{}
func (l *JSONLogger) Log(msg string) {
    fmt.Printf(`{"level":"info","msg":%q}`+"\n", msg)
}

func NewLogger(logType, param string) (Logger, error) {
    switch logType {
    case "console":
        return &ConsoleLogger{}, nil
    case "file":
        return &FileLogger{path: param}, nil
    case "json":
        return &JSONLogger{}, nil
    default:
        return nil, fmt.Errorf("unknown logger: %s", logType)
    }
}

logger, _ := NewLogger("json", "")
logger.Log("server started") // {"level":"info","msg":"server started"}
```

---

## 13. Pattern Structure Summary

```
┌──────────────────────────────────────────────────────┐
│                  Factory Method                       │
├──────────────────┬───────────────────────────────────┤
│ Component        │ Go Implementation                 │
├──────────────────┼───────────────────────────────────┤
│ Product          │ interface { Method() }            │
│ ConcreteProduct  │ struct implementing interface     │
│ Creator          │ factory func or Creator interface │
│ ConcreteCreator  │ struct with CreateProduct()       │
└──────────────────┴───────────────────────────────────┘

Variants (simplest → most powerful):
  1. NewX()              — simple constructor function
  2. NewX() Interface    — hides concrete type
  3. Creator interface   — full GoF pattern
  4. Factory struct      — factory generator
  5. Closure factory     — encode defaults
  6. Functional options  — flexible configuration
```

---

## 14. When to Use Factory Method

| Situation | Use Factory? |
|-----------|-------------|
| Multiple types share an interface | Yes |
| Creation logic is complex / varies by type | Yes |
| Need to swap implementations for tests | Yes |
| Want to hide concrete type from caller | Yes |
| Adding new types without changing client code | Yes |
| Simple struct with sensible zero values | No — just use `T{}` |
| Only one implementation, ever | No — premature abstraction |

---

## 15. Quick Reference

```go
// 1. Product interface
type Notification interface {
    Send(to, msg string) error
}

// 2. Concrete products (unexported keeps them hidden)
type emailNotification struct{}
func (e *emailNotification) Send(to, msg string) error { return nil }

type smsNotification struct{}
func (s *smsNotification) Send(to, msg string) error { return nil }

// 3. Factory function — the factory method
func NewNotification(kind string) (Notification, error) {
    switch kind {
    case "email": return &emailNotification{}, nil
    case "sms":   return &smsNotification{}, nil
    default:      return nil, fmt.Errorf("unknown kind: %s", kind)
    }
}

// 4. Client — depends only on the interface
n, err := NewNotification("sms")
if err != nil { log.Fatal(err) }
n.Send("+1-555-0100", "Your code is 123456")

// 5. Add new type without touching client code:
//    type pushNotification struct{}
//    func (p *pushNotification) Send(...) error { ... }
//    case "push": return &pushNotification{}, nil
```

---

## Further Resources

- [Refactoring.guru — Factory Method in Go](https://refactoring.guru/design-patterns/factory-method/go/example)
- [Refactoring.guru — All Go Design Patterns](https://refactoring.guru/design-patterns/go)
- [Soham Kamani — Factory Patterns in Go](https://www.sohamkamani.com/golang/2018-06-20-golang-factory-patterns/)
- [DEV — Understanding Factory Method Pattern](https://dev.to/kittipat1413/understanding-the-factory-method-pattern-1gc)
- [Go by Example — Closures](https://gobyexample.com/closures)
