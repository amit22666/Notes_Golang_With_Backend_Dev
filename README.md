# Golang & Backend Dev — Study Notes

A structured collection of Go programming notes covering concurrency, design patterns, OOP concepts, and a full career roadmap — all sourced from curated YouTube videos.

---

## Contents

| # | Topic | Category | YouTube Source |
|---|-------|----------|----------------|
| 00 | [Golang Roadmap That Gets You Hired](00%20go-roadmap-get-hired-notes.md) | Career / Roadmap | [Watch](https://www.youtube.com/watch?v=ZzC3PNeCk6w) |
| 01 | [Concurrency Patterns — Part 1](01%20go-concurrency-patterns-notes.md) | Concurrency | [Watch](https://www.youtube.com/watch?v=qyM8Pi1KiiM) |
| 02 | [Concurrency Patterns — Part 2](02%20go-concurrency-patterns-part2-notes.md) | Concurrency | [Watch](https://www.youtube.com/watch?v=wELNUHb3kuA) |
| 03 | [Concurrency Performance Pattern](03%20go-concurrency-performance-notes.md) | Concurrency | [Watch](https://www.youtube.com/watch?v=Bk1c30avsuU) |
| 04 | [Mind-Blowing Concurrency Pattern](04%20go-concurrency-mind-blowing-pattern-notes.md) | Concurrency | [Watch](https://www.youtube.com/watch?v=bnbEULxcX3o) |
| 05 | [Context Package Explained](05%20go-context-package-notes.md) | Concurrency | [Watch](https://www.youtube.com/watch?v=8omcakb31xQ) |
| 06 | [Closures](06%20go-closures-notes.md) | Language Feature | [Watch](https://www.youtube.com/watch?v=jHd0FczIjAE) |
| 07 | [Generics](07%20go-generics-notes.md) | Language Feature | [Watch](https://www.youtube.com/watch?v=yb4-RPBqVcs) |
| 08 | [Pointers](08%20go-pointers-notes.md) | Language Feature | [Watch](https://www.youtube.com/watch?v=v-ttLYKqaO8) |
| 09 | [Interfaces](09%20go-interfaces-notes.md) | OOP Concepts | [Watch](https://www.youtube.com/watch?v=IbXSEGB8LRs) |
| 10 | [Abstraction](10%20go-abstraction-notes.md) | OOP Concepts | [Watch](https://www.youtube.com/watch?v=CRY4_-p5FgM) |
| 11 | [Composition](11%20go-composition-notes.md) | OOP Concepts | [Watch](https://www.youtube.com/watch?v=kgCYq3EGoyE) |
| 12 | [Polymorphism](12%20go-polymorphism-notes.md) | OOP Concepts | [Watch](https://www.youtube.com/watch?v=cPhS10BkChw) |
| 13 | [Single Responsibility Principle](13%20go-srp-notes.md) | SOLID Principles | [Watch](https://www.youtube.com/watch?v=u7UhIiG2gNk) |
| 14 | [Factory Method Pattern](14%20go-factory-method-pattern-notes.md) | Design Patterns | [Watch](https://www.youtube.com/watch?v=y0HRazQsvUY) |
| 15 | [Abstract Factory Pattern](15%20go-abstract-factory-pattern-notes.md) | Design Patterns | [Watch](https://www.youtube.com/watch?v=F9tQ46YkQLU) |
| 16 | [Builder Pattern](16%20go-builder-pattern-notes.md) | Design Patterns | [Watch](https://www.youtube.com/watch?v=oP76NM4qZhw) |

---

## YouTube Playlists

| Playlist | Description |
|----------|-------------|
| [Concurrency Patterns](https://www.youtube.com/playlist?list=PL7g1jYj15RUNqJStuwE9SCmeOKpgxC0HP) | Goroutines, channels, sync, context — 5-part series |
| [Golang in X Seconds](https://www.youtube.com/playlist?list=PL7g1jYj15RUMMCMDYPyZHN3CaWbt3Rl5y) | Quick bite-sized Go concept videos |
| [Design Patterns (Visualized)](https://www.youtube.com/playlist?list=PL7g1jYj15RUO-crQOgDV0_dVp2OaoSgye) | Creational patterns in Go |
| [SOLID Principles in Go](https://www.youtube.com/playlist?list=PL7g1jYj15RUP6aXNe66M91sB-ek9AC3w_) | SOLID design principles with real-world Go examples |
| [How to Structure a Go Application](https://www.youtube.com/watch?v=MpFog2kZsHk&list=PL7g1jYj15RUPjxpD_PDt8L7IlA-VpT0t8) | Ketan Coding — Go project structure series |

---

## Topics at a Glance

### Concurrency (Notes 01–05)
- Goroutines, channels (buffered / unbuffered)
- WaitGroup, Mutex, select
- Worker pool, pipeline, fan-out / fan-in
- `sync.Pool`, `errgroup`, done-channel pattern
- `context` — cancellation, deadlines, values

### Language Features (Notes 06–08)
- Closures and variable capture
- Generics with type constraints (Go 1.18+)
- Pointers, escape analysis, stack vs heap

### OOP Concepts (Notes 09–12)
- Interfaces (implicit satisfaction)
- Abstraction via interfaces
- Composition over inheritance (struct embedding)
- Polymorphism with interface types

### SOLID Principles (Note 13)
- Single Responsibility Principle with real-world Go example

### Design Patterns (Notes 14–16)
- Factory Method — `NewX()` returning interface
- Abstract Factory — family of related objects
- Builder — step-by-step construction with `Build()`

### Career Roadmap (Note 00)
- Full 11-phase learning path: Fundamentals → Microservices → DevOps
- Portfolio project ideas
- Common interview questions (language, concurrency, internals, system design)
- Recommended learning resources and rough timeline

---

## Suggested Study Order

```
00 → Roadmap overview (start here)
08 → Pointers
09 → Interfaces
10 → Abstraction
11 → Composition
12 → Polymorphism
06 → Closures
07 → Generics
01 → Concurrency Part 1
02 → Concurrency Part 2
03 → Concurrency Performance
04 → Mind-Blowing Pattern
05 → Context Package
13 → SRP
14 → Factory Method
15 → Abstract Factory
16 → Builder
```
