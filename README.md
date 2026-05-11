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
| 17 | [Introduction to REST APIs](17%20go-rest-api-introduction-notes.md) | Backend / Web APIs | [Watch](https://www.youtube.com/watch?v=Rj-kJeJqU1c) |
| 18 | [All HTTP Status Codes Explained](18%20go-http-status-codes-notes.md) | Backend / HTTP | [Watch](https://www.youtube.com/watch?v=Sqhmv8ucjUo) |
| 19 | [Echo Framework — Part 1](19%20go-echo-framework-part1-notes.md) | Web Framework | [Watch](https://www.youtube.com/watch?v=2dg0qMMC8F4) |
| 20 | [Echo Framework — Part 2](20%20go-echo-framework-part2-notes.md) | Web Framework | [Watch](https://www.youtube.com/watch?v=dS7__u7GHxM) |
| 21 | [JWT — JSON Web Tokens](21%20go-jwt-notes.md) | Auth / Security | [Watch](https://www.youtube.com/watch?v=c-tJRpAOJhk) |

---

## YouTube Playlists

| Playlist | Description |
|----------|-------------|
| [Concurrency Patterns](https://www.youtube.com/playlist?list=PL7g1jYj15RUNqJStuwE9SCmeOKpgxC0HP) | Goroutines, channels, sync, context — 5-part series |
| [Golang in X Seconds](https://www.youtube.com/playlist?list=PL7g1jYj15RUMMCMDYPyZHN3CaWbt3Rl5y) | Quick bite-sized Go concept videos |
| [Design Patterns (Visualized)](https://www.youtube.com/playlist?list=PL7g1jYj15RUO-crQOgDV0_dVp2OaoSgye) | Creational patterns in Go |
| [SOLID Principles in Go](https://www.youtube.com/playlist?list=PL7g1jYj15RUP6aXNe66M91sB-ek9AC3w_) | SOLID design principles with real-world Go examples |
| [30 Days Golang Backend MasterClass](https://www.youtube.com/playlist?list=PLxBfKGlYrg-Hudnm0odvZZDn-J6BEmr1y) | Beginner to production — REST APIs, backend engineering |
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

### Backend / Web APIs & HTTP (Notes 17–18)
- REST principles: stateless, client-server, uniform interface
- HTTP methods (GET, POST, PUT, PATCH, DELETE) → CRUD mapping
- HTTP status codes (2xx / 4xx / 5xx)
- URL and resource design best practices
- JSON encoding/decoding with struct tags
- Building REST APIs with `net/http` vs Gin
- Routing — path params, query params, route groups
- Request binding and input validation (`binding` tags)
- Testing endpoints with curl
- Echo framework — `echo.New()`, handler signature, routing, groups
- Path params (`c.Param`), query params (`c.QueryParam`), binding (`c.Bind`)
- Response helpers — `c.JSON`, `c.String`, `c.XML`, `c.NoContent`, `c.Redirect`
- Built-in middleware — Logger, Recover, CORS, RateLimiter, Gzip, Secure
- Custom middleware pattern, `c.Set/Get` for passing data between middleware/handler
- `echo.ErrNotFound`, `echo.NewHTTPError`, custom global error handler
- Validation with `go-playground/validator` — struct tags, field-level errors
- Cookies — `c.SetCookie`, `c.Cookie`, delete by setting `MaxAge=-1`
- Request/response headers — `c.Request().Header.Get`, `c.Response().Header().Set`
- File upload — single (`c.FormFile`) and multiple (`c.MultipartForm`)
- Static files — `e.Static("/prefix", "dir")`
- JWT middleware — `echojwt.WithConfig`, claims extraction from context
- Graceful shutdown — goroutine + OS signal + `e.Shutdown(ctx)` with timeout
- Testing Echo handlers with `httptest` — path params, JSON body, error assertions
- HTTP status codes — all 5 families (1xx–5xx) with use cases
- Go `net/http` status constants + `http.StatusText()`
- REST API decision guide (which code to return when)
- 401 vs 403, 400 vs 422, 404 vs 410 distinctions
- Common status code mistakes to avoid

### Auth / Security (Note 21)
- JWT structure — header, payload (claims), signature
- Standard registered claims (`exp`, `iat`, `sub`, `iss`, `aud`, `jti`)
- HS256 (symmetric) vs RS256 (asymmetric) — when to use each
- `golang-jwt/jwt/v5` — generate, sign, parse, validate tokens
- Custom claims struct vs `MapClaims` — type-safe approach
- All JWT error types and error handling patterns
- Access token (15 min) + refresh token (7 days) pattern
- Token revocation — Redis blacklist with `jti`
- JWT middleware for Echo — `echo-jwt` package + custom middleware
- Algorithm confusion attack and how to prevent it
- Security best practices — env vars, HTTPS, `HttpOnly` cookies, RS256

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
17 → Introduction to REST APIs
18 → All HTTP Status Codes
19 → Echo Framework Part 1
20 → Echo Framework Part 2
21 → JWT (JSON Web Tokens)
```
