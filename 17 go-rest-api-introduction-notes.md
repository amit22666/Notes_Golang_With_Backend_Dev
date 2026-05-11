# DAY-7 | Introduction to REST APIs — Study Notes

**Source:** https://www.youtube.com/watch?v=Rj-kJeJqU1c&list=PLxBfKGlYrg-Hudnm0odvZZDn-J6BEmr1y&index=8  
**Playlist:** 30 Days Golang Backend MasterClass — From Beginner to Production  
**Category:** Backend / Web APIs

---

## Prerequisites

- Go fundamentals (structs, functions, packages)
- Basic understanding of HTTP protocol
- Go modules (`go mod init`)

---

## Timeline

| Timestamp | Topic |
|-----------|-------|
| 0:00 | What is an API? |
| 2:00 | What is REST? |
| 5:00 | REST Constraints / Principles |
| 9:00 | HTTP Methods (CRUD mapping) |
| 13:00 | HTTP Status Codes |
| 17:00 | URL & Resource Design |
| 21:00 | JSON — the data format |
| 25:00 | Building REST API with `net/http` |
| 32:00 | Introducing Gin framework |
| 38:00 | Routing, Path Params, Query Params |
| 44:00 | Request body — JSON binding |
| 48:00 | Response with status codes |
| 52:00 | Testing with curl / Postman |

---

## 1. What is an API?

**API** = Application Programming Interface — a contract that lets two software systems communicate.

```
Client  ──── HTTP Request ────►  Server (API)
        ◄─── HTTP Response ───   (returns data)
```

- The client does NOT need to know how the server works internally
- The server exposes a fixed set of **endpoints** (URLs)
- Data is exchanged in a standard format (**JSON**)

**Real-world analogy:** A waiter in a restaurant. You (client) don't go into the kitchen (server). You give your order (request) to the waiter (API), who brings back your food (response).

---

## 2. What is REST?

**REST** = **RE**presentational **S**tate **T**ransfer

- Introduced by Roy Fielding in his 2000 PhD dissertation
- An **architectural style** (not a protocol or standard)
- Uses HTTP as the transport layer
- Treats everything as a **resource** identified by a URL

```
https://api.example.com/users          ← collection resource
https://api.example.com/users/42       ← single resource (id=42)
https://api.example.com/users/42/posts ← nested resource
```

---

## 3. REST Constraints (6 Principles)

| Constraint | Meaning |
|------------|---------|
| **Client-Server** | UI and data storage are separated — each can evolve independently |
| **Stateless** | Each request must contain ALL information needed; server stores NO session |
| **Cacheable** | Responses must define whether they can be cached |
| **Uniform Interface** | Consistent URL structure + HTTP methods for all resources |
| **Layered System** | Client doesn't know if it talks to actual server or a proxy/load balancer |
| **Code on Demand** *(optional)* | Server can send executable code to client (e.g., JavaScript) |

> **Most important:** **Stateless** — this is what makes REST scalable. Any server instance can handle any request.

---

## 4. HTTP Methods → CRUD Mapping

| HTTP Method | CRUD Operation | Description | Request Body |
|-------------|---------------|-------------|-------------|
| `GET` | Read | Fetch resource(s) | No |
| `POST` | Create | Create new resource | Yes (JSON) |
| `PUT` | Update (full) | Replace entire resource | Yes (JSON) |
| `PATCH` | Update (partial) | Modify specific fields | Yes (JSON) |
| `DELETE` | Delete | Remove resource | No |

```
GET    /users        → list all users
GET    /users/42     → get user with id=42
POST   /users        → create a new user
PUT    /users/42     → replace user 42 entirely
PATCH  /users/42     → update specific fields of user 42
DELETE /users/42     → delete user 42
```

**Safe methods:** GET, HEAD, OPTIONS — do NOT modify data  
**Idempotent methods:** GET, PUT, DELETE — calling multiple times = same result

---

## 5. HTTP Status Codes

| Range | Category | Common Codes |
|-------|----------|-------------|
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirection | 301 Moved Permanently, 304 Not Modified |
| 4xx | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity |
| 5xx | Server Error | 500 Internal Server Error, 503 Service Unavailable |

```
200 OK          → GET, PUT, PATCH, DELETE success
201 Created     → POST success (resource created)
204 No Content  → DELETE success (nothing to return)
400 Bad Request → invalid JSON / missing required fields
401 Unauthorized→ missing or invalid token
403 Forbidden   → authenticated but not allowed
404 Not Found   → resource doesn't exist
500 Internal    → server-side bug
```

---

## 6. URL & Resource Design — Best Practices

```
✅ GOOD (noun, plural, hierarchical)
GET  /users
GET  /users/42
GET  /users/42/orders
POST /users

❌ BAD (verb in URL — REST anti-pattern)
GET  /getUsers
POST /createUser
GET  /deleteUser?id=42
```

**Rules:**
- Use **nouns**, not verbs (the HTTP method IS the verb)
- Use **plural** names (`/users`, not `/user`)
- Use **lowercase** and hyphens (`/blog-posts`, not `/blogPosts`)
- Nest resources to show relationships (`/users/42/orders`)
- Keep URLs **stable** — clients depend on them

---

## 7. JSON — The Data Format

**JSON** = JavaScript Object Notation — the universal language of REST APIs.

```json
{
  "id": 1,
  "name": "Amit Kumar",
  "email": "amit@example.com",
  "age": 25,
  "active": true,
  "tags": ["golang", "backend"],
  "address": {
    "city": "Bangalore",
    "country": "India"
  }
}
```

### Go Struct ↔ JSON (struct tags)

```go
type User struct {
    ID       int      `json:"id"`
    Name     string   `json:"name"`
    Email    string   `json:"email"`
    Password string   `json:"-"`          // omit from JSON (sensitive)
    Age      int      `json:"age,omitempty"` // omit if zero value
}
```

| Tag | Effect |
|-----|--------|
| `json:"name"` | Use `name` as JSON key |
| `json:"-"` | Never include in JSON output |
| `json:"age,omitempty"` | Omit field if value is zero/empty |

```go
// Encoding (Go struct → JSON bytes)
user := User{ID: 1, Name: "Amit"}
data, err := json.Marshal(user)
// data = []byte(`{"id":1,"name":"Amit","email":""}`)

// Decoding (JSON bytes → Go struct)
var u User
err = json.Unmarshal(data, &u)
```

---

## 8. Building a REST API with `net/http`

The standard library approach — no external dependencies.

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "strings"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

var users = []User{
    {ID: 1, Name: "Alice"},
    {ID: 2, Name: "Bob"},
}

func writeJSON(w http.ResponseWriter, status int, data any) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func usersHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        writeJSON(w, http.StatusOK, users)

    case http.MethodPost:
        var u User
        if err := json.NewDecoder(r.Body).Decode(&u); err != nil {
            writeJSON(w, http.StatusBadRequest, map[string]string{"error": err.Error()})
            return
        }
        u.ID = len(users) + 1
        users = append(users, u)
        writeJSON(w, http.StatusCreated, u)

    default:
        http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
    }
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/users", usersHandler)

    log.Println("Server running on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

**Problems with `net/http` for REST:**
- No built-in path parameter extraction (`/users/{id}`)
- No route grouping
- Manual method dispatching (`switch r.Method`)
- Verbose JSON response helpers needed

---

## 9. Gin Framework — The Better Way

**Gin** is the most popular Go web framework. Minimal overhead, clean API.

```bash
go get github.com/gin-gonic/gin
```

### Setup

```go
package main

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

var users = []User{
    {ID: 1, Name: "Alice", Email: "alice@example.com"},
    {ID: 2, Name: "Bob", Email: "bob@example.com"},
}

func main() {
    r := gin.Default()

    r.GET("/users", getUsers)
    r.GET("/users/:id", getUserByID)
    r.POST("/users", createUser)
    r.PUT("/users/:id", updateUser)
    r.DELETE("/users/:id", deleteUser)

    r.Run(":8080")   // listens on 0.0.0.0:8080
}
```

### Handler Functions

```go
// GET /users — return all users
func getUsers(c *gin.Context) {
    c.JSON(http.StatusOK, users)
}

// GET /users/:id — return one user
func getUserByID(c *gin.Context) {
    idStr := c.Param("id")   // path parameter
    id, err := strconv.Atoi(idStr)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "invalid id"})
        return
    }
    for _, u := range users {
        if u.ID == id {
            c.JSON(http.StatusOK, u)
            return
        }
    }
    c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
}

// POST /users — create a user
func createUser(c *gin.Context) {
    var input User
    if err := c.ShouldBindJSON(&input); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    input.ID = len(users) + 1
    users = append(users, input)
    c.JSON(http.StatusCreated, input)
}

// PUT /users/:id — replace a user
func updateUser(c *gin.Context) {
    id, _ := strconv.Atoi(c.Param("id"))
    var input User
    if err := c.ShouldBindJSON(&input); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    for i, u := range users {
        if u.ID == id {
            input.ID = id
            users[i] = input
            c.JSON(http.StatusOK, users[i])
            return
        }
    }
    c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
}

// DELETE /users/:id — remove a user
func deleteUser(c *gin.Context) {
    id, _ := strconv.Atoi(c.Param("id"))
    for i, u := range users {
        if u.ID == id {
            users = append(users[:i], users[i+1:]...)
            c.JSON(http.StatusNoContent, nil)
            return
        }
    }
    c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
}
```

---

## 10. Routing — Path Params & Query Params

```go
// Path parameter — part of the URL path
r.GET("/users/:id", handler)         // /users/42
id := c.Param("id")                  // → "42"

// Optional path segment (wildcard)
r.GET("/files/*filepath", handler)   // /files/a/b/c.txt
path := c.Param("filepath")          // → "/a/b/c.txt"

// Query parameter — after the ?
r.GET("/users", handler)             // /users?page=2&limit=10
page  := c.Query("page")            // → "2"
limit := c.DefaultQuery("limit", "20") // → "10" (or "20" if missing)
```

### Route Groups

```go
// Group routes with a common prefix
v1 := r.Group("/api/v1")
{
    v1.GET("/users", getUsers)
    v1.POST("/users", createUser)
    v1.GET("/users/:id", getUserByID)
    v1.PUT("/users/:id", updateUser)
    v1.DELETE("/users/:id", deleteUser)
}
```

---

## 11. Request Binding — JSON Input

Gin provides multiple binding methods:

```go
// ShouldBindJSON — returns error (preferred, doesn't abort)
var input CreateUserInput
if err := c.ShouldBindJSON(&input); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
    return
}

// BindJSON — calls c.AbortWithError on failure (auto-aborts)
if err := c.BindJSON(&input); err != nil {
    return  // response already sent
}
```

### Input Validation with struct tags

```go
type CreateUserInput struct {
    Name  string `json:"name"  binding:"required"`
    Email string `json:"email" binding:"required,email"`
    Age   int    `json:"age"   binding:"required,min=18,max=100"`
}
```

| Tag | Meaning |
|-----|---------|
| `binding:"required"` | Field must be present |
| `binding:"email"` | Must be a valid email |
| `binding:"min=18"` | Numeric minimum |
| `binding:"max=100"` | Numeric maximum |
| `binding:"len=10"` | String must be exactly 10 chars |

---

## 12. Response Helpers in Gin

```go
// JSON response
c.JSON(http.StatusOK, gin.H{"message": "success"})

// gin.H is just map[string]any — shorthand
c.JSON(200, gin.H{
    "data":    user,
    "message": "user created",
})

// Respond with a struct directly
c.JSON(http.StatusCreated, newUser)

// IndentedJSON — pretty-printed (good for debugging)
c.IndentedJSON(http.StatusOK, users)

// String response
c.String(http.StatusOK, "Hello %s", name)

// XML response
c.XML(http.StatusOK, user)

// Abort (stop handler chain)
c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "unauthorized"})
```

---

## 13. Testing with curl

```bash
# GET all users
curl http://localhost:8080/api/v1/users

# GET single user
curl http://localhost:8080/api/v1/users/1

# POST — create user
curl -X POST http://localhost:8080/api/v1/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Charlie","email":"charlie@example.com","age":25}'

# PUT — update user
curl -X PUT http://localhost:8080/api/v1/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice Updated","email":"alice@example.com","age":30}'

# DELETE — delete user
curl -X DELETE http://localhost:8080/api/v1/users/1

# With verbose output (see headers + status code)
curl -v http://localhost:8080/api/v1/users/999
```

---

## 14. Request / Response Flow (Full Picture)

```
┌─────────┐          HTTP Request            ┌──────────────┐
│  Client │ ─────────────────────────────►  │  Gin Router  │
│(curl/   │  Method + URL + Headers + Body   │              │
│ browser)│                                  │  Match route │
│         │                                  │  Run handler │
│         │          HTTP Response            │              │
│         │ ◄─────────────────────────────── │  gin.Context │
└─────────┘  Status + Headers + JSON body    └──────────────┘
```

**Inside the handler:**

```
1. Extract path params     →  c.Param("id")
2. Extract query params    →  c.Query("page")
3. Decode request body     →  c.ShouldBindJSON(&input)
4. Validate input          →  binding tags / manual checks
5. Business logic          →  DB query, computation
6. Send response           →  c.JSON(statusCode, data)
```

---

## 15. `net/http` vs Gin — Comparison

| Feature | `net/http` | Gin |
|---------|-----------|-----|
| Path parameters | Manual parsing | `c.Param("id")` |
| Query parameters | `r.URL.Query().Get("page")` | `c.Query("page")` |
| JSON decoding | `json.NewDecoder(r.Body).Decode` | `c.ShouldBindJSON` |
| JSON response | `json.NewEncoder(w).Encode` | `c.JSON` |
| Route groups | Manual prefix logic | `r.Group("/prefix")` |
| Middleware | `http.Handler` wrapping | `r.Use(middleware)` |
| Performance | Baseline | ~40x faster than other frameworks |
| External dependency | None | `github.com/gin-gonic/gin` |

**Rule of thumb:**
- Simple internal tooling → `net/http`
- Production REST APIs → **Gin** (or Echo/Chi)

---

## Key Takeaways

- **REST** is an architectural style using HTTP methods + URLs to represent operations on resources
- **Stateless** — every request is independent; server keeps no session
- **HTTP methods** map directly to CRUD: GET=Read, POST=Create, PUT=Update, DELETE=Delete
- **Status codes** communicate the result: 2xx=success, 4xx=client error, 5xx=server error
- **JSON** is the standard data format; Go struct tags control field naming
- **Gin** handles routing, JSON binding, and response helpers far cleaner than raw `net/http`
- Use `c.ShouldBindJSON` + `binding` tags for input validation
- Group routes under `/api/v1` for versioning from day one
