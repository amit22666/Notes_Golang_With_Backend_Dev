# Go Pointers — Learn golang pointers in 179.001 seconds

**Source:** https://www.youtube.com/watch?v=v-ttLYKqaO8&list=PL7g1jYj15RUMMCMDYPyZHN3CaWbt3Rl5y&index=4  
**Duration:** ~3 minutes  
**Playlist:** Golang in X Seconds

---

## Prerequisites

- Basic Go syntax (variables, functions, structs)
- Mental model of variables as named memory locations

---

## Timeline

| Timestamp | Topic |
|-----------|-------|
| 0:00 | What is a pointer? Memory addresses |
| 0:25 | `&` operator — taking the address |
| 0:45 | `*` operator — dereferencing |
| 1:05 | Declaring pointer variables (`var p *int`, `new()`) |
| 1:25 | Nil pointers |
| 1:45 | Pass-by-value vs pass-by-pointer |
| 2:15 | Struct pointers and auto-dereference |
| 2:40 | Pointer receivers vs value receivers |
| 2:55 | Escape analysis — stack vs heap |

---

## 1. What Is a Pointer?

A pointer is a variable that stores the **memory address** of another variable.

Every variable lives at some address in memory. A pointer lets you hold onto that address and read or write through it.

```go
x := 42
fmt.Println(x)   // 42 — the value
fmt.Println(&x)  // 0xc000018090 — the address (varies)
```

---

## 2. The `&` Operator — Taking an Address

`&variable` returns a pointer to `variable` (its memory address).

```go
x := 10
p := &x          // p is of type *int, holds address of x
fmt.Println(p)   // 0xc000018090
fmt.Println(*p)  // 10 — value at that address
```

The type of `p` is `*int`: "pointer to int".

---

## 3. The `*` Operator — Dereferencing

`*pointer` gives you the value stored at the address the pointer holds.  
It also lets you **write** through the pointer.

```go
x := 10
p := &x

*p = 99          // write through pointer
fmt.Println(x)   // 99 — x was modified
```

Two distinct uses of `*`:
- In a **type**: `*int` means "pointer to int"
- In an **expression**: `*p` means "value at address p"

---

## 4. Declaring Pointer Variables

```go
// Method 1: var declaration (zero value is nil)
var p *int
fmt.Println(p)  // <nil>

// Method 2: address of existing variable
x := 5
p = &x

// Method 3: new() — allocates, returns pointer
p2 := new(int)   // *p2 == 0 (zero value)
*p2 = 7
fmt.Println(*p2) // 7
```

`new(T)` allocates zeroed storage for a value of type T and returns a `*T`.

---

## 5. Nil Pointers

A pointer's zero value is `nil` — it points to nothing.  
Dereferencing nil causes a **runtime panic**.

```go
var p *int
fmt.Println(p == nil) // true

*p = 5 // panic: runtime error: invalid memory address or nil pointer dereference
```

Always guard before dereferencing when a pointer may be nil:

```go
func safePrint(p *int) {
    if p == nil {
        fmt.Println("nothing here")
        return
    }
    fmt.Println(*p)
}
```

---

## 6. Pass-by-Value vs Pass-by-Pointer

Go is **always pass-by-value** — function arguments are copied.  
To let a function mutate the caller's variable, pass a pointer.

```go
// Pass by value — caller's variable unchanged
func resetVal(n int) {
    n = 0
}

// Pass by pointer — caller's variable is modified
func resetPtr(n *int) {
    *n = 0
}

func main() {
    x := 42
    resetVal(x)
    fmt.Println(x) // 42 — unchanged

    resetPtr(&x)
    fmt.Println(x) // 0 — modified
}
```

When to pass a pointer:
- You need to modify the caller's data
- The struct is large (avoid copying)
- You need to distinguish "zero value" from "not set" (`nil` check)

---

## 7. Struct Pointers — Auto-Dereference

Go automatically dereferences pointer-to-struct for field access.  
No need for `(*p).Field`; `p.Field` works.

```go
type Person struct {
    Name string
    Age  int
}

func birthday(p *Person) {
    p.Age++ // shorthand for (*p).Age++
}

func main() {
    bob := Person{Name: "Bob", Age: 30}
    birthday(&bob)
    fmt.Println(bob.Age) // 31
}
```

Struct literal with `&` creates a pointer directly:

```go
bob := &Person{Name: "Bob", Age: 30} // bob is *Person
bob.Age = 31  // auto-deref still works
```

---

## 8. Pointer Receivers vs Value Receivers

Methods can have either a value receiver or a pointer receiver.

```go
type Rectangle struct {
    Width, Height float64
}

// Value receiver — operates on a copy, cannot mutate
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

// Pointer receiver — can mutate, shares the original
func (r *Rectangle) Scale(factor float64) {
    r.Width *= factor
    r.Height *= factor
}

func main() {
    rect := Rectangle{Width: 10, Height: 5}
    fmt.Println(rect.Area())  // 50

    rect.Scale(2)             // Go auto-takes address: (&rect).Scale(2)
    fmt.Println(rect.Width)   // 20
    fmt.Println(rect.Height)  // 10
}
```

**Rule of thumb:**
- Use **pointer receivers** when the method mutates state or when the struct is large.
- Use **value receivers** for read-only, small structs where copies are cheap.
- Be consistent: if any method on a type has a pointer receiver, prefer pointer receivers for all methods on that type.

---

## 9. Loop Variable Pointer Pitfall

Classic pre-Go 1.22 bug: loop variable is reused each iteration.

```go
// Bug — all closures capture the same &i
funcs := make([]func(), 3)
for i := 0; i < 3; i++ {
    funcs[i] = func() { fmt.Println(&i, i) }
}
for _, f := range funcs {
    f() // prints 3, 3, 3 (all see final i)
}

// Fix 1: shadow the variable
for i := 0; i < 3; i++ {
    i := i // new variable per iteration
    funcs[i] = func() { fmt.Println(i) }
}

// Fix 2: pass as argument
for i := 0; i < 3; i++ {
    funcs[i] = func(val int) func() {
        return func() { fmt.Println(val) }
    }(i)
}
```

Go 1.22+ fixes this: each iteration gets its own loop variable.

---

## 10. Escape Analysis — Stack vs Heap

Go's compiler decides where to allocate memory: **stack** (fast, auto-freed) or **heap** (garbage-collected).

```go
// Value semantics — stays on stack (no escape)
func createUserV1() user {
    u := user{name: "Alice", age: 30}
    return u  // copy returned, u freed on return
}

// Pointer semantics — escapes to heap
func createUserV2() *user {
    u := user{name: "Alice", age: 30}
    return &u  // address returned — u must survive → heap
}
```

Inspect escape analysis:

```bash
go build -gcflags "-m -m" ./...
# Output: ./main.go:12:9: &u escapes to heap
```

Key insight: returning a pointer to a local variable is **safe** in Go — the compiler automatically moves it to the heap. Unlike C, there are no dangling pointer bugs.

Reasons a value escapes to heap:
- Pointer returned from a function (as above)
- Pointer stored in an interface (`var i interface{} = &u`)
- Pointer passed to a goroutine
- Value too large for the stack

---

## 11. Quick Reference

```go
// Operators
&x      // address of x → *T
*p      // value at address p → T
*p = v  // write through pointer

// Declaration
var p *int          // nil pointer
p = &x              // point to x
p := new(int)       // allocate zeroed int, return *int

// Nil guard
if p != nil { fmt.Println(*p) }

// Struct auto-deref
type S struct { V int }
s := &S{V: 1}
s.V = 2   // same as (*s).V = 2

// Value vs pointer receiver
func (s S)  ReadOnly() {}    // copy
func (s *S) Mutate()  {}    // shared original

// Escape analysis
go build -gcflags "-m" .
```

---

## 12. When to Use Pointers

| Situation | Use Pointer? |
|-----------|-------------|
| Need to mutate caller's variable | Yes |
| Large struct (avoid copy cost) | Yes |
| Implementing an interface that requires it | Yes |
| Need nil to mean "not set" | Yes |
| Small read-only data (int, bool, small struct) | No |
| Returning multiple values via out-params | Prefer multiple returns |
| Slices and maps (already reference types) | Usually no |

---

## 13. Slices and Maps Are Already References

Slices and maps are internally reference types — passing them to a function already shares the underlying data.

```go
func appendX(s []int) {
    s = append(s, 99) // doesn't affect caller (header copied)
}

func modify(s []int) {
    s[0] = 99 // affects caller (same backing array)
}

func addKey(m map[string]int) {
    m["x"] = 1 // affects caller (map is a reference)
}
```

You only need `*[]int` if you want to modify the slice header itself (length, capacity, or replace the slice entirely with a new one from `append`).

---

## 14. unsafe.Pointer (Advanced)

`unsafe.Pointer` bypasses Go's type system — use only in very low-level code.

```go
import "unsafe"

x := int64(42)
// Reinterpret memory as a different type
p := (*float64)(unsafe.Pointer(&x))
fmt.Println(*p) // nonsense bits, but compiles
```

Legitimate uses: interop with C via `cgo`, implementing `sync.atomic` operations on arbitrary types.  
**Avoid in normal application code** — the garbage collector and type safety cannot protect you.

---

## Further Resources

- [Go Tour — Pointers](https://go.dev/tour/moretypes/1)
- [Ardan Labs — Language Mechanics on Stacks and Pointers](https://www.ardanlabs.com/blog/2017/05/language-mechanics-on-stacks-and-pointers.html)
- [Effective Go — Pointers vs Values](https://go.dev/doc/effective_go#pointers_vs_values)
- [Go FAQ — Should I define methods on values or pointers?](https://go.dev/doc/faq#methods_on_values_or_pointers)
- [go build escape analysis flags](https://pkg.go.dev/cmd/compile)
