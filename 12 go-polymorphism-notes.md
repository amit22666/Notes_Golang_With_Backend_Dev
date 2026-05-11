# Go Polymorphism — Master Golang with Polymorphism

**Source:** https://www.youtube.com/watch?v=cPhS10BkChw&list=PL7g1jYj15RUMMCMDYPyZHN3CaWbt3Rl5y&index=7  
**Duration:** ~3 minutes  
**Playlist:** Golang in X Seconds

---

## Prerequisites

- Go interfaces and implicit implementation
- Structs and methods
- Generics basics (for section 7)

---

## Timeline

| Timestamp | Topic |
|-----------|-------|
| 0:00 | What is polymorphism? Three forms |
| 0:20 | Subtype polymorphism via interfaces |
| 0:40 | Dynamic dispatch — one function, many types |
| 1:00 | Slice of interfaces — uniform treatment |
| 1:20 | Open/Closed Principle connection |
| 1:40 | Type switch — inspecting concrete type |
| 2:00 | Function types as lightweight polymorphism |
| 2:20 | Parametric polymorphism — generics |
| 2:45 | Interface dispatch internals (iface/itab) |

---

## 1. What Is Polymorphism?

Polymorphism means **"one interface, many forms"** — the same code works with different types.

Go supports three kinds:

| Kind | Mechanism | Resolved at |
|------|-----------|-------------|
| **Subtype** | Interfaces | Runtime |
| **Parametric** | Generics (`[T Constraint]`) | Compile time |
| **Ad-hoc** | Function types / method sets | Varies |

Go has **no inheritance**, so classical OOP polymorphism doesn't exist. Everything goes through interfaces or generics.

---

## 2. Subtype Polymorphism via Interfaces

The primary form. Different concrete types implement the same interface; code written against the interface works for all of them.

```go
type Shape interface {
    Area() float64
    Perimeter() float64
}

type Rectangle struct{ Width, Height float64 }

func (r Rectangle) Area() float64      { return r.Width * r.Height }
func (r Rectangle) Perimeter() float64 { return 2 * (r.Width + r.Height) }

type Circle struct{ Radius float64 }

func (c Circle) Area() float64      { return math.Pi * c.Radius * c.Radius }
func (c Circle) Perimeter() float64 { return 2 * math.Pi * c.Radius }

type Triangle struct{ Base, Height, A, B float64 }

func (t Triangle) Area() float64      { return 0.5 * t.Base * t.Height }
func (t Triangle) Perimeter() float64 { return t.Base + t.A + t.B }
```

```go
// One function — works with every Shape, present and future
func printShapeInfo(s Shape) {
    fmt.Printf("Area: %.2f  Perimeter: %.2f\n", s.Area(), s.Perimeter())
}

printShapeInfo(Rectangle{Width: 4, Height: 5})  // Area: 20.00  Perimeter: 18.00
printShapeInfo(Circle{Radius: 3})               // Area: 28.27  Perimeter: 18.85
printShapeInfo(Triangle{Base: 6, Height: 4, A: 5, B: 5}) // Area: 12.00 ...
```

`printShapeInfo` never imports `Rectangle`, `Circle`, or `Triangle`. It only sees `Shape`.

---

## 3. Dynamic Dispatch — Slice of Interfaces

The real power: treat a collection of different types uniformly.

```go
shapes := []Shape{
    Rectangle{Width: 4, Height: 5},
    Circle{Radius: 3},
    Triangle{Base: 6, Height: 4, A: 5, B: 5},
}

totalArea := 0.0
for _, s := range shapes {
    totalArea += s.Area() // correct method called for each concrete type
}
fmt.Printf("Total area: %.2f\n", totalArea) // 60.27
```

At each iteration, Go dispatches to the method of the **actual** concrete type — `Rectangle.Area`, `Circle.Area`, or `Triangle.Area`.

---

## 4. Real-World Example: Income Calculator

Adding new income types requires **zero changes** to the calculation function — the Open/Closed Principle in action.

```go
type Income interface {
    Calculate() int
    Source() string
}

type FixedBilling struct {
    Project string
    Amount  int
}
func (f FixedBilling) Calculate() int    { return f.Amount }
func (f FixedBilling) Source() string    { return f.Project }

type TimeAndMaterial struct {
    Project  string
    Hours    int
    RatePerHr int
}
func (t TimeAndMaterial) Calculate() int  { return t.Hours * t.RatePerHr }
func (t TimeAndMaterial) Source() string  { return t.Project }

// Added later — calculateNetIncome needs NO modification
type Advertisement struct {
    Name      string
    CPC       int
    Clicks    int
}
func (a Advertisement) Calculate() int  { return a.CPC * a.Clicks }
func (a Advertisement) Source() string  { return a.Name }

func calculateNetIncome(streams []Income) int {
    total := 0
    for _, s := range streams {
        earned := s.Calculate()
        fmt.Printf("%-20s $%d\n", s.Source(), earned)
        total += earned
    }
    return total
}

func main() {
    streams := []Income{
        FixedBilling{"Project Alpha", 5000},
        TimeAndMaterial{"Project Beta", 160, 25},
        Advertisement{"Banner Ad", 2, 500},
        Advertisement{"Popup Ad", 5, 750},
    }
    fmt.Printf("Net income: $%d\n", calculateNetIncome(streams))
}
// Output:
// Project Alpha        $5000
// Project Beta         $4000
// Banner Ad            $1000
// Popup Ad             $3750
// Net income: $13750
```

---

## 5. Payment Processor Example

Classic use case: swap implementations without touching business logic.

```go
type PaymentProcessor interface {
    Charge(amount float64) error
    Refund(amount float64) error
}

type StripeProcessor struct{ APIKey string }
func (s *StripeProcessor) Charge(amount float64) error {
    fmt.Printf("Stripe: charging $%.2f\n", amount)
    return nil
}
func (s *StripeProcessor) Refund(amount float64) error {
    fmt.Printf("Stripe: refunding $%.2f\n", amount)
    return nil
}

type PayPalProcessor struct{ ClientID string }
func (p *PayPalProcessor) Charge(amount float64) error {
    fmt.Printf("PayPal: charging $%.2f\n", amount)
    return nil
}
func (p *PayPalProcessor) Refund(amount float64) error {
    fmt.Printf("PayPal: refunding $%.2f\n", amount)
    return nil
}

type OrderService struct {
    processor PaymentProcessor
}

func (o *OrderService) PlaceOrder(amount float64) error {
    return o.processor.Charge(amount) // works for Stripe, PayPal, or any future processor
}

// Swap at wire-up time — OrderService code never changes
svc := &OrderService{processor: &StripeProcessor{APIKey: "sk_live_..."}}
svc.PlaceOrder(99.99) // Stripe: charging $99.99

svc.processor = &PayPalProcessor{ClientID: "..."}
svc.PlaceOrder(99.99) // PayPal: charging $99.99
```

---

## 6. Type Switch — Recovering Concrete Type

Sometimes you need to know the concrete type inside an interface — use a type switch.

```go
type Animal interface{ Sound() string }

type Dog struct{ Name string }
func (d Dog) Sound() string { return "Woof" }

type Cat struct{ Name string }
func (c Cat) Sound() string { return "Meow" }

type Bird struct{ Name string }
func (b Bird) Sound() string { return "Tweet" }

func describe(a Animal) {
    switch v := a.(type) {
    case Dog:
        fmt.Printf("Dog named %s says %s\n", v.Name, v.Sound())
    case Cat:
        fmt.Printf("Cat named %s says %s\n", v.Name, v.Sound())
    case Bird:
        fmt.Printf("Bird named %s says %s\n", v.Name, v.Sound())
    default:
        fmt.Printf("Unknown animal: %T\n", v)
    }
}

animals := []Animal{Dog{"Rex"}, Cat{"Whiskers"}, Bird{"Tweety"}}
for _, a := range animals {
    describe(a)
}
// Dog named Rex says Woof
// Cat named Whiskers says Meow
// Bird named Tweety says Tweet
```

Prefer the interface method (`a.Sound()`) when the operation is common to all types.  
Use type switch only when you need type-specific behaviour not expressible in the interface.

---

## 7. Function Types as Lightweight Polymorphism

A function type can replace a single-method interface — simpler and more flexible.

```go
// Instead of a one-method interface...
type Transformer interface {
    Transform(s string) string
}

// ...use a function type directly
type TransformFunc func(string) string

func applyAll(s string, fns ...TransformFunc) string {
    for _, fn := range fns {
        s = fn(s)
    }
    return s
}

upper  := TransformFunc(strings.ToUpper)
trim   := TransformFunc(strings.TrimSpace)
exclaim := TransformFunc(func(s string) string { return s + "!" })

result := applyAll("  hello world  ", trim, upper, exclaim)
fmt.Println(result) // HELLO WORLD!
```

The standard library uses this pattern with `http.HandlerFunc`, `sort.Slice`, `filepath.WalkFunc`, etc.

---

## 8. Parametric Polymorphism — Generics

Generics let you write **one function that works for multiple types**, resolved at compile time (no runtime interface overhead).

```go
// Without generics — need separate functions or use any (loses type safety)
func SumInts(nums []int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

// With generics — one function, statically typed
func Sum[T int | int64 | float64](nums []T) T {
    var total T
    for _, n := range nums {
        total += n
    }
    return total
}

fmt.Println(Sum([]int{1, 2, 3}))         // 6
fmt.Println(Sum([]float64{1.1, 2.2}))   // 3.3
fmt.Println(Sum([]int64{100, 200}))      // 300
```

```go
// Generic container — polymorphic over the element type
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) { s.items = append(s.items, item) }
func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    top := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return top, true
}

intStack := Stack[int]{}
intStack.Push(1)
intStack.Push(2)
v, _ := intStack.Pop()
fmt.Println(v) // 2

strStack := Stack[string]{}
strStack.Push("hello")
```

---

## 9. Combining Embedding and Interfaces (Inherited Polymorphism)

Embedding + interface satisfaction lets a composed type be used wherever any embedded type is accepted.

```go
type Logger struct{}
func (l Logger) Log(msg string) { fmt.Println("[LOG]", msg) }

type AuditLogger struct {
    Logger  // promotes Log
}
func (a AuditLogger) Log(msg string) {
    a.Logger.Log(msg)                  // call embedded
    fmt.Println("[AUDIT] recorded:", msg) // add behaviour
}

type Logged interface{ Log(string) }

func process(l Logged, msg string) {
    l.Log(msg)
}

process(Logger{}, "startup")
// [LOG] startup

process(AuditLogger{}, "payment")
// [LOG] payment
// [AUDIT] recorded: payment
```

---

## 10. Interface Dispatch Internals (iface / itab)

Understanding how Go implements polymorphism at runtime.

An interface value is a **two-word struct**:

```
+----------+----------+
|  *itab   |  *data   |
+----------+----------+
  type info   actual value
```

- `*itab` — points to the **interface table**: the concrete type + a vtable of method function pointers
- `*data` — pointer to the actual value (or the value itself if it fits in a pointer)

```
itab {
    inter  *interfacetype  // which interface
    _type  *_type          // concrete type
    hash   uint32          // for type switches
    fun    [n]uintptr      // method pointers (vtable)
}
```

When you call `s.Area()` on an interface:
1. Load `itab` from the interface value
2. Index into `itab.fun` to get the `Area` function pointer
3. Call that function pointer with `data` as the receiver

The runtime **caches itabs globally** — repeated assignments of the same (interface, type) pair reuse the cached itab. Method pointer lookup is `O(1)` with the function pointer table already in L1 cache.

**Empty interface (`any` / `eface`)** is simpler — no itab (no methods), just `{*_type, *data}`.

---

## 11. Interface Polymorphism vs Generics

| | Interfaces (subtype) | Generics (parametric) |
|--|--|--|
| **Dispatch** | Runtime (dynamic) | Compile time (static) |
| **Overhead** | Two pointer indirections per call | Zero (direct call after monomorphisation) |
| **Flexibility** | Any type added later | Types fixed at call site |
| **Best for** | Collections of mixed types, DI, mocking | Containers, algorithms over homogeneous types |
| **Binary size** | Smaller (one implementation) | Larger (one copy per instantiated type) |

```go
// Use interfaces when types are HETEROGENEOUS (different at runtime)
shapes := []Shape{Circle{}, Rectangle{}, Triangle{}}

// Use generics when types are HOMOGENEOUS (same type, different T)
func Min[T cmp.Ordered](a, b T) T {
    if a < b { return a }
    return b
}
```

---

## 12. Quick Reference

```go
// Define interface (the polymorphic contract)
type Doer interface {
    Do(input string) (string, error)
}

// Implement for multiple types (implicitly)
type FileDoer  struct{ Path string }
type HTTPDoer  struct{ URL  string }

func (f FileDoer) Do(input string) (string, error) { /* ... */ return "", nil }
func (h HTTPDoer) Do(input string) (string, error) { /* ... */ return "", nil }

// Polymorphic function — works for all Doers
func run(d Doer, input string) {
    result, err := d.Do(input)
    if err != nil { log.Fatal(err) }
    fmt.Println(result)
}

// Polymorphic collection
doers := []Doer{FileDoer{"data.txt"}, HTTPDoer{"https://api.example.com"}}
for _, d := range doers {
    run(d, "hello")
}

// Type switch — recover concrete type
switch v := d.(type) {
case FileDoer: fmt.Println("file:", v.Path)
case HTTPDoer: fmt.Println("http:", v.URL)
}

// Generics — parametric polymorphism
func Map[T, U any](s []T, fn func(T) U) []U {
    out := make([]U, len(s))
    for i, v := range s { out[i] = fn(v) }
    return out
}
```

---

## 13. Summary

| Concept | Key Point |
|---------|-----------|
| Polymorphism in Go | No inheritance — achieved via interfaces and generics |
| Subtype polymorphism | Interface as contract; any type implementing methods qualifies |
| Dynamic dispatch | Runtime: itab vtable lookup, `O(1)`, cached globally |
| Slice of interfaces | Treat heterogeneous types uniformly in loops |
| Open/Closed Principle | Add new types without modifying existing functions |
| Type switch | Recover concrete type from interface when needed |
| Function types | Lightweight ad-hoc polymorphism for single-method cases |
| Generics | Compile-time parametric polymorphism for homogeneous collections |
| Interface vs generics | Mixed runtime types → interfaces; same-T algorithms → generics |

---

## Further Resources

- [Go Tour — Interfaces](https://go.dev/tour/methods/9)
- [golangbot.com — Polymorphism in Go](https://golangbot.com/polymorphism/)
- [Go Internals — Chapter II: Interfaces (itab)](https://cmc.gitbook.io/go-internals/chapter-ii-interfaces)
- [Go Blog — An Introduction to Generics](https://go.dev/blog/intro-generics)
- [Dave Cheney — SOLID Go Design](https://dave.cheney.net/2016/08/20/solid-go-design)
