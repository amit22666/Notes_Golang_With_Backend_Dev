# Go Programming — Generics in 2.8 Minutes | Study Notes

**Source:** [Go Programming - Generics in 2.8 Minutes](https://www.youtube.com/watch?v=yb4-RPBqVcs&list=PL7g1jYj15RUMMCMDYPyZHN3CaWbt3Rl5y&index=2)

> A short-form (~2.8 min) video on Go generics (introduced in Go 1.18). These notes expand every concept shown into a full reference with patterns, gotchas, and real-world examples.

---

## Timeline Overview

| Timestamp | Topic |
|-----------|-------|
| 0:00 | The problem — code duplication without generics |
| 0:30 | Type parameter syntax `[T any]` |
| 1:00 | Built-in constraints: `any`, `comparable` |
| 1:30 | Union type constraints `int \| float64` |
| 2:00 | Tilde operator `~` — underlying types |
| 2:20 | Type inference — omitting type arguments |
| 2:40 | Named constraints as interfaces |

---

## 1. The Problem — Code Duplication (0:00)

Before generics, the only way to write a function that worked on multiple types was to either duplicate it or use `interface{}` and lose type safety.

```go
// Without generics — must write one per type
func SumInts(m map[string]int64) int64 {
    var s int64
    for _, v := range m { s += v }
    return s
}

func SumFloats(m map[string]float64) float64 {
    var s float64
    for _, v := range m { s += v }
    return s
}

// OR use interface{} — no type safety, runtime panics possible
func Sum(vals []interface{}) interface{} {
    // what operations can we do on interface{}? None safely.
    return nil
}
```

**Generics solve this**: write the logic once, let the compiler specialise it per type.

---

## 2. Type Parameter Syntax (0:30)

Type parameters are declared in **square brackets** immediately after the function name.

```
func Name[TypeParam Constraint](regularArgs) returnType
      ↑    ↑         ↑
  func    type       constraint on what
  name    param      types are allowed
  name    name
```

### Simplest example

```go
// T can be ANY type
func Print[T any](v T) {
    fmt.Println(v)
}

Print("hello")       // T = string (inferred)
Print(42)            // T = int   (inferred)
Print([]int{1,2,3})  // T = []int (inferred)
Print[bool](true)    // T = bool  (explicit)
```

### Generic function — one function replaces two

```go
// Before: two functions
func SumInts(m map[string]int64) int64    { ... }
func SumFloats(m map[string]float64) float64 { ... }

// After: one generic function
func Sum[K comparable, V int64 | float64](m map[K]V) V {
    var s V
    for _, v := range m {
        s += v
    }
    return s
}

ints   := map[string]int64{"a": 1, "b": 2}
floats := map[string]float64{"x": 1.1, "y": 2.2}

fmt.Println(Sum(ints))   // 3
fmt.Println(Sum(floats)) // 3.3
```

### Multiple type parameters

```go
func Map[K comparable, V any](m map[K]V, f func(V) V) map[K]V {
    result := make(map[K]V, len(m))
    for k, v := range m {
        result[k] = f(v)
    }
    return result
}
```

---

## 3. Built-in Constraints: `any` and `comparable` (1:00)

A **constraint** restricts which concrete types may be used as a type argument. It is written as an interface.

### `any` — no restriction

```go
// 'any' is an alias for interface{} — accepts every type
func Identity[T any](v T) T {
    return v
}
```

Use `any` when you only need to store or pass the value — you can't call methods or use operators like `+`, `<` on it.

### `comparable` — supports `==` and `!=`

```go
// Required for map keys, and for using == / !=
func Contains[T comparable](slice []T, target T) bool {
    for _, v := range slice {
        if v == target { // only works because T is comparable
            return true
        }
    }
    return false
}

fmt.Println(Contains([]int{1, 2, 3}, 2))         // true
fmt.Println(Contains([]string{"a", "b"}, "c"))   // false
```

**Types that satisfy `comparable`:** all basic types (int, string, bool, float…), pointers, arrays, structs with comparable fields.
**Types that don't:** slices, maps, functions.

---

## 4. Union Type Constraints `T1 | T2` (1:30)

Constrain a type parameter to a **specific set** of types using `|`.

```go
// V can only be int64 OR float64
func Sum[V int64 | float64](s []V) V {
    var total V
    for _, v := range s {
        total += v // + operator works because both types support it
    }
    return total
}

fmt.Println(Sum([]int64{1, 2, 3}))       // 6
fmt.Println(Sum([]float64{1.1, 2.2}))    // 3.3
// Sum([]string{"a"}) → compile error: string doesn't satisfy int64|float64
```

### Min / Max with Ordered types

```go
// Works for any type that supports < operator
func Min[T int | int8 | int16 | int32 | int64 | float32 | float64 | string](x, y T) T {
    if x < y {
        return x
    }
    return y
}

fmt.Println(Min(3, 7))         // 3
fmt.Println(Min(3.14, 2.71))   // 2.71
fmt.Println(Min("apple", "banana")) // apple
```

---

## 5. The Tilde Operator `~` — Underlying Types (2:00)

Without `~`, a constraint `int` only allows the **exact** built-in `int` type. With `~int`, it allows `int` **and any type whose underlying type is `int`**.

```go
type Celsius    float64
type Fahrenheit float64
type Kelvin     float64

// Without ~: only float64 works — Celsius, Fahrenheit rejected
func AvgFloat64(vals []float64) float64 { ... }

// With ~float64: Celsius, Fahrenheit, Kelvin all work
func Avg[T ~float64](vals []T) T {
    var sum T
    for _, v := range vals {
        sum += v
    }
    return sum / T(len(vals))
}

temps := []Celsius{20.0, 22.5, 19.0}
fmt.Println(Avg(temps)) // 20.5 — works even though Celsius ≠ float64
```

### Named type with `~` in union

```go
type MyString string

// Without ~: MyString rejected
// With ~string: MyString accepted
func Println[T ~string | int](v T) {
    fmt.Println(v)
}

Println("plain string")       // ok
Println(MyString("custom"))   // ok — underlying type is string
Println(42)                   // ok
```

---

## 6. Type Inference — Omitting Type Arguments (2:20)

The compiler can **infer** type arguments from the function's regular arguments. You rarely need to specify them explicitly.

```go
func Reverse[T any](s []T) []T {
    r := make([]T, len(s))
    for i, v := range s {
        r[len(s)-i-1] = v
    }
    return r
}

// Explicit (always works)
Reverse[int]([]int{1, 2, 3})

// Inferred (cleaner — compiler sees []int, deduces T = int)
Reverse([]int{1, 2, 3})
Reverse([]string{"a", "b", "c"})
```

### When inference fails — you must be explicit

Type inference only works when the type parameter appears in the function's parameter list. If it's only in the return type or body, you must provide it explicitly.

```go
func Zero[T any]() T {
    var z T
    return z
}

// Zero() // ERROR: cannot infer T — no function argument to infer from
Zero[int]()    // must be explicit
Zero[string]()
```

---

## 7. Named Constraints as Interfaces (2:40)

Instead of repeating `int | int8 | int32 | int64 | float32 | float64` everywhere, declare a **reusable constraint interface**.

```go
// Constraint interface — used ONLY as a type parameter constraint
type Number interface {
    int | int8 | int16 | int32 | int64 |
    uint | uint8 | uint16 | uint32 | uint64 |
    float32 | float64
}

type Ordered interface {
    Integer | Float | ~string
}

type Integer interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64
}

type Float interface {
    ~float32 | ~float64
}

// Now reuse cleanly
func Sum[T Number](vals []T) T {
    var s T
    for _, v := range vals { s += v }
    return s
}

func Min[T Ordered](x, y T) T {
    if x < y { return x }
    return y
}

func Max[T Ordered](x, y T) T {
    if x > y { return x }
    return y
}
```

### `golang.org/x/exp/constraints` — ready-made constraints

```go
import "golang.org/x/exp/constraints"

// constraints.Ordered = Integer | Float | ~string
func GMin[T constraints.Ordered](x, y T) T {
    if x < y { return x }
    return y
}

// constraints.Integer, constraints.Float, constraints.Complex
// constraints.Signed, constraints.Unsigned
```

> **Note:** In Go 1.21+ the `cmp` package provides `cmp.Ordered` in the standard library, replacing the need for `golang.org/x/exp/constraints.Ordered`.

```go
import "cmp"

func Min[T cmp.Ordered](x, y T) T {
    if x < y { return x }
    return y
}
```

---

## 8. Generic Types — Structs and Methods

Type parameters work on types (structs), not just functions.

### Generic Stack

```go
package main

import "fmt"

type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(v T) {
    s.items = append(s.items, v)
}

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    top := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return top, true
}

func (s *Stack[T]) Peek() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    return s.items[len(s.items)-1], true
}

func (s *Stack[T]) Size() int { return len(s.items) }

func main() {
    var ints Stack[int]
    ints.Push(1)
    ints.Push(2)
    ints.Push(3)
    v, _ := ints.Pop()
    fmt.Println(v) // 3

    var words Stack[string]
    words.Push("hello")
    words.Push("world")
    top, _ := words.Peek()
    fmt.Println(top) // world
}
```

### Generic Key-Value Pair

```go
type Pair[K, V any] struct {
    Key   K
    Value V
}

func NewPair[K, V any](k K, v V) Pair[K, V] {
    return Pair[K, V]{Key: k, Value: v}
}

p := NewPair("age", 30)
fmt.Println(p.Key, p.Value) // age 30
```

### Generic Map with `comparable` key

```go
type Cache[K comparable, V any] struct {
    data map[K]V
}

func NewCache[K comparable, V any]() *Cache[K, V] {
    return &Cache[K, V]{data: make(map[K]V)}
}

func (c *Cache[K, V]) Set(k K, v V) { c.data[k] = v }

func (c *Cache[K, V]) Get(k K) (V, bool) {
    v, ok := c.data[k]
    return v, ok
}

cache := NewCache[string, int]()
cache.Set("hits", 42)
v, ok := cache.Get("hits")
fmt.Println(v, ok) // 42 true
```

> **Limitation:** You cannot add type parameters to individual methods — only to the whole type. Type parameters must be declared at the type level.

```go
// ILLEGAL — cannot add new type params on a method
func (s *Stack[T]) Map[U any](f func(T) U) []U { ... }

// LEGAL — use a package-level function instead
func StackMap[T, U any](s *Stack[T], f func(T) U) []U { ... }
```

---

## 9. Generic Utility Functions

### Filter, Map, Reduce

```go
// Filter — keep elements satisfying predicate
func Filter[T any](s []T, predicate func(T) bool) []T {
    result := make([]T, 0, len(s))
    for _, v := range s {
        if predicate(v) {
            result = append(result, v)
        }
    }
    return result
}

// Map — transform every element
func Map[T, U any](s []T, f func(T) U) []U {
    result := make([]U, len(s))
    for i, v := range s {
        result[i] = f(v)
    }
    return result
}

// Reduce — accumulate to a single value
func Reduce[T, U any](s []T, init U, f func(U, T) U) U {
    acc := init
    for _, v := range s {
        acc = f(acc, v)
    }
    return acc
}

func main() {
    nums := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

    evens  := Filter(nums, func(n int) bool { return n%2 == 0 })
    // [2 4 6 8 10]

    doubled := Map(nums, func(n int) int { return n * 2 })
    // [2 4 6 8 10 12 14 16 18 20]

    strs := Map(nums, func(n int) string { return fmt.Sprintf("%d", n) })
    // ["1" "2" ... "10"]

    sum := Reduce(nums, 0, func(acc, n int) int { return acc + n })
    // 55
}
```

### Keys / Values from a map

```go
func Keys[K comparable, V any](m map[K]V) []K {
    keys := make([]K, 0, len(m))
    for k := range m {
        keys = append(keys, k)
    }
    return keys
}

func Values[K comparable, V any](m map[K]V) []V {
    vals := make([]V, 0, len(m))
    for _, v := range m {
        vals = append(vals, v)
    }
    return vals
}
```

### Standard library generics (Go 1.21+)

```go
import (
    "slices"
    "maps"
    "cmp"
)

s := []int{3, 1, 4, 1, 5, 9}

slices.Sort(s)                             // in-place sort
slices.Contains(s, 4)                     // true
slices.Index(s, 9)                        // index of 9
slices.Reverse(s)                         // in-place reverse
min := slices.Min(s)                      // minimum element
max := slices.Max(s)                      // maximum element
slices.SortFunc(s, cmp.Compare)           // custom comparator

m := map[string]int{"a": 1, "b": 2}
keys   := slices.Collect(maps.Keys(m))    // Go 1.23
values := slices.Collect(maps.Values(m))  // Go 1.23
maps.DeleteFunc(m, func(k string, v int) bool { return v < 2 })
```

---

## 10. Memoization with Generics

```go
func Memoize[K comparable, V any](f func(K) V) func(K) V {
    cache := map[K]V{}
    return func(k K) V {
        if v, ok := cache[k]; ok {
            return v
        }
        v := f(k)
        cache[k] = v
        return v
    }
}

var fib func(int) int
fib = Memoize(func(n int) int {
    if n <= 1 { return n }
    return fib(n-1) + fib(n-2)
})

fmt.Println(fib(40)) // fast — results cached
```

---

## 11. When to Use Generics (and When Not To)

### Use generics when:

```
✓ Writing the SAME logic repeated for different types
✓ Container/collection types: Stack, Queue, Set, Tree
✓ Utility functions: Map, Filter, Reduce, Contains, Keys
✓ Algorithms on slices/maps: Sort, Search, Min, Max
✓ You want compile-time type safety instead of interface{}
```

### Don't use generics when:

```
✗ Behaviour differs per type — use interfaces instead
✗ Only one or two types will ever be used — just write both
✗ Adding complexity for no real reuse benefit
✗ The function already works cleanly with interface{}
✗ You need to reflect on the type at runtime
```

### Generics vs Interfaces — the key distinction

```go
// Interface: runtime dispatch — any type implementing the interface
type Stringer interface {
    String() string
}
func Print(s Stringer) { fmt.Println(s.String()) }

// Generic: compile-time specialisation — specific types listed/constrained
func Print[T fmt.Stringer](s T) { fmt.Println(s.String()) }

// Rule of thumb:
// - Use interfaces when behaviour varies per type (polymorphism)
// - Use generics when the ALGORITHM is the same across types (code reuse)
```

---

## 12. Common Mistakes

### 1. Using operations not allowed by constraint

```go
func Double[T any](v T) T {
    return v + v  // COMPILE ERROR: operator + not defined for T any
}

// Fix: constrain to types that support +
func Double[T int | float64 | string](v T) T {
    return v + v  // ok
}
```

### 2. Forgetting `comparable` for map keys or `==`

```go
func Find[T any](s []T, target T) int {
    for i, v := range s {
        if v == target { ... } // COMPILE ERROR: cannot use == on T any
    }
}

// Fix:
func Find[T comparable](s []T, target T) int { ... }
```

### 3. Trying to add type params to methods

```go
type MyType[T any] struct{ val T }

// ILLEGAL
func (m MyType[T]) Convert[U any]() U { ... }

// Fix: package-level function
func Convert[T, U any](m MyType[T], f func(T) U) U {
    return f(m.val)
}
```

### 4. Forgetting `~` for custom types

```go
type Meter float64

type Length interface{ float64 }  // Meter rejected!
type Length interface{ ~float64 } // Meter accepted

func Add[T Length](a, b T) T { return a + b }

Add(Meter(1.5), Meter(2.5)) // only works with ~float64
```

### 5. Instantiating with wrong type

```go
func Sum[T int | float64](vals []T) T { ... }

Sum([]string{"a", "b"}) // COMPILE ERROR: string does not satisfy int|float64
```

---

## Quick Reference

```go
// Generic function
func Name[T Constraint](arg T) T { ... }

// Multiple type params
func Name[K comparable, V any](m map[K]V) []V { ... }

// Inline union constraint
func Name[T int | float64 | string](v T) { ... }

// Tilde — underlying types
func Name[T ~int | ~string](v T) { ... }

// Named constraint interface
type Number interface {
    ~int | ~int32 | ~int64 | ~float32 | ~float64
}
func Name[T Number](v T) T { ... }

// Generic type (struct)
type Box[T any] struct{ val T }
func (b Box[T]) Get() T { return b.val }

// Instantiation
box := Box[string]{val: "hello"}

// Explicit type argument
Name[int](42)

// Inferred type argument (usually preferred)
Name(42)

// Built-in constraints
any         // interface{} — any type
comparable  // supports == and !=

// Standard library (Go 1.21+)
import "cmp"    // cmp.Ordered
import "slices" // slices.Sort, Contains, Index, Min, Max, Reverse…
import "maps"   // maps.Keys, Values, DeleteFunc…
```

---

## Constraint Cheat Sheet

| Constraint | What it allows | Use for |
|-----------|---------------|---------|
| `any` | Every type | Store/pass values |
| `comparable` | Types with `==`, `!=` | Map keys, equality checks |
| `int \| float64` | Exactly these types | Arithmetic operations |
| `~int \| ~float64` | These + any type with same underlying | Custom numeric types |
| `cmp.Ordered` | All ordered types (int, float, string + aliases) | `<`, `>`, comparisons |
| `constraints.Integer` | All integer types | Integer arithmetic |
| `constraints.Float` | All float types | Float arithmetic |

---

## Further Resources

- [Tutorial: Getting started with generics — go.dev](https://go.dev/doc/tutorial/generics)
- [An Introduction to Generics — go.dev blog](https://go.dev/blog/intro-generics)
- [Generic Interfaces — go.dev blog](https://go.dev/blog/generic-interfaces)
- [Writing generic collection types — DoltHub](https://www.dolthub.com/blog/2024-07-01-golang-generic-collections/)
- [Understanding generics in Go 1.18 — LogRocket](https://blog.logrocket.com/understanding-generics-go-1-18/)
- [cmp package — pkg.go.dev](https://pkg.go.dev/cmp)
- [slices package — pkg.go.dev](https://pkg.go.dev/slices)
