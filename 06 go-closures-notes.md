# Closures | In 210 Seconds — Study Notes

**Source:** [Closures | In 210 Seconds](https://www.youtube.com/watch?v=jHd0FczIjAE&list=PL7g1jYj15RUMMCMDYPyZHN3CaWbt3Rl5y&index=2)

> A compact 3.5-minute video on Go closures. These notes expand every concept shown with full examples, patterns, and gotchas.

---

## Timeline Overview

| Timestamp | Topic |
|-----------|-------|
| 0:00 | Anonymous functions |
| 0:40 | What a closure is — capturing variables |
| 1:20 | Stateful closures — the counter example |
| 2:00 | Independent state per closure instance |
| 2:30 | Real-world use: function factories |
| 3:00 | The loop variable gotcha |

---

## 1. Anonymous Functions (0:00)

A closure starts with an **anonymous function** — a function with no name, assigned to a variable or passed directly.

```go
package main

import "fmt"

func main() {
    // Assigned to a variable
    greet := func(name string) string {
        return "Hello, " + name
    }
    fmt.Println(greet("Go")) // Hello, Go

    // Passed directly as an argument
    apply := func(f func(int) int, v int) int {
        return f(v)
    }
    result := apply(func(x int) int { return x * x }, 5)
    fmt.Println(result) // 25
}
```

### IIFE — Immediately Invoked Function Expression

An anonymous function called the moment it's defined:

```go
sum := func(a, b int) int {
    return a + b
}(3, 7) // called immediately with args 3 and 7

fmt.Println(sum) // 10
```

Useful for: one-off computations, initialising complex variables, scoping temp variables.

---

## 2. What a Closure Is — Capturing Variables (0:40)

> **A closure is an anonymous function that references (captures) variables from the scope where it was defined.**

The function "closes over" the outer variable — it keeps a **reference** to it, not a copy.

```go
package main

import "fmt"

func main() {
    message := "hello"

    // This function closes over 'message'
    printMessage := func() {
        fmt.Println(message) // references outer 'message'
    }

    printMessage() // hello

    message = "world" // mutate the outer variable
    printMessage()    // world  ← closure sees the updated value
}
```

**Key insight:** the closure holds a *reference* to `message`, not its value at the time of creation. When `message` changes, the closure sees the new value.

### Closure modifying the outer variable

```go
func main() {
    n := 0

    increment := func() {
        n++ // modifies the OUTER n
    }

    increment()
    increment()
    fmt.Println(n) // 2 — closure modified the outer variable
}
```

---

## 3. Stateful Closures — The Counter (1:20)

When a closure is returned from a function, the captured variable **outlives** the function that created it. The variable is moved to the heap and lives as long as the closure does.

```go
package main

import "fmt"

func makeCounter() func() int {
    n := 0                // lives on the heap — outlives makeCounter()
    return func() int {
        n++
        return n
    }
}

func main() {
    counter := makeCounter()

    fmt.Println(counter()) // 1
    fmt.Println(counter()) // 2
    fmt.Println(counter()) // 3
}
```

**What happens step by step:**
1. `makeCounter()` is called — `n` is created on the heap (escape analysis promotes it)
2. An anonymous function that references `n` is returned
3. `makeCounter()` returns — but `n` is NOT garbage collected (the closure holds a reference)
4. Each call to `counter()` increments and returns the same `n`

---

## 4. Independent State Per Closure Instance (2:00)

Each call to the factory function creates a **brand-new, independent** captured variable.

```go
package main

import "fmt"

func makeCounter() func() int {
    n := 0
    return func() int {
        n++
        return n
    }
}

func main() {
    a := makeCounter() // a gets its own n
    b := makeCounter() // b gets its own n — completely separate

    fmt.Println(a()) // 1
    fmt.Println(a()) // 2
    fmt.Println(a()) // 3

    fmt.Println(b()) // 1 — b starts fresh
    fmt.Println(b()) // 2

    fmt.Println(a()) // 4 — a continues from where it left off
}
```

### Fibonacci generator — multiple captured variables

```go
func makeFib() func() int {
    a, b := 0, 1
    return func() int {
        a, b = b, a+b
        return a
    }
}

func main() {
    fib := makeFib()
    for i := 0; i < 10; i++ {
        fmt.Print(fib(), " ") // 1 1 2 3 5 8 13 21 34 55
    }
}
```

---

## 5. Real-World Use Cases (2:30)

### Use Case 1: Function Factory (Configurable Behaviour)

```go
func adder(base int) func(int) int {
    return func(x int) int {
        return base + x
    }
}

func multiplier(factor int) func(int) int {
    offset := 2
    return func(x int) int {
        return x*factor + offset
    }
}

func main() {
    add5  := adder(5)
    add10 := adder(10)
    triple := multiplier(3)

    fmt.Println(add5(3))   // 8
    fmt.Println(add10(3))  // 13
    fmt.Println(triple(4)) // 14  (4*3 + 2)
}
```

### Use Case 2: Middleware / Function Wrapping

The most common production use — wrapping an HTTP handler with logging, timing, auth:

```go
package main

import (
    "fmt"
    "net/http"
    "time"
)

// timed wraps any handler and logs how long it takes
func timed(f func(http.ResponseWriter, *http.Request)) func(http.ResponseWriter, *http.Request) {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        f(w, r)                                               // call the original handler
        fmt.Printf("%s %s took %v\n", r.Method, r.URL.Path, time.Since(start))
    }
}

// Injecting dependencies (e.g. DB) without globals
func hello(db *Database) func(http.ResponseWriter, *http.Request) {
    return func(w http.ResponseWriter, r *http.Request) {
        // db is captured — no global variable needed
        user, _ := db.GetUser(r.URL.Query().Get("id"))
        fmt.Fprintln(w, user.Name)
    }
}

func main() {
    db := NewDatabase("localhost:5432")
    http.HandleFunc("/hello", timed(hello(db)))
    http.ListenAndServe(":3000", nil)
}
```

### Use Case 3: Memoization (Caching)

```go
func memoize(f func(int) int) func(int) int {
    cache := map[int]int{} // captured — persists across calls
    return func(x int) int {
        if val, ok := cache[x]; ok {
            fmt.Printf("cache hit for %d\n", x)
            return val
        }
        result := f(x)
        cache[x] = result
        return result
    }
}

func slowSquare(n int) int {
    time.Sleep(100 * time.Millisecond) // simulate expensive work
    return n * n
}

func main() {
    cachedSquare := memoize(slowSquare)
    fmt.Println(cachedSquare(5)) // slow — computes
    fmt.Println(cachedSquare(5)) // instant — cache hit
    fmt.Println(cachedSquare(6)) // slow — computes
}
```

### Use Case 4: sort.Search / Callbacks

```go
import "sort"

numbers := []int{-5, 0, 1, 2, 8, 11, 12}
// sort.Search needs a closure to know what it's searching for
index := sort.Search(len(numbers), func(i int) bool {
    return numbers[i] >= 7 // closure captures 'numbers' from outer scope
})
fmt.Println("First number >= 7:", numbers[index]) // 8
```

### Use Case 5: Isolating State (no global variables)

```go
// BEFORE — global variable, accessible everywhere, hard to test
var requestCount int
func countRequests() {
    requestCount++
}

// AFTER — state hidden inside closure, nobody else can touch it
func makeRequestCounter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}
count := makeRequestCounter()
fmt.Println(count()) // 1
fmt.Println(count()) // 2
```

---

## 6. The Loop Variable Gotcha (3:00)

The most common closure bug in Go.

### The Problem

```go
package main

import "fmt"

func main() {
    funcs := make([]func(), 3)

    for i := 0; i < 3; i++ {
        funcs[i] = func() {
            fmt.Println(i) // captures REFERENCE to i, not the value
        }
    }

    for _, f := range funcs {
        f()
    }
}
// Output: 3
//         3
//         3   ← all print the FINAL value of i
```

**Why:** `i` is a single variable. All three closures point to the same `i`. By the time they run, the loop has finished and `i == 3`.

### Fix 1: Shadow the variable (simplest)

```go
for i := 0; i < 3; i++ {
    i := i  // creates a NEW variable per iteration — shadows outer i
    funcs[i] = func() {
        fmt.Println(i)
    }
}
// Output: 0, 1, 2
```

### Fix 2: Pass as parameter to IIFE

```go
for i := 0; i < 3; i++ {
    funcs[i] = func(val int) func() {
        return func() { fmt.Println(val) }
    }(i) // i is passed BY VALUE here — captured as a copy
}
// Output: 0, 1, 2
```

### Fix 3: Helper function

```go
func makeFunc(val int) func() {
    return func() { fmt.Println(val) }
}

for i := 0; i < 3; i++ {
    funcs[i] = makeFunc(i) // i passed by value to makeFunc
}
// Output: 0, 1, 2
```

> **Note (Go 1.22+):** Loop variables are now per-iteration by default. Fix 1 is no longer needed in Go 1.22+ but is still good practice for clarity and backward compat.

### The same bug with `go` and `defer`

```go
// BUG: goroutine captures reference to i
for i := 0; i < 3; i++ {
    go func() {
        fmt.Println(i) // likely prints 3 three times
    }()
}

// FIX: pass i as argument
for i := 0; i < 3; i++ {
    go func(n int) {
        fmt.Println(n) // prints 0, 1, 2 (in any order)
    }(i)
}
```

```go
// BUG: defer captures reference — runs in LIFO order with final value
for i := 0; i < 3; i++ {
    defer fmt.Println(i) // prints 2, 1, 0 — correct values because
}                        // fmt.Println args are evaluated at defer time

// But with closures inside defer, it's different:
for i := 0; i < 3; i++ {
    defer func() {
        fmt.Println(i) // BUG: prints 3, 3, 3
    }()
}

// FIX:
for i := 0; i < 3; i++ {
    defer func(n int) {
        fmt.Println(n) // prints 2, 1, 0
    }(i)
}
```

---

## 7. The `defer setup()()` Trick

When a function returns a teardown function, invoke it with `()()`:

```go
func setup() func() {
    fmt.Println("setting up...")
    return func() {
        fmt.Println("tearing down...")
    }
}

func main() {
    // WRONG — only calls setup(), defers nothing meaningful
    defer setup()

    // RIGHT — calls setup() immediately, defers the returned func()
    defer setup()()
}
// Output:
// setting up...
// tearing down...
```

Common pattern in tests:

```go
func TestSomething(t *testing.T) {
    defer startServer()() // starts server now, defers shutdown
    // ... test code ...
}
```

---

## 8. Closures in Structs (Functional Objects)

Embed closures in structs to create stateful objects without defining methods:

```go
type Counter struct {
    Increment func()
    Value     func() int
    Reset     func()
}

func NewCounter() Counter {
    n := 0
    return Counter{
        Increment: func() { n++ },
        Value:     func() int { return n },
        Reset:     func() { n = 0 },
    }
}

func main() {
    c := NewCounter()
    c.Increment()
    c.Increment()
    c.Increment()
    fmt.Println(c.Value()) // 3
    c.Reset()
    fmt.Println(c.Value()) // 0
}
```

All three functions share the **same** `n` — they all close over the same variable.

---

## 9. Under the Hood — Heap Escape (Performance Note)

When a closure outlives the function that created it, the captured variable **escapes to the heap**.

```go
func makeCounter() func() int {
    n := 0  // Go detects this escapes — allocates on heap, not stack
    return func() int {
        n++
        return n
    }
}
```

**Check with escape analysis:**
```bash
go build -gcflags="-m" ./...
# Look for: "moved to heap: n"
```

**Performance implications:**
- Heap allocation is slightly slower than stack
- Captured variables stay alive as long as any closure references them
- Long-lived closures holding large objects = memory leak potential

**When it matters:**
- In tight loops allocating many closures → consider reusing or using structs
- For most code, the overhead is negligible

---

## Summary — Mental Model

```
Closure = anonymous function + captured variables

                 ┌─────────────────────────────────┐
                 │  outer scope                     │
                 │                                  │
                 │  x := 10  ◄──────────────────┐  │
                 │                               │  │
                 │  f := func() {                │  │
                 │      fmt.Println(x) ──────────┘  │  (reference, not copy)
                 │  }                               │
                 └─────────────────────────────────┘
```

| Concept | Key point |
|---------|-----------|
| Captures by **reference** | Closure sees mutations; mutations are visible outside too |
| State persists | Captured vars outlive the enclosing function (heap allocation) |
| Each instance is independent | New call to factory = new captured variable |
| Loop gotcha | All closures in a loop share ONE variable — shadow or pass as arg |
| `defer f()()` | Calls `f()` now, defers the returned function |

---

## Quick Reference

```go
// Basic closure
f := func() { fmt.Println("hello") }
f()

// IIFE
result := func(x int) int { return x * x }(5)

// Factory returning closure
func makeAdder(n int) func(int) int {
    return func(x int) int { return x + n }
}
add5 := makeAdder(5)
add5(3) // 8

// Middleware pattern
func logged(f func()) func() {
    return func() {
        fmt.Println("before")
        f()
        fmt.Println("after")
    }
}

// Loop fix (pre Go 1.22)
for i := 0; i < n; i++ {
    i := i                  // shadow
    go func() { use(i) }()
}

// defer setup teardown
defer setup()()
```

---

## Further Resources

- [Go by Example: Closures](https://gobyexample.com/closures)
- [What is a Closure? — calhoun.io](https://www.calhoun.io/what-is-a-closure/)
- [5 Useful Ways to Use Closures — calhoun.io](https://www.calhoun.io/5-useful-ways-to-use-closures-in-go/)
- [Closure Gotchas — calhoun.io](https://www.calhoun.io/gotchas-and-common-mistakes-with-closures-in-go/)
- [How to Use Closures in Go — freeCodeCamp](https://www.freecodecamp.org/news/how-to-use-closures-in-go/)
