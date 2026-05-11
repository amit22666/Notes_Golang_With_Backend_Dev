# All HTTP Status Codes Explained In Detail — Study Notes

**Source:** https://www.youtube.com/watch?v=Sqhmv8ucjUo&list=PLxBfKGlYrg-Hudnm0odvZZDn-J6BEmr1y&index=9  
**Playlist:** 30 Days Golang Backend MasterClass — From Beginner to Production  
**Category:** Backend / HTTP / REST APIs

---

## Prerequisites

- Basic understanding of HTTP protocol
- REST API concepts (from Day 7 notes)
- Go `net/http` package basics

---

## Timeline

| Timestamp | Topic |
|-----------|-------|
| 0:00 | What are HTTP status codes? |
| 2:00 | The 5 families (1xx–5xx) |
| 5:00 | 1xx — Informational |
| 8:00 | 2xx — Success codes |
| 15:00 | 3xx — Redirection codes |
| 22:00 | 4xx — Client error codes |
| 38:00 | 5xx — Server error codes |
| 48:00 | Go constants (`net/http`) |
| 52:00 | Common mistakes in REST APIs |
| 55:00 | Quick decision guide |

---

## 1. What Are HTTP Status Codes?

Every HTTP response carries a **3-digit numeric code** that tells the client what happened with their request.

```
Client ──► Request ──► Server
       ◄── Response ◄──
           [STATUS CODE] + Headers + Body
```

**Structure:**
- First digit → category (1–5)
- Last two digits → specific meaning within that category

```
200  → 2 = success class,   00 = "standard OK"
404  → 4 = client error,    04 = "not found"
500  → 5 = server error,    00 = "general server error"
```

---

## 2. The 5 Families at a Glance

| Range | Family | One-line meaning | Visible to user? |
|-------|--------|-----------------|-----------------|
| **1xx** | Informational | Request received, processing continues | Rarely |
| **2xx** | Success | Request was received, understood, accepted | Yes |
| **3xx** | Redirection | Further action needed to complete request | Sometimes |
| **4xx** | Client Error | Request has a problem — client's fault | Yes |
| **5xx** | Server Error | Server failed to handle a valid request — server's fault | Yes |

> **Memory trick:** "1 info, 2 good, 3 go there, 4 your fault, 5 server's fault"

---

## 3. 1xx — Informational

Temporary responses. Server is telling the client "I got your request, keep going."  
Rare in everyday REST API development.

| Code | Name | When it happens |
|------|------|----------------|
| **100** | Continue | Client sent headers first (Expect: 100-continue); server says "send the body now" |
| **101** | Switching Protocols | Server agrees to upgrade — e.g., HTTP → WebSocket |
| **102** | Processing | Server received request but hasn't finished (WebDAV) |
| **103** | Early Hints | Server sends headers early so client can preload resources |

### 101 — WebSocket Upgrade (most relevant)

```
Client request:
  GET /chat HTTP/1.1
  Upgrade: websocket
  Connection: Upgrade

Server response:
  HTTP/1.1 101 Switching Protocols
  Upgrade: websocket
  Connection: Upgrade
```

```go
// Go — upgrading to WebSocket (gorilla/websocket)
var upgrader = websocket.Upgrader{}

func wsHandler(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil) // sends 101 internally
    if err != nil { return }
    defer conn.Close()
}
```

---

## 4. 2xx — Success

The client's request was received, understood, and accepted. **These are what you want.**

### 200 OK
The universal success code. Use for successful GET, PUT, PATCH, DELETE responses.

```
GET /users/42      → 200 OK  (user found, returned in body)
PUT /users/42      → 200 OK  (updated, return updated resource)
PATCH /users/42    → 200 OK  (partial update done)
DELETE /users/42   → 200 OK  (deleted, return confirmation message)
```

```go
c.JSON(http.StatusOK, user)            // 200
```

---

### 201 Created
A new resource was **successfully created**. Always use for successful POST requests that create something.

Must include a `Location` header pointing to the new resource.

```
POST /users        → 201 Created
  Body: {"id": 5, "name": "Amit"}
  Header: Location: /users/5
```

```go
c.Header("Location", fmt.Sprintf("/users/%d", newUser.ID))
c.JSON(http.StatusCreated, newUser)    // 201
```

---

### 202 Accepted
Request received and **will be processed** — but processing hasn't finished yet.  
Use for async operations (background jobs, queues).

```
POST /reports/generate   → 202 Accepted
  Body: {"job_id": "abc123", "status": "queued"}
```

```go
c.JSON(http.StatusAccepted, gin.H{
    "job_id": jobID,
    "status": "queued",
    "poll_url": "/jobs/" + jobID,
})  // 202
```

---

### 204 No Content
Success, but **nothing to return**. No response body allowed.  
Use for DELETE and sometimes PUT/PATCH when you don't return the updated resource.

```
DELETE /users/42   → 204 No Content   (deleted, nothing to show)
```

```go
c.Status(http.StatusNoContent)         // 204 — no body!
```

> **Warning:** Do NOT send a body with 204. Clients will ignore it or error.

---

### 206 Partial Content
Server is returning only **part of the resource** — used for file downloads, video streaming, resumable uploads.

```
GET /video.mp4
  Request Header:  Range: bytes=0-1023
  Response:        206 Partial Content
                   Content-Range: bytes 0-1023/146515
```

---

### 2xx Summary Table

| Code | Name | Use when |
|------|------|----------|
| **200** | OK | Any successful request with a response body |
| **201** | Created | POST that creates a new resource |
| **202** | Accepted | Async operation accepted but not yet complete |
| **204** | No Content | Success with no body (DELETE, some PUT/PATCH) |
| **206** | Partial Content | Range requests (streaming, resumable downloads) |

---

## 5. 3xx — Redirection

Client must take additional action — usually follow a new URL.

| Code | Name | Permanent? | Method changes? | Use case |
|------|------|-----------|----------------|----------|
| **301** | Moved Permanently | Yes | GET stays GET | Old URL permanently replaced |
| **302** | Found | No | MAY change to GET | Temporary redirect |
| **303** | See Other | No | Always GET | After POST, redirect to result page |
| **304** | Not Modified | — | — | Caching — resource hasn't changed |
| **307** | Temporary Redirect | No | Preserved | Temp redirect, keep original method |
| **308** | Permanent Redirect | Yes | Preserved | Like 301 but method preserved |

### 301 vs 302 vs 307 vs 308

```
301 Moved Permanently   — permanent, method may change to GET   (SEO-friendly)
302 Found               — temporary, method may change to GET
307 Temporary Redirect  — temporary, method PRESERVED (POST stays POST)
308 Permanent Redirect  — permanent, method PRESERVED
```

### 304 Not Modified — Caching

Client sends `If-None-Match` or `If-Modified-Since` header:

```
GET /users HTTP/1.1
If-None-Match: "abc123"

HTTP/1.1 304 Not Modified
→ No body sent. Client uses cached version.
```

```go
// Gin — manual ETag check
func getUsers(c *gin.Context) {
    etag := `"abc123"`
    if c.GetHeader("If-None-Match") == etag {
        c.Status(http.StatusNotModified)  // 304
        return
    }
    c.Header("ETag", etag)
    c.JSON(http.StatusOK, users)
}
```

---

## 6. 4xx — Client Error

The request had a problem **on the client's side**. Client must fix the request.

### 400 Bad Request
Malformed request syntax, invalid JSON, missing required fields, bad query params.

```
POST /users
Body: {name: "Amit"}    ← invalid JSON (key not quoted)

→ 400 Bad Request
  {"error": "invalid JSON body"}
```

```go
if err := c.ShouldBindJSON(&input); err != nil {
    c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})  // 400
    return
}
```

---

### 401 Unauthorized
**"Who are you?"** — Authentication is missing or invalid.  
Client needs to provide credentials (JWT token, API key, etc.).

> Despite the name, 401 really means **unauthenticated**, not unauthorized.

```
GET /profile
Authorization: Bearer invalid_token

→ 401 Unauthorized
  WWW-Authenticate: Bearer realm="api"
  {"error": "invalid or expired token"}
```

```go
c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid token"})  // 401
```

---

### 403 Forbidden
**"I know who you are, but you can't do this."** — Authenticated but not authorized.  
Do NOT reveal that the resource exists to unauthorized users (use 404 if you want to hide it).

```
DELETE /users/99     ← logged in as user 5, trying to delete user 99

→ 403 Forbidden
  {"error": "you cannot delete another user's account"}
```

```go
c.JSON(http.StatusForbidden, gin.H{"error": "insufficient permissions"})  // 403
```

### 401 vs 403 — The Key Difference

| | 401 Unauthorized | 403 Forbidden |
|--|-----------------|--------------|
| **Who are you?** | Unknown (not logged in) | Known (logged in) |
| **Can you fix it?** | Yes — log in first | No — you don't have permission |
| **Real meaning** | Unauthenticated | Unauthorized |

---

### 404 Not Found
The resource **does not exist** at this URL.  
Also used intentionally to hide the existence of a resource from unauthorized users.

```
GET /users/9999   ← no user with this ID

→ 404 Not Found
  {"error": "user not found"}
```

```go
c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})  // 404
```

---

### 405 Method Not Allowed
The HTTP method used is not supported for this endpoint.

```
DELETE /users     ← DELETE on a collection endpoint not supported

→ 405 Method Not Allowed
  Allow: GET, POST
```

```go
// Gin handles this automatically. Manual example:
c.Header("Allow", "GET, POST")
c.JSON(http.StatusMethodNotAllowed, gin.H{"error": "method not allowed"})  // 405
```

---

### 408 Request Timeout
The server timed out waiting for the client to send the full request.

---

### 409 Conflict
The request conflicts with the **current state** of the resource.  
Use when trying to create a duplicate or violating a unique constraint.

```
POST /users
Body: {"email": "existing@example.com"}   ← email already registered

→ 409 Conflict
  {"error": "email already in use"}
```

```go
c.JSON(http.StatusConflict, gin.H{"error": "email already in use"})  // 409
```

---

### 410 Gone
The resource **existed but has been permanently deleted**. Unlike 404, the server is certain it used to exist.

```
GET /posts/42   ← post was deleted and will never come back

→ 410 Gone
```

---

### 413 Content Too Large
The request body exceeds the server's configured limit.

```go
// Gin — limit body size
r.Use(func(c *gin.Context) {
    c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, 10<<20) // 10 MB
    c.Next()
})
```

---

### 415 Unsupported Media Type
The `Content-Type` of the request is not supported by the server.

```
POST /users
Content-Type: text/plain    ← server only accepts application/json

→ 415 Unsupported Media Type
```

---

### 422 Unprocessable Entity
JSON is **syntactically valid** but **semantically wrong** — validation failed.  
Use when data parses correctly but fails business rules.

```
POST /users
Body: {"email": "not-an-email", "age": -5}    ← valid JSON, invalid values

→ 422 Unprocessable Entity
  {"errors": {"email": "must be valid email", "age": "must be positive"}}
```

```go
// 422 vs 400:
// 400 = can't even parse the request
// 422 = parsed fine, but values are wrong

c.JSON(http.StatusUnprocessableEntity, gin.H{
    "errors": validationErrors,
})  // 422
```

---

### 429 Too Many Requests
Client has sent **too many requests** in a given time window — rate limiting.  
Should include `Retry-After` header telling client when to retry.

```
→ 429 Too Many Requests
  Retry-After: 60
  {"error": "rate limit exceeded, try again in 60 seconds"}
```

```go
c.Header("Retry-After", "60")
c.JSON(http.StatusTooManyRequests, gin.H{"error": "rate limit exceeded"})  // 429
```

---

### 451 Unavailable For Legal Reasons
Resource cannot be provided due to legal obligations (censorship, GDPR takedown, court order).  
Named after Fahrenheit 451.

---

### 4xx Summary Table

| Code | Name | Fault | Use when |
|------|------|-------|----------|
| **400** | Bad Request | Malformed request / invalid JSON |
| **401** | Unauthorized | Missing or invalid authentication token |
| **403** | Forbidden | Authenticated but not permitted |
| **404** | Not Found | Resource doesn't exist |
| **405** | Method Not Allowed | Wrong HTTP verb for this endpoint |
| **409** | Conflict | Duplicate entry / state conflict |
| **410** | Gone | Resource permanently deleted |
| **413** | Content Too Large | Request body too big |
| **415** | Unsupported Media Type | Wrong Content-Type header |
| **422** | Unprocessable Entity | Valid JSON but failing business validation |
| **429** | Too Many Requests | Rate limit exceeded |
| **451** | Unavailable For Legal Reasons | Legal takedown |

---

## 7. 5xx — Server Error

The client's request was valid, but **the server failed** to fulfill it. Not the client's fault.

### 500 Internal Server Error
The **catch-all** server error. Unhandled exception, panic, unexpected state.  
Never expose internal error details to clients in production.

```
→ 500 Internal Server Error
  {"error": "something went wrong"}    ← generic, never leak stack traces
```

```go
// Gin recovery middleware catches panics → auto 500
r := gin.Default()  // includes gin.Recovery() which returns 500 on panic

// Manual:
c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})  // 500
```

---

### 501 Not Implemented
The server **does not support** the functionality required by the request.  
Used when a route is planned but not yet built.

```
PATCH /users/42   ← PATCH not implemented yet

→ 501 Not Implemented
```

---

### 502 Bad Gateway
The server acted as a **gateway/proxy** and got an **invalid response** from the upstream server.  
Common in microservices / load balancer setups.

```
User → Nginx (proxy) → Go API → Database
                    ↑
              Go API crashed → Nginx returns 502
```

---

### 503 Service Unavailable
Server is **temporarily unable** to handle requests — overloaded or under maintenance.  
Include `Retry-After` header.

```
→ 503 Service Unavailable
  Retry-After: 30
  {"error": "service temporarily unavailable"}
```

```go
c.Header("Retry-After", "30")
c.JSON(http.StatusServiceUnavailable, gin.H{"error": "service unavailable"})  // 503
```

---

### 504 Gateway Timeout
The server (as gateway) **didn't get a response in time** from an upstream service.

```
User → Nginx → Go API → Database (slow query, times out)
                    ↑
              Database didn't respond → 504
```

---

### 5xx Summary Table

| Code | Name | Use when |
|------|------|----------|
| **500** | Internal Server Error | Unhandled exception / generic server failure |
| **501** | Not Implemented | Feature not built yet |
| **502** | Bad Gateway | Upstream server returned bad response |
| **503** | Service Unavailable | Server overloaded or in maintenance |
| **504** | Gateway Timeout | Upstream service didn't respond in time |

---

## 8. Go `net/http` Status Code Constants

Go's `net/http` package defines every status code as a named constant — always use these instead of raw numbers.

```go
// 1xx
http.StatusContinue           // 100
http.StatusSwitchingProtocols // 101
http.StatusEarlyHints         // 103

// 2xx
http.StatusOK                 // 200
http.StatusCreated            // 201
http.StatusAccepted           // 202
http.StatusNoContent          // 204
http.StatusPartialContent     // 206

// 3xx
http.StatusMovedPermanently   // 301
http.StatusFound              // 302
http.StatusSeeOther           // 303
http.StatusNotModified        // 304
http.StatusTemporaryRedirect  // 307
http.StatusPermanentRedirect  // 308

// 4xx
http.StatusBadRequest                   // 400
http.StatusUnauthorized                 // 401
http.StatusForbidden                    // 403
http.StatusNotFound                     // 404
http.StatusMethodNotAllowed             // 405
http.StatusRequestTimeout               // 408
http.StatusConflict                     // 409
http.StatusGone                         // 410
http.StatusRequestEntityTooLarge        // 413
http.StatusUnsupportedMediaType         // 415
http.StatusUnprocessableEntity          // 422
http.StatusTooManyRequests              // 429
http.StatusUnavailableForLegalReasons   // 451

// 5xx
http.StatusInternalServerError          // 500
http.StatusNotImplemented              // 501
http.StatusBadGateway                  // 502
http.StatusServiceUnavailable          // 503
http.StatusGatewayTimeout              // 504
```

```go
// Lookup the text name of any code
fmt.Println(http.StatusText(200)) // → "OK"
fmt.Println(http.StatusText(404)) // → "Not Found"
fmt.Println(http.StatusText(500)) // → "Internal Server Error"
```

---

## 9. REST API Decision Guide — Which Code to Return?

```
Incoming request
│
├── Can the request be parsed?
│       No  → 400 Bad Request
│
├── Is the user authenticated?
│       No  → 401 Unauthorized
│
├── Is the user authorized?
│       No  → 403 Forbidden
│
├── Does the resource exist?
│       No  → 404 Not Found
│
├── Is the method allowed?
│       No  → 405 Method Not Allowed
│
├── Does validation pass?
│       No  → 422 Unprocessable Entity
│
├── Does it conflict with existing data?
│       Yes → 409 Conflict
│
├── Are rate limits exceeded?
│       Yes → 429 Too Many Requests
│
├── Did the server crash?
│       Yes → 500 Internal Server Error
│
└── Everything OK?
        └── What did the request do?
                POST (created)  → 201 Created
                DELETE (no body)→ 204 No Content
                Async job       → 202 Accepted
                Everything else → 200 OK
```

---

## 10. Common Mistakes to Avoid

| Mistake | Wrong | Correct |
|---------|-------|---------|
| Using 200 for all responses | `POST /users → 200` | `POST /users → 201` |
| Using 200 for errors | `{"error": "not found", "status": 200}` | HTTP 404 status |
| Confusing 401 and 403 | Using 403 for "not logged in" | 401 = not authed, 403 = not allowed |
| Using 400 for validation | `{"email": "invalid"}` returns 400 | Returns 422 |
| Leaking server details in 500 | `{"error": "pq: duplicate key..."}` | `{"error": "internal server error"}` |
| Not using 204 for DELETE | Returning `{"message":"deleted"}` with 200 | `204 No Content` (no body) |
| Using 404 when gone | Resource was deleted permanently | Use 410 Gone |

---

## 11. Status Codes in Gin — Full Pattern

```go
package main

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

func createUser(c *gin.Context) {
    var input struct {
        Name  string `json:"name"  binding:"required"`
        Email string `json:"email" binding:"required,email"`
    }

    // 400 — can't parse request
    if err := c.ShouldBindJSON(&input); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }

    // 409 — duplicate check
    if emailExists(input.Email) {
        c.JSON(http.StatusConflict, gin.H{"error": "email already in use"})
        return
    }

    user, err := db.CreateUser(input.Name, input.Email)
    if err != nil {
        // 500 — server-side failure
        c.JSON(http.StatusInternalServerError, gin.H{"error": "internal server error"})
        return
    }

    // 201 — resource created successfully
    c.Header("Location", fmt.Sprintf("/users/%d", user.ID))
    c.JSON(http.StatusCreated, user)
}

func getUser(c *gin.Context) {
    id := c.Param("id")
    user, err := db.FindUser(id)
    if err != nil {
        // 404 — not found
        c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
        return
    }
    // 200 — success
    c.JSON(http.StatusOK, user)
}

func deleteUser(c *gin.Context) {
    id := c.Param("id")
    if err := db.DeleteUser(id); err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
        return
    }
    // 204 — success, nothing to return
    c.Status(http.StatusNoContent)
}
```

---

## Key Takeaways

- Status codes are **3-digit numbers** grouped into 5 families: 1xx info, 2xx success, 3xx redirect, 4xx client error, 5xx server error
- **Always use named constants** from `net/http` — never hardcode raw numbers
- **2xx** = everything worked; pick the right one (200 / 201 / 202 / 204)
- **401** = "who are you?" (unauthenticated); **403** = "I know you, but no" (unauthorized)
- **400** = malformed request; **422** = valid JSON but bad data/validation failed
- **404** = doesn't exist; **409** = exists but conflicts; **410** = existed but permanently gone
- **429** = rate limited — always include `Retry-After`
- **500** = generic server failure — never expose internal error details
- **502/503/504** = gateway/proxy problems common in microservices and load balancers
