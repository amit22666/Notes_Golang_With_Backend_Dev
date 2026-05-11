# Go Interfaces — Master Golang with Interfaces

**Source:** https://www.youtube.com/watch?v=IbXSEGB8LRs&list=PL7g1jYj15RUMMCMDYPyZHN3CaWbt3Rl5y&index=4  
**Duration:** ~3 minutes  
**Playlist:** Golang in X Seconds

---

## Prerequisites

- Go structs and methods
- Basic understanding of pointer vs value receivers
- Familiarity with Go types

---

## Timeline

| Timestamp | Topic |
|-----------|-------|
| 0:00 | What is an interface? |
| 0:20 | Interface definition syntax |
| 0:40 | Implicit implementation — no `implements` keyword |
| 1:00 | Value vs pointer receivers with interfaces |
| 1:20 | Interface values: (type, value) pair internals |
| 1:40 | Nil interface and the nil pointer gotcha |
| 1:55 | Empty interface / `any` |
| 2:10 | Type assertion and comma-ok idiom |
| 2:30 | Type switch |
| 2:50 | Interface composition / embedding |

---

## 1. What Is an Interface?

An interface defines a set of **method signatures** — a contract for behavior.  
Any type that implements all those methods satisfies the interface. No explicit declaration needed.

```go
type Animal interface {
    Sound() string
    Move()
}
```

Go interfaces are about **what a type can do**, not what it is.

---

## 2. Defining an Interface

```go
type Shape interface {
    Area() float64
    Perimeter() float64
}
```

- All methods are public (uppercase) or package-scoped (lowercase).
- No fields, no implementation — only method signatures.
- By convention, single-method interfaces use the **-er suffix**: `Reader`, `Writer`, `Stringer`, `Closer`.

---

## 3. Implicit Implementation

Unlike Java/C#, Go has **no `implements` keyword**.  
A type satisfies an interface simply by having the required methods.

```go
type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return math.Pi * c.Radius * c.Radius
}

func (c Circle) Perimeter() float64 {
    return 2 * math.Pi * c.Radius
}

// Circle implicitly implements Shape — no declaration needed
var s Shape = Circle{Radius: 5}
fmt.Println(s.Area()) // 78.539...
```

This decouples the implementation from the interface definition — the interface can live in a completely different package.

---

## 4. Value Receiver vs Pointer Receiver

This is the most common interface gotcha.

**Value receiver** — both the value type and pointer type satisfy the interface:

```go
type Greeter interface {
    Greet() string
}

type English struct{}

func (e English) Greet() string { return "Hello!" }

var g Greeter = English{}   // works
var g2 Greeter = &English{} // also works
```

**Pointer receiver** — only the pointer type satisfies the interface:

```go
type Spanish struct{ name string }

func (s *Spanish) Greet() string { return "¡Hola, " + s.name + "!" }

var g Greeter = &Spanish{name: "Ana"} // works
var g2 Greeter = Spanish{name: "Ana"} // compile error:
// Spanish does not implement Greeter (Greet has pointer receiver)
```

**Rule:** If any method uses a pointer receiver, use a pointer when assigning to an interface.

---

## 5. Interface Values: The (type, value) Pair

Under the hood, an interface value holds two things:
- **Dynamic type** — the concrete type stored
- **Dynamic value** — the actual value

```go
var s Shape

// s == nil (both type and value are nil)
fmt.Println(s == nil) // true

s = Circle{Radius: 3}
// s holds: type=Circle, value={Radius:3}
fmt.Println(s.Area()) // 28.274...

s = &Circle{Radius: 3}
// s holds: type=*Circle, value=0xc0000b4000
```

Two interface values are equal if they have the same dynamic type **and** equal dynamic values.

---

## 6. The Nil Interface Gotcha

This is one of Go's most surprising behaviors. An interface holding a **nil pointer** is **not nil**.

```go
type MyError struct{ msg string }
func (e *MyError) Error() string { return e.msg }

func getError(fail bool) error {
    var err *MyError // nil pointer
    if fail {
        err = &MyError{"something failed"}
    }
    return err // BUG: returns (type=*MyError, value=nil), NOT nil interface
}

err := getError(false)
fmt.Println(err == nil) // false! The interface has a type, so it's not nil
```

**Fix:** Return `nil` directly for the interface type:

```go
func getError(fail bool) error {
    if fail {
        return &MyError{"something failed"}
    }
    return nil // returns true nil interface
}
```

---

## 7. Empty Interface — `any`

An interface with no methods is satisfied by **every type**.  
`any` is an alias for `interface{}` (introduced in Go 1.18).

```go
func printAnything(v any) {
    fmt.Println(v)
}

printAnything(42)
printAnything("hello")
printAnything([]int{1, 2, 3})

// Storing mixed types
values := []any{1, "two", 3.0, true}
```

Avoid `any` when you know the type — it loses compile-time type safety.

---

## 8. Type Assertion

Extract the concrete value from an interface.

**Direct assertion** — panics if the type is wrong:

```go
var i any = "hello"
s := i.(string)
fmt.Println(s) // "hello"

n := i.(int) // panic: interface conversion: interface {} is string, not int
```

**Comma-ok form** — safe, never panics:

```go
var i any = "hello"

s, ok := i.(string)
fmt.Println(s, ok) // "hello" true

n, ok := i.(int)
fmt.Println(n, ok) // 0 false
```

Always use the comma-ok form when the type is not guaranteed.

---

## 9. Type Switch

A type switch tests an interface value against multiple types in sequence.

```go
func describe(i any) {
    switch v := i.(type) {
    case nil:
        fmt.Println("nil")
    case int:
        fmt.Printf("int: %d\n", v)
    case string:
        fmt.Printf("string: %q\n", v)
    case bool:
        fmt.Printf("bool: %t\n", v)
    case []int:
        fmt.Printf("[]int with %d elements\n", len(v))
    default:
        fmt.Printf("unknown type: %T\n", v)
    }
}

describe(42)        // int: 42
describe("hello")   // string: "hello"
describe(nil)       // nil
describe(3.14)      // unknown type: float64
```

Multiple types in one case — `v` stays as `any`:

```go
case int, int64:
    fmt.Println("some integer kind") // v is any here
```

---

## 10. Interface Composition (Embedding)

Build larger interfaces from smaller ones:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

// Composed interfaces from the standard library
type ReadWriter interface {
    Reader
    Writer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

A type implementing `ReadWriteCloser` automatically satisfies `Reader`, `Writer`, and `Closer` too.

---

## 11. Common Standard Library Interfaces

### `error`
```go
type error interface {
    Error() string
}

type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error: %s — %s", e.Field, e.Message)
}

// Usage
var err error = &ValidationError{Field: "email", Message: "invalid format"}
fmt.Println(err) // validation error: email — invalid format
```

### `fmt.Stringer`
```go
type Stringer interface {
    String() string
}

type Point struct{ X, Y int }

func (p Point) String() string {
    return fmt.Sprintf("(%d, %d)", p.X, p.Y)
}

p := Point{3, 4}
fmt.Println(p) // (3, 4)  — fmt calls String() automatically
```

### `io.Reader` / `io.Writer`
```go
// io.Reader — anything that can be read from
type Reader interface {
    Read(p []byte) (n int, err error)
}

// io.Writer — anything that can be written to
type Writer interface {
    Write(p []byte) (n int, err error)
}

// All of these implement io.Reader:
// os.File, bytes.Buffer, strings.Reader, http.Response.Body, net.Conn

func countBytes(r io.Reader) (int, error) {
    buf := make([]byte, 512)
    total := 0
    for {
        n, err := r.Read(buf)
        total += n
        if err == io.EOF {
            return total, nil
        }
        if err != nil {
            return total, err
        }
    }
}
```

### `sort.Interface`
```go
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}

type ByAge []Person

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }

people := []Person{{"Alice", 30}, {"Bob", 25}}
sort.Sort(ByAge(people)) // sort by age
```

---

## 12. Interfaces for Polymorphism

```go
type Shape interface {
    Area() float64
    String() string
}

type Circle struct{ Radius float64 }
type Rectangle struct{ Width, Height float64 }

func (c Circle) Area() float64    { return math.Pi * c.Radius * c.Radius }
func (c Circle) String() string   { return fmt.Sprintf("Circle(r=%.2f)", c.Radius) }

func (r Rectangle) Area() float64  { return r.Width * r.Height }
func (r Rectangle) String() string { return fmt.Sprintf("Rect(%gx%g)", r.Width, r.Height) }

func totalArea(shapes []Shape) float64 {
    total := 0.0
    for _, s := range shapes {
        total += s.Area()
    }
    return total
}

shapes := []Shape{
    Circle{Radius: 3},
    Rectangle{Width: 4, Height: 5},
}
fmt.Println(totalArea(shapes)) // 48.274...
```

---

## 13. Interfaces for Testing (Mocking)

```go
type EmailSender interface {
    Send(to, subject, body string) error
}

type UserService struct {
    mailer EmailSender
}

func (us *UserService) Register(email string) error {
    // ... create user ...
    return us.mailer.Send(email, "Welcome!", "Thanks for joining.")
}

// Production
type SMTPMailer struct{}
func (s *SMTPMailer) Send(to, subject, body string) error { /* real SMTP */ return nil }

// Test mock — no external dependency
type MockMailer struct {
    Sent []string
}
func (m *MockMailer) Send(to, subject, body string) error {
    m.Sent = append(m.Sent, to)
    return nil
}
```

---

## 14. Compile-Time Interface Check

Verify a type implements an interface **at compile time** (not runtime):

```go
// Will fail to compile if *MyType doesn't implement io.Writer
var _ io.Writer = (*MyType)(nil)

// Also works with value types
var _ fmt.Stringer = MyType{}
```

Place these near the type definition as a safety net. The blank identifier `_` discards the value; only the assignment (and thus the interface check) matters.

---

## 15. HTTP Handler — Function as Interface

The standard library uses a clever pattern: a function type that implements an interface.

```go
// http.Handler interface
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

// http.HandlerFunc is a function type that implements Handler
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}

// Now a plain function can be used as an http.Handler
func helloHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello!")
}

http.Handle("/hello", http.HandlerFunc(helloHandler))
// or shorter:
http.HandleFunc("/hello", helloHandler)
```

---

## 16. Quick Reference

```go
// Define
type Doer interface {
    Do() error
}

// Implement (implicit)
type MyType struct{}
func (m MyType) Do() error { return nil }

// Assign
var d Doer = MyType{}
var d2 Doer = &MyType{} // pointer receiver → must use pointer

// Compile-time check
var _ Doer = (*MyType)(nil)

// Type assertion
val, ok := d.(MyType)   // safe
val = d.(MyType)        // panics if wrong type

// Type switch
switch v := d.(type) {
case MyType:    fmt.Println("MyType", v)
case *MyType:   fmt.Println("*MyType", v)
default:        fmt.Printf("unknown: %T\n", v)
}

// Empty interface
func f(v any) { fmt.Println(v) }

// Nil check (nil interface vs nil pointer in interface)
var d Doer          // true nil
var mt *MyType      // nil pointer
d = mt              // d != nil! (has type *MyType, value nil)
```

---

## 17. Best Practices

| Practice | Explanation |
|----------|-------------|
| **Keep interfaces small** | 1–3 methods is ideal; easier to implement and compose |
| **Name by behavior** | `-er` suffix: `Reader`, `Stringer`, `Closer` |
| **Accept interfaces, return structs** | Parameters as interfaces = flexible; return concrete types = full API |
| **Define interfaces where used** | Interface belongs in the consumer's package, not the implementer's |
| **Don't pre-emptively interface** | "Don't design with interfaces, discover them" — Rob Pike |
| **Use compile-time check** | `var _ Interface = (*Type)(nil)` near type definition |
| **Avoid nil interface gotcha** | Return `nil` directly, never a typed nil pointer |

---

## 18. Common Mistakes

| Mistake | Fix |
|---------|-----|
| Pointer receiver method, assigning value type to interface | Use `&MyType{}` instead of `MyType{}` |
| Returning a typed nil pointer as an interface | Return `nil` directly from the function |
| Giant interfaces with 10+ methods | Break into smaller focused interfaces |
| Defining interface in the implementer's package | Move interface to the consumer package |
| Using `any` everywhere | Use concrete types or narrow interfaces |
| Panicking type assertion `i.(T)` | Use comma-ok: `v, ok := i.(T)` |

---

## Further Resources

- [Go Tour — Interfaces](https://go.dev/tour/methods/9)
- [Effective Go — Interfaces](https://go.dev/doc/effective_go#interfaces)
- [Go Spec — Interface Types](https://go.dev/ref/spec#Interface_types)
- [Go Blog — The Laws of Reflection](https://go.dev/blog/laws-of-reflection)
- [YourBasic — Type assertions and type switches](https://yourbasic.org/golang/type-assertion-switch/)
