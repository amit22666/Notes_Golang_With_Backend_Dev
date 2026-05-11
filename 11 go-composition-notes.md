# Go Composition — Master Golang with Composition

**Source:** https://www.youtube.com/watch?v=kgCYq3EGoyE&list=PL7g1jYj15RUMMCMDYPyZHN3CaWbt3Rl5y&index=6  
**Duration:** ~3 minutes  
**Playlist:** Golang in X Seconds

---

## Prerequisites

- Go structs and methods
- Interfaces and implicit implementation
- Pointer vs value receivers

---

## Timeline

| Timestamp | Topic |
|-----------|-------|
| 0:00 | Why Go has no inheritance |
| 0:20 | Struct embedding syntax |
| 0:35 | Promoted fields |
| 0:50 | Promoted methods |
| 1:10 | Critical difference from inheritance (receiver stays embedded type) |
| 1:25 | Method shadowing and explicit access |
| 1:45 | Multiple embedding and ambiguity |
| 2:05 | Interface satisfaction via embedding |
| 2:20 | Embedding interfaces in structs (decorator pattern) |
| 2:45 | Practical patterns: sync.Mutex, sort.Reverse, context |

---

## 1. Why Composition, Not Inheritance?

Go deliberately has **no inheritance**. Instead it has **composition** — building types out of other types.

Inheritance creates tight coupling through "is-a" hierarchies that are hard to change.  
Composition uses "has-a" relationships that stay flexible.

> "Favour object composition over class inheritance." — Gang of Four (1994)

Go enforces this at the language level: there is no `extends`, no `super`, no class hierarchy.  
The mechanism is **struct embedding**.

---

## 2. Basic Struct Embedding

Embed a type by declaring it **without a field name** inside another struct:

```go
type Animal struct {
    Name string
}

type Dog struct {
    Animal        // embedded — no field name
    Breed string
}

d := Dog{
    Animal: Animal{Name: "Rex"}, // must use type name in literal
    Breed:  "Labrador",
}

fmt.Println(d.Name)  // Rex   — promoted field (shorthand)
fmt.Println(d.Animal.Name) // Rex — explicit access also works
fmt.Println(d.Breed) // Labrador
```

The embedded type appears with its **type name as its implicit field name**.

---

## 3. Promoted Fields

Fields of the embedded struct are **promoted** to the outer struct — accessible directly.

```go
type Address struct {
    Street string
    City   string
    Zip    string
}

type Person struct {
    Name    string
    Age     int
    Address // embedded
}

p := Person{
    Name: "Alice",
    Age:  30,
    Address: Address{
        Street: "123 Main St",
        City:   "Springfield",
        Zip:    "12345",
    },
}

fmt.Println(p.City)   // Springfield (promoted)
fmt.Println(p.Zip)    // 12345 (promoted)
fmt.Println(p.Address.City) // also valid (explicit)
```

---

## 4. Promoted Methods

Methods on the embedded type are also **promoted** — callable directly on the outer struct.

```go
type Animal struct {
    Name string
}

func (a Animal) Speak() string {
    return a.Name + " makes a sound"
}

func (a Animal) String() string {
    return "Animal: " + a.Name
}

type Dog struct {
    Animal
    Breed string
}

d := Dog{Animal: Animal{Name: "Rex"}, Breed: "Lab"}
fmt.Println(d.Speak())  // Rex makes a sound
fmt.Println(d.String()) // Animal: Rex
```

`Dog` now has both `Speak()` and `String()` without writing a single forwarding method.

---

## 5. Critical Difference from Inheritance

**The receiver is always the embedded type, not the outer type.**

This is the most important distinction from class-based inheritance.

```go
type Base struct{ val int }

func (b Base) Who() string {
    return fmt.Sprintf("I am Base, val=%d", b.val)
}

type Derived struct {
    Base
}

d := Derived{Base: Base{val: 42}}
fmt.Println(d.Who()) // "I am Base, val=42"
// Receiver is Base, NOT Derived — no "virtual dispatch"
```

In Java/Python, `d.Who()` would receive `Derived` as `this`/`self`.  
In Go, `d.Who()` receives a `Base` value — the embedding struct is invisible to the method.

**Implication:** Embedding is **delegation**, not inheritance. The promoted method runs as if called on the embedded field directly.

---

## 6. Method Shadowing (Override)

If the outer struct defines a method with the same name, it **shadows** the embedded one.

```go
type Animal struct{ Name string }

func (a Animal) Speak() string { return a.Name + " says: ..." }

type Dog struct {
    Animal
    Breed string
}

// Dog defines its own Speak — shadows Animal.Speak
func (d Dog) Speak() string { return d.Name + " says: Woof!" }

d := Dog{Animal: Animal{Name: "Rex"}, Breed: "Lab"}
fmt.Println(d.Speak())        // Rex says: Woof! (Dog's version)
fmt.Println(d.Animal.Speak()) // Rex says: ... (explicit access to shadowed)
```

Shadowing is **not virtual dispatch** — there is no dynamic polymorphism at the struct level (interfaces provide that).

---

## 7. Multiple Embedding

A struct can embed multiple types, inheriting all their promoted fields and methods:

```go
type Walker struct{}
func (w Walker) Walk() { fmt.Println("walking") }

type Swimmer struct{}
func (s Swimmer) Swim() { fmt.Println("swimming") }

type Triathlete struct {
    Walker
    Swimmer
    Name string
}

t := Triathlete{Name: "Alice"}
t.Walk() // promoted from Walker
t.Swim() // promoted from Swimmer
```

**Ambiguity:** If two embedded types have the same field or method name, accessing it directly is a compile error. You must qualify explicitly:

```go
type A struct{ val int }
type B struct{ val int }

type C struct {
    A
    B
}

c := C{}
// c.val  // compile error: ambiguous selector c.val
c.A.val = 1 // explicit — OK
c.B.val = 2 // explicit — OK
```

If `C` itself defines `val`, it shadows both — no ambiguity.

---

## 8. Interface Satisfaction via Embedding

If an embedded type implements an interface, the **outer struct automatically satisfies that interface** too.

```go
type Stringer interface {
    String() string
}

type Base struct{ ID int }

func (b Base) String() string { return fmt.Sprintf("Base#%d", b.ID) }

type Widget struct {
    Base
    Label string
}

var s Stringer = Widget{Base: Base{ID: 7}, Label: "btn"}
fmt.Println(s.String()) // Base#7
```

`Widget` never explicitly declared it implements `Stringer` — embedding did that for free.

---

## 9. Embedding Interfaces in Structs (Decorator Pattern)

You can embed an **interface** inside a struct. This is one of Go's most powerful patterns.

**Use case:** Wrap an existing implementation, overriding only the methods you care about.

```go
// Without embedding: must write all 8 net.Conn forwarding methods manually
// With embedding: only write the one we want to intercept

type StatsConn struct {
    net.Conn       // embed the interface
    BytesRead uint64
}

// Only override Read — all other net.Conn methods forward automatically
func (sc *StatsConn) Read(p []byte) (int, error) {
    n, err := sc.Conn.Read(p)
    sc.BytesRead += uint64(n)
    return n, err
}

// Usage: wrap any net.Conn
conn, _ := net.Dial("tcp", "example.com:80")
stats := &StatsConn{Conn: conn}
// stats still satisfies net.Conn — use it anywhere net.Conn is accepted
```

This is the **Decorator** pattern — add behavior without modifying the original.

---

## 10. sort.Reverse — Interface Wrapping in the Standard Library

`sort.Reverse` is a canonical example. It embeds `sort.Interface` and inverts `Less`:

```go
type reverse struct {
    sort.Interface // embed the interface
}

// Only override the one method that needs changing
func (r reverse) Less(i, j int) bool {
    return r.Interface.Less(j, i) // swap arguments to invert order
}

// sort.Reverse returns the wrapper
func Reverse(data Interface) Interface {
    return &reverse{data}
}
```

```go
nums := []int{3, 1, 4, 1, 5, 9}
sort.Sort(sort.Reverse(sort.IntSlice(nums)))
fmt.Println(nums) // [9 5 4 3 1 1]
```

`Len()` and `Swap()` forward to the embedded `sort.Interface` automatically.  
Only `Less()` is intercepted. Three lines of code, infinite power.

---

## 11. context.valueCtx — Embedding for Delegation Chains

The `context` package uses embedding to build context trees:

```go
type valueCtx struct {
    Context            // embed parent context
    key, val interface{}
}

func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        return c.val  // found at this level
    }
    return c.Context.Value(key) // delegate up the chain
}
```

`Deadline()`, `Done()`, `Err()` all forward to the parent automatically.  
Only `Value()` is intercepted to check the local key.

---

## 12. Embedding sync.Mutex

Embed `sync.Mutex` to expose locking as part of a type's API:

```go
// Mutex as EXPORTED API — embed it
type SafeMap struct {
    sync.Mutex
    m map[string]int
}

func (sm *SafeMap) Set(key string, val int) {
    sm.Lock()
    defer sm.Unlock()
    sm.m[key] = val
}

// Callers can also lock externally if they need compound operations
sm := &SafeMap{m: make(map[string]int)}
sm.Lock()
// ... multi-step operation ...
sm.Unlock()
```

If the mutex should be **internal only** (not part of the public API), use a named field instead:

```go
type SafeMap struct {
    mu sync.Mutex // unexported named field — not promoted
    m  map[string]int
}
```

---

## 13. Embedding Pointer vs Value

You can embed either the value type or a pointer to it:

```go
// Value embedding — outer struct contains a copy
type Dog struct {
    Animal        // copy of Animal
    Breed string
}

// Pointer embedding — outer struct holds a pointer
type Dog struct {
    *Animal       // pointer to Animal
    Breed string
}
```

**Value embedding:**
- `Dog` always has its own `Animal` — no nil risk
- Copying a `Dog` copies the embedded `Animal`

**Pointer embedding:**
- Multiple structs can share the same `Animal` instance
- Must initialize the pointer before use (nil dereference risk)
- Methods with pointer receivers on `Animal` are promoted as-is

```go
a := &Animal{Name: "Rex"}
d1 := Dog{Animal: a, Breed: "Lab"}
d2 := Dog{Animal: a, Breed: "Poodle"}
a.Name = "Max"
fmt.Println(d1.Name) // Max — both share the same Animal
fmt.Println(d2.Name) // Max
```

---

## 14. Real-World Composition Example

Building a layered HTTP middleware using struct embedding:

```go
type Logger struct{}

func (l Logger) Log(msg string) {
    fmt.Printf("[LOG] %s\n", msg)
}

type Metrics struct{}

func (m Metrics) Record(event string) {
    fmt.Printf("[METRIC] %s\n", event)
}

type UserHandler struct {
    Logger          // embed logger
    Metrics         // embed metrics
    userRepo UserRepository
}

func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    h.Log("GetUser called")         // promoted from Logger
    h.Record("user.get")            // promoted from Metrics
    // ... handle request ...
}
```

---

## 15. Partial Interface Implementation (Test Doubles)

Embedding an interface in a struct lets you implement **only the methods you need** — useful for test mocks of large interfaces:

```go
// Large interface with many methods
type Storage interface {
    Get(key string) ([]byte, error)
    Set(key string, val []byte) error
    Delete(key string) error
    List(prefix string) ([]string, error)
    // ... 10 more methods
}

// Test only needs Get — implement just that
type fakeStorage struct {
    Storage  // embed interface — all other methods "exist" (but panic if called)
    data map[string][]byte
}

func (f *fakeStorage) Get(key string) ([]byte, error) {
    v, ok := f.data[key]
    if !ok {
        return nil, errors.New("not found")
    }
    return v, nil
}

// Use in test — only Get is called, so embedded nil methods never fire
func TestMyFeature(t *testing.T) {
    store := &fakeStorage{data: map[string][]byte{"x": []byte("hello")}}
    // pass store wherever Storage is needed
}
```

> Warning: if any unimplemented method gets called at runtime, you'll get a nil pointer panic. Only use this pattern when you're certain which methods are exercised.

---

## 16. Composition vs Embedding: When to Use Each

| Approach | When to use |
|----------|-------------|
| **Named field** (`dog Dog`) | Relationship is clear "has-a"; caller should know |
| **Embedding** (unnamed `Dog`) | Want to promote methods/fields; feels like "is-a" but still composition |
| **Interface embedding in struct** | Decorator/wrapper — override a subset of a large interface |
| **Interface composition** | Build large contracts from small ones (`io.ReadWriter`) |

---

## 17. Quick Reference

```go
// Struct embedding
type Outer struct {
    Inner        // promoted fields + methods
    ExtraField string
}

// Initialization
o := Outer{Inner: Inner{...}, ExtraField: "x"}

// Access
o.InnerField    // promoted (shorthand)
o.Inner.InnerField  // explicit

// Method shadowing
func (o Outer) Method() {} // shadows Inner.Method()
o.Inner.Method()           // call shadowed version explicitly

// Multiple embedding
type Multi struct { A; B }
// m.A.SharedField vs m.B.SharedField — must qualify if ambiguous

// Embed interface in struct (decorator)
type Wrapper struct {
    OriginalInterface
}
func (w *Wrapper) SpecificMethod() {} // override one method

// Embed value vs pointer
type WithValue   struct { MyType        }
type WithPointer struct { *MyType       }

// Interface composition
type ReadWriter interface { Reader; Writer }
```

---

## 18. Common Gotchas

| Gotcha | Explanation |
|--------|-------------|
| Embedded receiver doesn't see outer type | `Inner.Method()` gets an `Inner` receiver, not `Outer` |
| Nil embedded interface panics | `Wrapper{}.Method()` panics if the embedded interface is nil |
| Ambiguous selector on multiple embedding | Qualify with type name: `c.A.Field` not `c.Field` |
| Copying embeds a value | Embedding by value copies data; changes to original don't propagate |
| Exported embedding exposes internals | Embedding `sync.Mutex` makes `Lock`/`Unlock` part of the public API |

---

## Further Resources

- [Effective Go — Embedding](https://go.dev/doc/effective_go#embedding)
- [Eli Bendersky — Embedding in Go: Part 1 (structs in structs)](https://eli.thegreenplace.net/2020/embedding-in-go-part-1-structs-in-structs/)
- [Eli Bendersky — Embedding in Go: Part 3 (interfaces in structs)](https://eli.thegreenplace.net/2020/embedding-in-go-part-3-interfaces-in-structs/)
- [Go by Example: Struct Embedding](https://gobyexample.com/struct-embedding)
- [Go 101 — Type Embedding](https://go101.org/article/type-embedding.html)
