# DAY-8 | Echo Framework | PART-1 — Study Notes

**Source:** https://www.youtube.com/watch?v=2dg0qMMC8F4&list=PLxBfKGlYrg-Hudnm0odvZZDn-J6BEmr1y&index=10  
**Playlist:** 30 Days Golang Backend MasterClass — From Beginner to Production  
**Category:** Backend / Web Framework

---

## Prerequisites

- Go fundamentals (structs, interfaces, error handling)
- REST API basics (HTTP methods, status codes — Day 7 & 8 notes)
- Basic understanding of `net/http`

---

## Timeline

| Timestamp | Topic |
|-----------|-------|
| 0:00 | What is Echo? Why not Gin / net/http? |
| 4:00 | Installation & project setup |
| 7:00 | First Echo server — Hello World |
| 10:00 | `echo.New()` & `e.Start()` |
| 13:00 | Handler signature — `func(echo.Context) error` |
| 16:00 | Routing — GET, POST, PUT, PATCH, DELETE |
| 21:00 | Path parameters — `c.Param()` |
| 25:00 | Query parameters — `c.QueryParam()` |
| 29:00 | Request binding — `c.Bind()` |
| 34:00 | Response methods — JSON, String, XML, NoContent |
| 40:00 | Built-in middleware — Logger, Recover, CORS |
| 46:00 | Custom middleware |
| 50:00 | Route groups |
| 54:00 | Error handling — `HTTPError` |

---

## 1. What Is Echo?

**Echo** is a high-performance, minimalist, extensible Go web framework built on top of `net/http`.

- Created by **Vishal Rana** (labstack), released 2015
- Latest stable: **Echo v4** (v5 in development as of 2026)
- Import path: `github.com/labstack/echo/v4`

### Key Design Principles

| Principle | What it means |
|-----------|--------------|
| **Minimalist** | No magic, no bloat — only what you need |
| **Performant** | Trie-based router, zero-allocation in hot paths |
| **Extensible** | Plug in your own binder, validator, renderer |
| **Error-centric** | Handlers return `error` — clean, composable, idiomatic Go |

### Echo vs Gin vs net/http

| Feature | `net/http` | Gin | Echo |
|---------|-----------|-----|------|
| Path parameters | Manual | `c.Param()` | `c.Param()` |
| JSON binding | Manual decode | `c.ShouldBindJSON()` | `c.Bind()` |
| JSON response | Manual encode | `c.JSON()` | `c.JSON()` |
| Error handling | Manual | `c.AbortWithError()` | `return err` (idiomatic) |
| Middleware | Wrap `http.Handler` | `r.Use()` | `e.Use()` |
| Route groups | Manual prefix | `r.Group()` | `e.Group()` |
| Built-in middleware | None | Few | Many |
| TLS / Auto TLS | Manual | Manual | `e.StartAutoTLS()` |
| WebSocket helper | No | No | Yes |

> **Echo's biggest win over Gin:** Handlers simply `return error`. No `c.Abort()`, no imperative flow control — errors bubble up naturally through middleware.

---

## 2. Installation & Project Setup

```bash
mkdir echo-demo && cd echo-demo
go mod init echo-demo
go get github.com/labstack/echo/v4
go get github.com/labstack/echo/v4/middleware
```

**Minimal project structure:**
```
echo-demo/
├── go.mod
├── go.sum
└── main.go
```

---

## 3. First Echo Server — Hello World

```go
package main

import (
    "net/http"
    "github.com/labstack/echo/v4"
)

func main() {
    e := echo.New()   // create Echo instance

    e.GET("/", func(c echo.Context) error {
        return c.String(http.StatusOK, "Hello, Echo!")
    })

    e.Logger.Fatal(e.Start(":8080"))  // start server
}
```

```bash
go run main.go
curl http://localhost:8080/
# → Hello, Echo!
```

### Key pieces

```go
echo.New()           // creates *Echo (the router + server)
e.Start(":8080")     // wraps http.ListenAndServe — blocks
e.Logger.Fatal(...)  // logs fatal error and exits if Start fails
```

---

## 4. Handler Signature

Every Echo handler is a function with this exact signature:

```go
type HandlerFunc func(echo.Context) error
```

- Takes an `echo.Context` — carries everything about the request/response
- Returns `error` — `nil` means success; non-nil triggers error handler

```go
// Handler as named function (preferred for real apps)
func helloHandler(c echo.Context) error {
    return c.String(http.StatusOK, "Hello!")
}

// Handler as inline closure (good for quick endpoints)
e.GET("/ping", func(c echo.Context) error {
    return c.JSON(http.StatusOK, map[string]string{"status": "ok"})
})
```

---

## 5. Routing — All HTTP Methods

```go
e := echo.New()

e.GET("/users",        getUsers)
e.POST("/users",       createUser)
e.GET("/users/:id",    getUserByID)
e.PUT("/users/:id",    updateUser)
e.PATCH("/users/:id",  patchUser)
e.DELETE("/users/:id", deleteUser)

// Less common
e.HEAD("/users",    headUsers)
e.OPTIONS("/users", optionsUsers)
e.CONNECT("/path",  connectHandler)
e.TRACE("/path",    traceHandler)

// Match any method
e.Any("/debug", debugHandler)

// Match specific methods only
e.Match([]string{"GET", "POST"}, "/search", searchHandler)
```

---

## 6. `echo.Context` — The Core Type

`echo.Context` is an **interface** that wraps `*http.Request` and `http.ResponseWriter` and adds many conveniences. It is the single argument passed to every handler.

```go
// Reading the raw request/response if needed
r := c.Request()   // *http.Request
w := c.Response()  // *echo.Response (wraps http.ResponseWriter)
```

---

## 7. Path Parameters

Defined with `:name` syntax in the route. Retrieved via `c.Param()`.

```go
e.GET("/users/:id", func(c echo.Context) error {
    id := c.Param("id")                    // always a string
    return c.JSON(http.StatusOK, map[string]string{"id": id})
})

// Wildcard — matches everything after /files/
e.GET("/files/*", func(c echo.Context) error {
    filepath := c.Param("*")               // e.g. "/images/logo.png"
    return c.String(http.StatusOK, "File: "+filepath)
})
```

```bash
curl http://localhost:8080/users/42
# → {"id":"42"}

curl http://localhost:8080/files/assets/style.css
# → File: /assets/style.css
```

### Type-safe path params (Echo v4.15+)

```go
import echo "github.com/labstack/echo/v4"

e.GET("/users/:id", func(c echo.Context) error {
    id, err := echo.PathParam[int](c, "id")
    if err != nil {
        return echo.ErrBadRequest
    }
    return c.JSON(http.StatusOK, map[string]int{"id": id})
})
```

---

## 8. Query Parameters

```go
// GET /search?q=golang&page=2&limit=10
e.GET("/search", func(c echo.Context) error {
    q     := c.QueryParam("q")                    // "golang"
    page  := c.QueryParam("page")                 // "2" (string)
    limit := c.QueryParam("limit")                // "10"

    // With default value
    sort := c.QueryParamNames()                   // all param names

    return c.JSON(http.StatusOK, map[string]string{
        "q":     q,
        "page":  page,
        "limit": limit,
    })
})

// All query params as url.Values
params := c.QueryParams()  // map[string][]string
```

### Type-safe query params (v4.15+)

```go
page,  err := echo.QueryParam[int](c, "page")
limit, err := echo.QueryParamOr[int](c, "limit", 20)  // default 20
```

---

## 9. Request Binding — `c.Bind()`

`c.Bind()` reads the request body/params and maps them to a struct automatically based on `Content-Type`.

| Content-Type | Source |
|---|---|
| `application/json` | JSON body |
| `application/xml` | XML body |
| `application/x-www-form-urlencoded` | Form fields |
| `multipart/form-data` | Form + file uploads |

```go
type CreateUserRequest struct {
    Name  string `json:"name"  form:"name"  query:"name"`
    Email string `json:"email" form:"email" query:"email"`
    Age   int    `json:"age"   form:"age"   query:"age"`
}

func createUser(c echo.Context) error {
    req := new(CreateUserRequest)

    if err := c.Bind(req); err != nil {
        return err   // Echo converts to 400 automatically
    }

    // req.Name, req.Email, req.Age are now populated
    return c.JSON(http.StatusCreated, req)
}
```

> **Note:** `c.Bind()` does NOT validate. Use `c.Validate()` after binding for validation.

### Manual JSON decoding (when you need full control)

```go
import "encoding/json"

func handler(c echo.Context) error {
    var req CreateUserRequest
    if err := json.NewDecoder(c.Request().Body).Decode(&req); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, err.Error())
    }
    return c.JSON(http.StatusCreated, req)
}
```

---

## 10. Response Methods

```go
// JSON — most common for REST APIs
c.JSON(http.StatusOK, user)
c.JSON(http.StatusCreated, newUser)
c.JSONPretty(http.StatusOK, user, "  ")   // indented

// String — plain text
c.String(http.StatusOK, "Hello!")

// HTML
c.HTML(http.StatusOK, "<h1>Hello</h1>")

// XML
c.XML(http.StatusOK, user)
c.XMLPretty(http.StatusOK, user, "  ")

// No Content — for DELETE / empty responses
c.NoContent(http.StatusNoContent)          // 204, no body

// Redirect
c.Redirect(http.StatusMovedPermanently, "https://new-url.com") // 301
c.Redirect(http.StatusFound, "/login")                          // 302

// File — serve a file
c.File("/path/to/file.pdf")

// Attachment — force browser download
c.Attachment("/path/to/report.csv", "report.csv")

// Stream — for large responses / SSE
c.Stream(http.StatusOK, "application/octet-stream", reader)

// Raw Blob
c.Blob(http.StatusOK, "image/png", pngBytes)
```

---

## 11. Built-in Middleware

Echo ships with production-ready middleware in `github.com/labstack/echo/v4/middleware`.

```go
import "github.com/labstack/echo/v4/middleware"

e := echo.New()

// Logger — logs every request: method, URI, status, latency
e.Use(middleware.Logger())

// Recover — catches panics, returns 500, prevents crash
e.Use(middleware.Recover())

// CORS — cross-origin resource sharing
e.Use(middleware.CORS())

// Custom CORS config
e.Use(middleware.CORSWithConfig(middleware.CORSConfig{
    AllowOrigins: []string{"https://yourfrontend.com", "http://localhost:3000"},
    AllowMethods: []string{http.MethodGet, http.MethodPost, http.MethodPut, http.MethodDelete},
    AllowHeaders: []string{echo.HeaderOrigin, echo.HeaderContentType, echo.HeaderAuthorization},
}))

// RequestID — adds X-Request-Id header to every request
e.Use(middleware.RequestID())

// Gzip — compresses responses
e.Use(middleware.Gzip())

// Rate Limiter
e.Use(middleware.RateLimiter(middleware.NewRateLimiterMemoryStore(20))) // 20 req/sec

// Body limit — reject bodies above size
e.Use(middleware.BodyLimit("2M"))  // max 2 MB body

// Timeout — cancel requests that take too long
e.Use(middleware.TimeoutWithConfig(middleware.TimeoutConfig{
    Timeout: 30 * time.Second,
}))

// Secure — adds security headers (XSS, clickjacking protection)
e.Use(middleware.Secure())

// Static files
e.Static("/static", "public")  // serve ./public at /static
```

### Middleware order matters

```go
// Apply in this order for most apps:
e.Use(middleware.Recover())    // 1st — catch panics first
e.Use(middleware.Logger())     // 2nd — log all requests
e.Use(middleware.RequestID())  // 3rd — tag each request
e.Use(middleware.CORS())       // 4th — handle CORS preflight
```

---

## 12. Custom Middleware

A middleware is just a function: `func(HandlerFunc) HandlerFunc`

```go
// Pattern
func MyMiddleware(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        // — before handler —
        // ...
        err := next(c)   // call next handler
        // — after handler —
        // ...
        return err
    }
}

e.Use(MyMiddleware)
```

### Example — Request Timing Middleware

```go
func TimingMiddleware(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        start := time.Now()
        err   := next(c)
        c.Response().Header().Set("X-Response-Time",
            time.Since(start).String())
        return err
    }
}
```

### Example — Auth Middleware

```go
func AuthMiddleware(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        token := c.Request().Header.Get("Authorization")
        if token == "" {
            return echo.ErrUnauthorized    // 401
        }
        userID, err := validateToken(token)
        if err != nil {
            return echo.NewHTTPError(http.StatusUnauthorized, "invalid token")
        }
        c.Set("userID", userID)  // pass data to handler
        return next(c)
    }
}
```

### Retrieving values set by middleware

```go
func getProfile(c echo.Context) error {
    userID := c.Get("userID").(int)
    return c.JSON(http.StatusOK, map[string]int{"userID": userID})
}
```

---

## 13. Route Groups

Groups allow shared prefixes and middleware across multiple routes.

```go
e := echo.New()

// Public routes
e.GET("/health", healthCheck)
e.POST("/login", login)
e.POST("/register", register)

// API v1 group — all routes under /api/v1
v1 := e.Group("/api/v1")
{
    v1.GET("/users",        getUsers)
    v1.POST("/users",       createUser)
    v1.GET("/users/:id",    getUserByID)
    v1.PUT("/users/:id",    updateUser)
    v1.DELETE("/users/:id", deleteUser)
}

// Protected group — /api/v1/admin with auth middleware
admin := v1.Group("/admin", AuthMiddleware)
{
    admin.GET("/stats",  getStats)
    admin.POST("/reset", resetData)
}
```

### Groups with dedicated middleware

```go
// Apply middleware only to this group
auth := e.Group("/secure", AuthMiddleware, TimingMiddleware)
auth.GET("/profile", getProfile)
auth.PUT("/profile", updateProfile)
```

---

## 14. Error Handling

### Echo's error model

Handlers return `error`. Echo's **centralized error handler** converts errors to HTTP responses automatically.

```go
// Return nil → success (response already written)
return c.JSON(http.StatusOK, data)

// Return built-in HTTPError → proper status + message
return echo.ErrNotFound          // 404
return echo.ErrUnauthorized      // 401
return echo.ErrForbidden         // 403
return echo.ErrBadRequest        // 400
return echo.ErrInternalServerError  // 500

// Custom HTTPError with message
return echo.NewHTTPError(http.StatusConflict, "email already in use")
return echo.NewHTTPError(http.StatusUnprocessableEntity, "validation failed")

// Standard Go error → becomes 500 Internal Server Error
return fmt.Errorf("database query failed: %w", err)
```

### Pre-defined errors in Echo

```go
echo.ErrBadRequest            // 400
echo.ErrUnauthorized          // 401
echo.ErrForbidden             // 403
echo.ErrNotFound              // 404
echo.ErrMethodNotAllowed      // 405
echo.ErrConflict              // 409
echo.ErrUnsupportedMediaType  // 415
echo.ErrUnprocessableEntity   // 422
echo.ErrTooManyRequests       // 429
echo.ErrInternalServerError   // 500
echo.ErrNotImplemented        // 501
echo.ErrBadGateway            // 502
echo.ErrServiceUnavailable    // 503
```

### Custom error handler (global)

```go
e.HTTPErrorHandler = func(err error, c echo.Context) {
    code := http.StatusInternalServerError
    msg  := "internal server error"

    var he *echo.HTTPError
    if errors.As(err, &he) {
        code = he.Code
        msg  = fmt.Sprintf("%v", he.Message)
    }

    // Never leak internal errors in production
    c.JSON(code, map[string]string{"error": msg})
}
```

---

## 15. Complete CRUD Example

```go
package main

import (
    "net/http"
    "strconv"
    "github.com/labstack/echo/v4"
    "github.com/labstack/echo/v4/middleware"
)

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

var (
    users  = []User{{1, "Alice", "alice@example.com"}, {2, "Bob", "bob@example.com"}}
    nextID = 3
)

func main() {
    e := echo.New()

    e.Use(middleware.Logger())
    e.Use(middleware.Recover())

    api := e.Group("/api/v1")
    {
        api.GET("/users",        getUsers)
        api.POST("/users",       createUser)
        api.GET("/users/:id",    getUserByID)
        api.PUT("/users/:id",    updateUser)
        api.DELETE("/users/:id", deleteUser)
    }

    e.Logger.Fatal(e.Start(":8080"))
}

func getUsers(c echo.Context) error {
    return c.JSON(http.StatusOK, users)
}

func createUser(c echo.Context) error {
    u := new(User)
    if err := c.Bind(u); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, err.Error())
    }
    u.ID = nextID
    nextID++
    users = append(users, *u)
    return c.JSON(http.StatusCreated, u)
}

func getUserByID(c echo.Context) error {
    id, err := strconv.Atoi(c.Param("id"))
    if err != nil {
        return echo.ErrBadRequest
    }
    for _, u := range users {
        if u.ID == id {
            return c.JSON(http.StatusOK, u)
        }
    }
    return echo.ErrNotFound
}

func updateUser(c echo.Context) error {
    id, err := strconv.Atoi(c.Param("id"))
    if err != nil {
        return echo.ErrBadRequest
    }
    input := new(User)
    if err := c.Bind(input); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, err.Error())
    }
    for i, u := range users {
        if u.ID == id {
            input.ID = id
            users[i] = *input
            return c.JSON(http.StatusOK, users[i])
        }
    }
    return echo.ErrNotFound
}

func deleteUser(c echo.Context) error {
    id, err := strconv.Atoi(c.Param("id"))
    if err != nil {
        return echo.ErrBadRequest
    }
    for i, u := range users {
        if u.ID == id {
            users = append(users[:i], users[i+1:]...)
            return c.NoContent(http.StatusNoContent)
        }
    }
    return echo.ErrNotFound
}
```

---

## 16. Testing with curl

```bash
# GET all users
curl http://localhost:8080/api/v1/users

# GET by ID
curl http://localhost:8080/api/v1/users/1

# POST — create
curl -X POST http://localhost:8080/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Charlie","email":"charlie@example.com"}'

# PUT — update
curl -X PUT http://localhost:8080/api/v1/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice Updated","email":"alice_new@example.com"}'

# DELETE
curl -X DELETE http://localhost:8080/api/v1/users/2

# GET with query params
curl "http://localhost:8080/api/v1/users?page=1&limit=10"

# GET non-existent (expect 404)
curl -v http://localhost:8080/api/v1/users/999
```

---

## Key Takeaways

- **`echo.New()`** creates the router; **`e.Start(":port")`** starts the server
- Handler signature is **`func(echo.Context) error`** — returning `error` is idiomatic Go, no `Abort()` calls needed
- **`c.Param("name")`** for path params, **`c.QueryParam("name")`** for query params
- **`c.Bind(&struct)`** auto-decodes JSON/form/XML body based on `Content-Type`
- Response helpers: **`c.JSON()`**, `c.String()`, `c.XML()`, `c.NoContent()`, `c.Redirect()`
- Middleware applied with **`e.Use()`** — order matters; Recover → Logger → others
- **`e.Group("/prefix", middleware...)`** scopes routes and middleware together
- Return **`echo.ErrNotFound`**, **`echo.NewHTTPError(code, msg)`** instead of writing error responses manually
- Echo's centralized error handler converts all returned errors to proper HTTP responses
