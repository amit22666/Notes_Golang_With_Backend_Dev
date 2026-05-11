# Builder Design Pattern — Explained in 10 Minutes (Go)

**Source:** https://www.youtube.com/watch?v=oP76NM4qZhw&list=PL7g1jYj15RUO-crQOgDV0_dVp2OaoSgye&index=1  
**Playlist:** Design Patterns (Visualized)  
**Category:** Creational Patterns

---

## Prerequisites

- Go structs, interfaces, methods
- Method receivers (value vs pointer)
- Basic Factory Method pattern

---

## Timeline

| Timestamp | Topic |
|-----------|-------|
| 0:00 | Problem: telescoping constructors |
| 1:00 | Builder pattern — definition and intent |
| 1:30 | Four participants: Product, Builder, ConcreteBuilder, Director |
| 2:00 | House example — iBuilder, NormalBuilder, IglooBuilder |
| 4:00 | Director — orchestrating construction order |
| 5:00 | Fluent builder — method chaining |
| 6:00 | Validation inside Build() |
| 7:00 | HTTP request builder — real-world example |
| 8:00 | Functional options — Go-idiomatic builder variant |
| 9:00 | Builder vs Functional Options vs Struct literal |
| 9:30 | When to use / when to avoid |

---

## 1. The Problem — Telescoping Constructors

When a struct has many optional fields, naive approaches break down fast.

```go
// BAD: positional args — what does each value mean?
cfg := NewServerConfig("localhost", 8080, false, 30, 5, "info")

// What if you only want TLS + default everything else?
cfg := NewServerConfig("localhost", 443, true, 0, 0, "")
//                                           ↑   ↑    ↑  all zeroes — confusing

// BAD: one overload per combination — doesn't scale
func NewServerConfigWithTLS(host string, port int) *ServerConfig { ... }
func NewServerConfigWithTLSAndTimeout(host string, port, timeout int) *ServerConfig { ... }
// Combinatorial explosion
```

The Builder pattern solves this by making construction a **series of named steps** rather than a single call with many arguments.

---

## 2. What Is the Builder Pattern?

> "Separate the construction of a complex object from its representation so that the same construction process can create different representations."
> — Gang of Four

**Intent:** Build complex objects step by step, keeping the partially-built object hidden until construction is complete.

**Key properties:**
- Construction happens through explicit, named method calls
- The product is only returned (via `Build()` or `getX()`) when fully assembled
- Different builders produce different products using the same steps
- A Director can encode reusable construction recipes

---

## 3. Four Participants

```
┌──────────────────────────────────────────────────────┐
│                Builder Pattern                        │
├───────────────────┬──────────────────────────────────┤
│ Participant       │ Role                              │
├───────────────────┼──────────────────────────────────┤
│ Product           │ The complex object being built    │
│ Builder           │ Interface declaring build steps   │
│ ConcreteBuilder   │ Implements steps for one variant  │
│ Director          │ Encodes a construction sequence   │
└───────────────────┴──────────────────────────────────┘
```

Director is **optional** in Go — you can call builder steps directly from the client when no standard recipes are needed.

---

## 4. Classic GoF Example — House Builder

### Product

```go
// house.go
type House struct {
    windowType string
    doorType   string
    floor      int
}
```

### Builder Interface

```go
// iBuilder.go
type IBuilder interface {
    setWindowType()
    setDoorType()
    setNumFloor()
    getHouse() House
}

func getBuilder(builderType string) IBuilder {
    switch builderType {
    case "normal":
        return newNormalBuilder()
    case "igloo":
        return newIglooBuilder()
    default:
        return nil
    }
}
```

### Concrete Builder 1 — Normal House

```go
// normalBuilder.go
type NormalBuilder struct {
    windowType string
    doorType   string
    floor      int
}

func newNormalBuilder() *NormalBuilder { return &NormalBuilder{} }

func (b *NormalBuilder) setWindowType() { b.windowType = "Wooden Window" }
func (b *NormalBuilder) setDoorType()   { b.doorType = "Wooden Door" }
func (b *NormalBuilder) setNumFloor()   { b.floor = 2 }

func (b *NormalBuilder) getHouse() House {
    return House{
        doorType:   b.doorType,
        windowType: b.windowType,
        floor:      b.floor,
    }
}
```

### Concrete Builder 2 — Igloo

```go
// iglooBuilder.go
type IglooBuilder struct {
    windowType string
    doorType   string
    floor      int
}

func newIglooBuilder() *IglooBuilder { return &IglooBuilder{} }

func (b *IglooBuilder) setWindowType() { b.windowType = "Snow Window" }
func (b *IglooBuilder) setDoorType()   { b.doorType = "Snow Door" }
func (b *IglooBuilder) setNumFloor()   { b.floor = 1 }

func (b *IglooBuilder) getHouse() House {
    return House{
        doorType:   b.doorType,
        windowType: b.windowType,
        floor:      b.floor,
    }
}
```

### Director — Encodes the Construction Order

```go
// director.go
type Director struct {
    builder IBuilder
}

func newDirector(b IBuilder) *Director { return &Director{builder: b} }

func (d *Director) setBuilder(b IBuilder) { d.builder = b }

func (d *Director) buildHouse() House {
    // Director controls the ORDER of steps
    d.builder.setDoorType()
    d.builder.setWindowType()
    d.builder.setNumFloor()
    return d.builder.getHouse()
}
```

### Client

```go
// main.go
func main() {
    normalBuilder := getBuilder("normal")
    iglooBuilder  := getBuilder("igloo")

    director := newDirector(normalBuilder)
    normalHouse := director.buildHouse()
    fmt.Printf("Normal — door: %s, window: %s, floors: %d\n",
        normalHouse.doorType, normalHouse.windowType, normalHouse.floor)

    director.setBuilder(iglooBuilder)
    iglooHouse := director.buildHouse()
    fmt.Printf("Igloo  — door: %s, window: %s, floors: %d\n",
        iglooHouse.doorType, iglooHouse.windowType, iglooHouse.floor)
}
// Output:
// Normal — door: Wooden Door, window: Wooden Window, floors: 2
// Igloo  — door: Snow Door,   window: Snow Window,   floors: 1
```

The same Director (`buildHouse`) produced two completely different houses — same process, different representations.

---

## 5. Fluent Builder — Method Chaining

Return `*Builder` from each setter to enable chaining. This is the most common Go style.

```go
type Car struct {
    color         string
    engineType    string
    hasSunroof    bool
    hasNavigation bool
}

type CarBuilder struct {
    car Car
}

func NewCarBuilder() *CarBuilder {
    return &CarBuilder{}
}

// Each setter returns *CarBuilder for chaining
func (b *CarBuilder) SetColor(color string) *CarBuilder {
    b.car.color = color
    return b
}

func (b *CarBuilder) SetEngineType(engine string) *CarBuilder {
    b.car.engineType = engine
    return b
}

func (b *CarBuilder) SetSunroof(v bool) *CarBuilder {
    b.car.hasSunroof = v
    return b
}

func (b *CarBuilder) SetNavigation(v bool) *CarBuilder {
    b.car.hasNavigation = v
    return b
}

func (b *CarBuilder) Build() Car {
    return b.car
}

// Usage — reads like English
myCar := NewCarBuilder().
    SetColor("blue").
    SetEngineType("electric").
    SetSunroof(true).
    SetNavigation(true).
    Build()
```

---

## 6. Validation in Build()

`Build()` is the ideal place to enforce invariants — the object is never partially exposed.

```go
type ServerConfig struct {
    Host       string
    Port       int
    TLS        bool
    Timeout    int
    MaxRetries int
}

type ServerConfigBuilder struct {
    host    string
    port    int
    tls     bool
    timeout int
    retries int
}

func NewServerConfigBuilder() *ServerConfigBuilder {
    return &ServerConfigBuilder{
        host:    "localhost", // sensible defaults
        port:    80,
        timeout: 30,
    }
}

func (b *ServerConfigBuilder) SetHost(h string) *ServerConfigBuilder {
    b.host = h; return b
}
func (b *ServerConfigBuilder) SetPort(p int) *ServerConfigBuilder {
    b.port = p; return b
}
func (b *ServerConfigBuilder) EnableTLS() *ServerConfigBuilder {
    b.tls = true; return b
}
func (b *ServerConfigBuilder) SetTimeout(t int) *ServerConfigBuilder {
    b.timeout = t; return b
}
func (b *ServerConfigBuilder) SetRetries(r int) *ServerConfigBuilder {
    b.retries = r; return b
}

// Build validates before handing back the product
func (b *ServerConfigBuilder) Build() (ServerConfig, error) {
    if b.host == "" {
        return ServerConfig{}, errors.New("host must not be empty")
    }
    if b.tls && b.port != 443 {
        return ServerConfig{}, errors.New("TLS requires port 443")
    }
    if b.timeout <= 0 {
        return ServerConfig{}, errors.New("timeout must be positive")
    }
    return ServerConfig{
        Host:       b.host,
        Port:       b.port,
        TLS:        b.tls,
        Timeout:    b.timeout,
        MaxRetries: b.retries,
    }, nil
}

// Usage
cfg, err := NewServerConfigBuilder().
    SetHost("api.example.com").
    SetPort(443).
    EnableTLS().
    SetRetries(3).
    Build()
if err != nil {
    log.Fatal(err)
}
```

---

## 7. Director With Presets

A Director can encode named configurations — reusable construction recipes.

```go
type ConfigDirector struct {
    builder *ServerConfigBuilder
}

func NewConfigDirector(b *ServerConfigBuilder) *ConfigDirector {
    return &ConfigDirector{builder: b}
}

func (d *ConfigDirector) BuildProduction() (ServerConfig, error) {
    return d.builder.
        SetHost("prod.internal").
        SetPort(443).
        EnableTLS().
        SetTimeout(60).
        SetRetries(5).
        Build()
}

func (d *ConfigDirector) BuildDevelopment() (ServerConfig, error) {
    return d.builder.
        SetHost("localhost").
        SetPort(8080).
        SetTimeout(5).
        SetRetries(1).
        Build()
}

director := NewConfigDirector(NewServerConfigBuilder())
prodCfg, _ := director.BuildProduction()
devCfg,  _ := director.BuildDevelopment()
```

---

## 8. Real-World Example — HTTP Request Builder

The standard library's `*http.Request` has many optional fields. A builder hides the construction complexity.

```go
type HTTPBuilder interface {
    AddHeader(name, value string) HTTPBuilder
    Body(r io.Reader) HTTPBuilder
    Method(method string) HTTPBuilder
    Close(close bool) HTTPBuilder
    Build() (*http.Request, error)
}

type httpBuilder struct {
    headers map[string][]string
    url     string
    method  string
    body    io.Reader
    close   bool
    ctx     context.Context
}

func NewHTTPBuilder(url string) HTTPBuilder {
    return &httpBuilder{
        headers: make(map[string][]string),
        url:     url,
        method:  http.MethodGet,          // sensible default
        ctx:     context.Background(),
    }
}

func (b *httpBuilder) AddHeader(name, value string) HTTPBuilder {
    b.headers[name] = append(b.headers[name], value)
    return b
}

func (b *httpBuilder) Body(r io.Reader) HTTPBuilder {
    b.body = r; return b
}

func (b *httpBuilder) Method(method string) HTTPBuilder {
    b.method = method; return b
}

func (b *httpBuilder) Close(close bool) HTTPBuilder {
    b.close = close; return b
}

func (b *httpBuilder) Build() (*http.Request, error) {
    req, err := http.NewRequestWithContext(b.ctx, b.method, b.url, b.body)
    if err != nil {
        return nil, err
    }
    for key, values := range b.headers {
        for _, v := range values {
            req.Header.Add(key, v)
        }
    }
    req.Close = b.close
    return req, nil
}

// Usage — clear and readable
req, err := NewHTTPBuilder("https://api.example.com/users").
    Method(http.MethodPost).
    AddHeader("Content-Type", "application/json").
    AddHeader("Authorization", "Bearer "+token).
    Body(bytes.NewBufferString(`{"name":"Alice"}`)).
    Build()
```

---

## 9. SQL Query Builder (Squirrel-style)

The builder pattern is the foundation of fluent SQL libraries.

```go
// Squirrel library (github.com/Masterminds/squirrel) — real-world builder
import sq "github.com/Masterminds/squirrel"

// Base query — builder state
users := sq.Select("*").From("users").Join("emails USING (email_id)")

// Extend with additional conditions — builder returns new state
active := users.Where(sq.Eq{"deleted_at": nil})
admins := active.Where(sq.Eq{"role": "admin"})

// Build — generate the final SQL
sql, args, err := admins.ToSql()
// SELECT * FROM users JOIN emails USING (email_id)
// WHERE deleted_at IS NULL AND role = ?
// args: ["admin"]
```

Each method returns a new builder (immutable style) — no mutation of shared state.

---

## 10. Builder Cloning

Share a base configuration; clone for variants.

```go
func (b *ServerConfigBuilder) Clone() *ServerConfigBuilder {
    clone := *b // copy the struct value
    return &clone
}

base := NewServerConfigBuilder().SetHost("internal.company.com").SetTimeout(30)

// Two specialised configs from the same base — no cross-contamination
apiCfg, _  := base.Clone().SetPort(8080).SetRetries(3).Build()
grpcCfg, _ := base.Clone().SetPort(50051).EnableTLS().SetRetries(5).Build()
```

---

## 11. Functional Options — Go-Idiomatic Builder Variant

Go's most idiomatic alternative to the Builder. No separate builder struct needed.

```go
type ServerConfig struct {
    Host    string
    Port    int
    TLS     bool
    Timeout time.Duration
}

type Option func(*ServerConfig)

func WithHost(h string) Option        { return func(c *ServerConfig) { c.Host = h } }
func WithPort(p int) Option           { return func(c *ServerConfig) { c.Port = p } }
func WithTLS() Option                 { return func(c *ServerConfig) { c.TLS = true } }
func WithTimeout(d time.Duration) Option {
    return func(c *ServerConfig) { c.Timeout = d }
}

func NewServerConfig(opts ...Option) *ServerConfig {
    cfg := &ServerConfig{           // set defaults here
        Host:    "localhost",
        Port:    80,
        Timeout: 30 * time.Second,
    }
    for _, opt := range opts {
        opt(cfg)
    }
    return cfg
}

// Usage — just as readable, much less boilerplate
cfg := NewServerConfig(
    WithHost("api.example.com"),
    WithPort(443),
    WithTLS(),
    WithTimeout(60*time.Second),
)
```

Used in Go's own standard library: `http.Server` options, `grpc.NewServer`, `zap.NewLogger`.

---

## 12. Builder vs Functional Options vs Struct Literal

| | Struct Literal | Functional Options | Builder Pattern |
|--|--|--|--|
| **Boilerplate** | None | Low | High |
| **Validation** | None (at creation) | In `New()` | In `Build()` |
| **Defaults** | Explicit | Encoded in `New()` | Encoded in builder |
| **Chaining** | No | No | Yes |
| **Immutability** | Depends | Depends | Build() returns copy |
| **Multiple variants** | Manual | Via option combos | Via Director |
| **Go idiom** | ✓ (simple structs) | ✓✓ (recommended) | ✓ (complex cases) |
| **When to use** | ≤3 fields, obvious | Optional fields, libraries | Complex + ordered steps + validation |

```go
// Struct literal — fine for simple, stable objects
cfg := ServerConfig{Host: "localhost", Port: 8080}

// Functional options — Go standard for library APIs
cfg := NewServerConfig(WithHost("localhost"), WithPort(8080))

// Builder — when steps have order, validation, or multiple presets
cfg, err := NewServerConfigBuilder().
    SetHost("localhost").EnableTLS().SetPort(443).
    Build()
```

---

## 13. Pattern Structure Summary

```
Client
  │
  ├── creates ──► ConcreteBuilder
  │                    │  sets fields step by step
  │                    │  returns product via Build()/getX()
  │
  └── OR uses ──► Director
                       │  calls builder steps in order
                       │  encodes reusable construction recipes
                       └──► ConcreteBuilder ──► Product
```

---

## 14. When to Use Builder

| Situation | Use Builder? |
|-----------|-------------|
| Object has many optional fields | Yes |
| Construction requires a specific **order** of steps | Yes |
| Different representations via same steps | Yes |
| Need validation before returning the object | Yes |
| Object should never be partially visible | Yes |
| Building complex objects (HTTP requests, SQL queries) | Yes |
| Simple struct with ≤3 fields | No — struct literal |
| Optional config for a library API | No — functional options |
| Only one fixed configuration | No — hard-code it |

---

## 15. Quick Reference

```go
// 1. Product
type Pizza struct {
    size   string
    crust  string
    sauce  string
    cheese bool
}

// 2. Builder (fluent)
type PizzaBuilder struct{ pizza Pizza }

func NewPizzaBuilder() *PizzaBuilder { return &PizzaBuilder{} }

func (b *PizzaBuilder) Size(s string)   *PizzaBuilder { b.pizza.size = s;   return b }
func (b *PizzaBuilder) Crust(c string)  *PizzaBuilder { b.pizza.crust = c;  return b }
func (b *PizzaBuilder) Sauce(s string)  *PizzaBuilder { b.pizza.sauce = s;  return b }
func (b *PizzaBuilder) WithCheese()     *PizzaBuilder { b.pizza.cheese = true; return b }

func (b *PizzaBuilder) Build() (Pizza, error) {
    if b.pizza.size == "" {
        return Pizza{}, errors.New("size is required")
    }
    return b.pizza, nil
}

// 3. Director (optional preset)
func MargheritaRecipe(b *PizzaBuilder) (Pizza, error) {
    return b.Size("medium").Crust("thin").Sauce("tomato").WithCheese().Build()
}

// 4. Client
pizza, err := MargheritaRecipe(NewPizzaBuilder())
// or directly:
pizza, err := NewPizzaBuilder().Size("large").Crust("thick").Sauce("bbq").Build()
```

---

## Further Resources

- [Refactoring.guru — Builder in Go](https://refactoring.guru/design-patterns/builder/go/example)
- [Refactoring.guru — All Go Design Patterns](https://refactoring.guru/design-patterns/go)
- [Dave Cheney — Functional Options for Friendly APIs](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)
- [Squirrel — Fluent SQL Builder for Go](https://github.com/Masterminds/squirrel)
- [DEV — Understanding the Builder Pattern in Go](https://dev.to/kittipat1413/understanding-the-builder-pattern-in-go-gp9)
