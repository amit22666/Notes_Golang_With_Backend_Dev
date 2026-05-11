# DAY-8 | Echo Framework | Part-2 — Study Notes

**Source:** https://www.youtube.com/watch?v=dS7__u7GHxM&list=PLxBfKGlYrg-Hudnm0odvZZDn-J6BEmr1y&index=11  
**Playlist:** 30 Days Golang Backend MasterClass — From Beginner to Production  
**Category:** Backend / Web Framework  
**Part 1 notes:** `19 go-echo-framework-part1-notes.md`

---

## Prerequisites

- Echo Part 1 — routing, handlers, middleware, binding, groups, error handling
- `go-playground/validator` library concept
- Basic understanding of JWT (JSON Web Tokens)

---

## Timeline

| Timestamp | Topic |
|-----------|-------|
| 0:00 | Recap of Part 1 |
| 2:00 | Validation — why `c.Bind()` alone is not enough |
| 5:00 | Installing go-playground/validator |
| 8:00 | Custom validator — implementing `echo.Validator` |
| 13:00 | Struct validation tags |
| 18:00 | Returning structured validation errors |
| 23:00 | Request & response headers |
| 27:00 | Cookies — set, read, delete |
| 33:00 | File upload — single file |
| 38:00 | File upload — multiple files |
| 43:00 | Static file serving |
| 47:00 | JWT middleware — protect routes |
| 54:00 | Graceful shutdown |
| 59:00 | Testing Echo handlers with `httptest` |

---

## 1. Why `c.Bind()` Alone Is Not Enough

`c.Bind()` only **decodes** (parses JSON/form into struct). It does **not validate**.

```go
// c.Bind() will accept this without error:
// {"name": "", "email": "not-an-email", "age": -5}

type CreateUserReq struct {
    Name  string `json:"name"`
    Email string `json:"email"`
    Age   int    `json:"age"`
}
```

You need a separate **validation** step after binding.

---

## 2. Validation with `go-playground/validator`

### Installation

```bash
go get github.com/go-playground/validator/v10
```

### Step 1 — Create a custom validator struct

Echo's `Validator` interface requires one method: `Validate(i interface{}) error`

```go
import "github.com/go-playground/validator/v10"

type CustomValidator struct {
    v *validator.Validate
}

func (cv *CustomValidator) Validate(i interface{}) error {
    return cv.v.Struct(i)
}
```

### Step 2 — Register it with Echo

```go
func main() {
    e := echo.New()
    e.Validator = &CustomValidator{v: validator.New()}
    // ...
}
```

### Step 3 — Add `validate` tags to your struct

```go
type CreateUserReq struct {
    Name     string `json:"name"     validate:"required,min=2,max=50"`
    Email    string `json:"email"    validate:"required,email"`
    Age      int    `json:"age"      validate:"required,gte=18,lte=120"`
    Password string `json:"password" validate:"required,min=8"`
    Role     string `json:"role"     validate:"oneof=admin user moderator"`
}
```

### Step 4 — Bind then Validate in handler

```go
func createUser(c echo.Context) error {
    req := new(CreateUserReq)

    // Step 1: decode body → struct
    if err := c.Bind(req); err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, err.Error())
    }

    // Step 2: validate struct fields
    if err := c.Validate(req); err != nil {
        return echo.NewHTTPError(http.StatusUnprocessableEntity, err.Error())
    }

    // req is now safe to use
    return c.JSON(http.StatusCreated, req)
}
```

---

## 3. Common Validation Tags Reference

| Tag | Meaning | Example |
|-----|---------|---------|
| `required` | Field must be present and non-zero | `validate:"required"` |
| `min=n` | String min length / number min value | `validate:"min=2"` |
| `max=n` | String max length / number max value | `validate:"max=100"` |
| `gte=n` | Greater than or equal (numbers) | `validate:"gte=18"` |
| `lte=n` | Less than or equal (numbers) | `validate:"lte=120"` |
| `gt=n` | Greater than | `validate:"gt=0"` |
| `lt=n` | Less than | `validate:"lt=1000"` |
| `email` | Valid email address format | `validate:"email"` |
| `url` | Valid URL | `validate:"url"` |
| `len=n` | Exact length | `validate:"len=10"` |
| `oneof=a b c` | Must be one of the listed values | `validate:"oneof=admin user"` |
| `alphanum` | Alphanumeric characters only | `validate:"alphanum"` |
| `numeric` | Only numeric string | `validate:"numeric"` |
| `uuid4` | Valid UUID v4 | `validate:"uuid4"` |
| `omitempty` | Skip validation if field is empty | `validate:"omitempty,email"` |

---

## 4. Structured Validation Error Response

The default validator error is a string dump. For APIs, return field-level errors:

```go
import (
    "github.com/go-playground/validator/v10"
    "github.com/labstack/echo/v4"
    "net/http"
)

type ValidationErrorResponse struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

func formatValidationErrors(err error) []ValidationErrorResponse {
    var errs []ValidationErrorResponse
    for _, e := range err.(validator.ValidationErrors) {
        errs = append(errs, ValidationErrorResponse{
            Field:   e.Field(),
            Message: validationMessage(e),
        })
    }
    return errs
}

func validationMessage(e validator.FieldError) string {
    switch e.Tag() {
    case "required":
        return e.Field() + " is required"
    case "email":
        return e.Field() + " must be a valid email"
    case "min":
        return e.Field() + " must be at least " + e.Param() + " characters"
    case "gte":
        return e.Field() + " must be >= " + e.Param()
    case "oneof":
        return e.Field() + " must be one of: " + e.Param()
    default:
        return e.Field() + " failed " + e.Tag() + " validation"
    }
}

// In handler:
if err := c.Validate(req); err != nil {
    return c.JSON(http.StatusUnprocessableEntity, map[string]interface{}{
        "errors": formatValidationErrors(err),
    })
}
```

**Response:**
```json
{
  "errors": [
    {"field": "Email",    "message": "Email must be a valid email"},
    {"field": "Password", "message": "Password must be at least 8 characters"}
  ]
}
```

---

## 5. Request & Response Headers

### Reading Request Headers

```go
func handler(c echo.Context) error {
    // Read a specific header
    auth      := c.Request().Header.Get("Authorization")
    ct        := c.Request().Header.Get("Content-Type")
    userAgent := c.Request().Header.Get("User-Agent")
    requestID := c.Request().Header.Get("X-Request-Id")

    // Get ALL headers
    headers := c.Request().Header    // http.Header → map[string][]string

    return c.JSON(http.StatusOK, map[string]string{
        "auth":       auth,
        "user_agent": userAgent,
    })
}
```

### Setting Response Headers

```go
func handler(c echo.Context) error {
    // Set a custom response header
    c.Response().Header().Set("X-Custom-Header", "my-value")
    c.Response().Header().Set("X-Request-Id", "abc-123")
    c.Response().Header().Set("Cache-Control", "no-cache, no-store")

    return c.JSON(http.StatusOK, map[string]string{"status": "ok"})
}
```

### Reading Real Client IP

```go
ip := c.RealIP()    // respects X-Forwarded-For / X-Real-IP headers
```

---

## 6. Cookies

### Set a Cookie

```go
func setCookie(c echo.Context) error {
    cookie := new(http.Cookie)
    cookie.Name     = "session_token"
    cookie.Value    = "abc123xyz"
    cookie.Expires  = time.Now().Add(24 * time.Hour)
    cookie.Path     = "/"
    cookie.HttpOnly = true                // not accessible via JS
    cookie.Secure   = true               // HTTPS only
    cookie.SameSite = http.SameSiteStrictMode

    c.SetCookie(cookie)
    return c.String(http.StatusOK, "cookie set")
}
```

### Read a Cookie

```go
func readCookie(c echo.Context) error {
    cookie, err := c.Cookie("session_token")
    if err != nil {
        // err = http.ErrNoCookie if not found
        return echo.ErrUnauthorized
    }
    return c.String(http.StatusOK, "hello, "+cookie.Value)
}
```

### Read All Cookies

```go
func readAllCookies(c echo.Context) error {
    cookies := c.Cookies()  // []*http.Cookie
    for _, cookie := range cookies {
        fmt.Printf("Name: %s  Value: %s\n", cookie.Name, cookie.Value)
    }
    return c.String(http.StatusOK, "done")
}
```

### Delete a Cookie (expire it immediately)

```go
func deleteCookie(c echo.Context) error {
    cookie := new(http.Cookie)
    cookie.Name    = "session_token"
    cookie.Value   = ""
    cookie.Expires = time.Unix(0, 0)   // set to past → browser deletes it
    cookie.MaxAge  = -1
    cookie.Path    = "/"
    c.SetCookie(cookie)
    return c.String(http.StatusOK, "cookie deleted")
}
```

---

## 7. File Upload

### Single File Upload

```go
import (
    "io"
    "os"
    "net/http"
    "github.com/labstack/echo/v4"
)

func uploadFile(c echo.Context) error {
    // 1. Get file from form field named "avatar"
    file, err := c.FormFile("avatar")
    if err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, "file is required")
    }

    // 2. Open the uploaded file
    src, err := file.Open()
    if err != nil {
        return err
    }
    defer src.Close()

    // 3. Create destination file on disk
    dst, err := os.Create("./uploads/" + file.Filename)
    if err != nil {
        return err
    }
    defer dst.Close()

    // 4. Copy content
    if _, err = io.Copy(dst, src); err != nil {
        return err
    }

    return c.JSON(http.StatusOK, map[string]string{
        "filename": file.Filename,
        "size":     fmt.Sprintf("%d bytes", file.Size),
    })
}

// Route
e.POST("/upload", uploadFile)
```

**Test with curl:**
```bash
curl -X POST http://localhost:8080/upload \
  -F "avatar=@/path/to/photo.jpg"
```

### Multiple File Upload

```go
func uploadMultiple(c echo.Context) error {
    form, err := c.MultipartForm()
    if err != nil {
        return err
    }

    files := form.File["files"]  // "files" = form field name
    var uploaded []string

    for _, file := range files {
        src, err := file.Open()
        if err != nil {
            return err
        }
        defer src.Close()

        dst, err := os.Create("./uploads/" + file.Filename)
        if err != nil {
            return err
        }
        defer dst.Close()

        if _, err = io.Copy(dst, src); err != nil {
            return err
        }
        uploaded = append(uploaded, file.Filename)
    }

    return c.JSON(http.StatusOK, map[string]interface{}{
        "uploaded": uploaded,
        "count":    len(uploaded),
    })
}

// Route
e.POST("/upload/multiple", uploadMultiple)
```

**Test with curl:**
```bash
curl -X POST http://localhost:8080/upload/multiple \
  -F "files=@photo1.jpg" \
  -F "files=@photo2.png"
```

### File size validation

```go
const maxUploadSize = 10 << 20  // 10 MB

func uploadFile(c echo.Context) error {
    c.Request().Body = http.MaxBytesReader(c.Response(), c.Request().Body, maxUploadSize)

    file, err := c.FormFile("avatar")
    if err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, "file too large or missing")
    }
    // ...
}
```

---

## 8. Static File Serving

Serve an entire directory of static assets (HTML, CSS, JS, images):

```go
// Serve files from ./public/ at /static/*
e.Static("/static", "public")

// Examples:
// GET /static/style.css     → serves ./public/style.css
// GET /static/js/app.js     → serves ./public/js/app.js
// GET /static/img/logo.png  → serves ./public/img/logo.png

// Serve a single specific file
e.File("/favicon.ico", "public/favicon.ico")
e.File("/robots.txt",  "public/robots.txt")

// Serve from embedded filesystem (embed package)
//go:embed public
var embeddedFiles embed.FS
e.StaticFS("/static", echo.MustSubFS(embeddedFiles, "public"))
```

---

## 9. JWT Middleware

### Installation

```bash
go get github.com/labstack/echo-jwt/v4
go get github.com/golang-jwt/jwt/v5
```

### Generate a Token (login handler)

```go
import (
    "time"
    "github.com/golang-jwt/jwt/v5"
)

var jwtSecret = []byte("your-secret-key")  // use env var in production

type JWTClaims struct {
    UserID int    `json:"user_id"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

func login(c echo.Context) error {
    var req struct {
        Email    string `json:"email"`
        Password string `json:"password"`
    }
    c.Bind(&req)

    // TODO: verify credentials against DB
    if req.Email != "admin@example.com" || req.Password != "secret" {
        return echo.ErrUnauthorized
    }

    claims := &JWTClaims{
        UserID: 1,
        Role:   "admin",
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    signed, err := token.SignedString(jwtSecret)
    if err != nil {
        return echo.ErrInternalServerError
    }

    return c.JSON(http.StatusOK, map[string]string{"token": signed})
}
```

### Protect Routes with JWT Middleware

```go
import echojwt "github.com/labstack/echo-jwt/v4"

func main() {
    e := echo.New()

    // Public routes — no auth
    e.POST("/login", login)
    e.POST("/register", register)

    // Protected routes — require valid JWT
    protected := e.Group("/api")
    protected.Use(echojwt.WithConfig(echojwt.Config{
        SigningKey: jwtSecret,
        // Token can be in Authorization header OR cookie
        TokenLookup: "header:Authorization:Bearer ,cookie:token",
    }))
    protected.GET("/profile", getProfile)
    protected.PUT("/profile", updateProfile)
}
```

### Extract Claims in Handler

```go
import (
    "github.com/golang-jwt/jwt/v5"
    "github.com/labstack/echo/v4"
)

func getProfile(c echo.Context) error {
    // Get the token stored by JWT middleware
    token, ok := c.Get("user").(*jwt.Token)
    if !ok {
        return echo.ErrUnauthorized
    }

    claims, ok := token.Claims.(jwt.MapClaims)
    if !ok {
        return echo.ErrUnauthorized
    }

    userID := int(claims["user_id"].(float64))
    role   := claims["role"].(string)

    return c.JSON(http.StatusOK, map[string]interface{}{
        "user_id": userID,
        "role":    role,
    })
}
```

**Test with curl:**
```bash
# 1. Login → get token
curl -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"secret"}'

# 2. Use token
curl http://localhost:8080/api/profile \
  -H "Authorization: Bearer <token>"
```

---

## 10. Graceful Shutdown

Stops the server cleanly after finishing in-flight requests, instead of abruptly killing connections.

```go
package main

import (
    "context"
    "errors"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/labstack/echo/v4"
    "github.com/labstack/echo/v4/middleware"
)

func main() {
    e := echo.New()
    e.Use(middleware.Logger())
    e.Use(middleware.Recover())

    e.GET("/", func(c echo.Context) error {
        return c.String(http.StatusOK, "Hello!")
    })

    // Start server in a goroutine (non-blocking)
    go func() {
        if err := e.Start(":8080"); err != nil && !errors.Is(err, http.ErrServerClosed) {
            e.Logger.Fatal("server error: ", err)
        }
    }()

    // Block until OS interrupt signal (Ctrl+C, SIGTERM from Docker/K8s)
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
    <-quit

    e.Logger.Info("shutting down server…")

    // Give in-flight requests 10 seconds to complete
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    if err := e.Shutdown(ctx); err != nil {
        e.Logger.Fatal("forced shutdown: ", err)
    }

    e.Logger.Info("server stopped cleanly")
}
```

### Why graceful shutdown matters

```
Without graceful shutdown:
  Request in-flight → Ctrl+C → connection killed → client gets error

With graceful shutdown:
  Ctrl+C received → stop accepting new connections
                  → wait for existing requests to finish (up to 10s)
                  → exit cleanly
```

---

## 11. Testing Echo Handlers

Echo handlers are regular functions — test them using `net/http/httptest`.

### Basic handler test

```go
import (
    "net/http"
    "net/http/httptest"
    "strings"
    "testing"

    "github.com/labstack/echo/v4"
    "github.com/stretchr/testify/assert"
)

func TestGetUsers(t *testing.T) {
    e := echo.New()

    req  := httptest.NewRequest(http.MethodGet, "/users", nil)
    rec  := httptest.NewRecorder()
    c    := e.NewContext(req, rec)

    if assert.NoError(t, getUsers(c)) {
        assert.Equal(t, http.StatusOK, rec.Code)
    }
}
```

### Test with path parameters

```go
func TestGetUserByID(t *testing.T) {
    e := echo.New()

    req := httptest.NewRequest(http.MethodGet, "/", nil)
    rec := httptest.NewRecorder()
    c   := e.NewContext(req, rec)

    // Set path param manually
    c.SetParamNames("id")
    c.SetParamValues("1")

    if assert.NoError(t, getUserByID(c)) {
        assert.Equal(t, http.StatusOK, rec.Code)
    }
}
```

### Test POST with JSON body

```go
func TestCreateUser(t *testing.T) {
    e := echo.New()
    e.Validator = &CustomValidator{v: validator.New()}

    body := `{"name":"Alice","email":"alice@example.com","age":25,"password":"secret123"}`

    req := httptest.NewRequest(http.MethodPost, "/users",
        strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")

    rec := httptest.NewRecorder()
    c   := e.NewContext(req, rec)

    if assert.NoError(t, createUser(c)) {
        assert.Equal(t, http.StatusCreated, rec.Code)
    }
}
```

### Test expecting an error status

```go
func TestGetUserNotFound(t *testing.T) {
    e := echo.New()

    req := httptest.NewRequest(http.MethodGet, "/", nil)
    rec := httptest.NewRecorder()
    c   := e.NewContext(req, rec)
    c.SetParamNames("id")
    c.SetParamValues("9999")

    err := getUserByID(c)

    var he *echo.HTTPError
    if assert.ErrorAs(t, err, &he) {
        assert.Equal(t, http.StatusNotFound, he.Code)
    }
}
```

### Run tests

```bash
go test ./...
go test -v ./...
go test -run TestCreateUser ./...
go test -cover ./...
```

---

## 12. Full App — Putting It All Together

```go
package main

import (
    "context"
    "errors"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/go-playground/validator/v10"
    "github.com/labstack/echo/v4"
    "github.com/labstack/echo/v4/middleware"
    echojwt "github.com/labstack/echo-jwt/v4"
)

type CustomValidator struct{ v *validator.Validate }
func (cv *CustomValidator) Validate(i interface{}) error { return cv.v.Struct(i) }

func main() {
    e := echo.New()
    e.Validator = &CustomValidator{v: validator.New()}

    // Global middleware
    e.Use(middleware.Recover())
    e.Use(middleware.Logger())
    e.Use(middleware.RequestID())
    e.Use(middleware.CORSWithConfig(middleware.CORSConfig{
        AllowOrigins: []string{"*"},
        AllowMethods: []string{http.MethodGet, http.MethodPost, http.MethodPut, http.MethodDelete},
    }))

    // Static files
    e.Static("/static", "public")

    // Public routes
    e.POST("/login",    login)
    e.POST("/register", register)

    // Protected API
    api := e.Group("/api/v1")
    api.Use(echojwt.WithConfig(echojwt.Config{SigningKey: []byte("secret")}))
    api.GET("/users",           getUsers)
    api.POST("/users",          createUser)
    api.GET("/users/:id",       getUserByID)
    api.PUT("/users/:id",       updateUser)
    api.DELETE("/users/:id",    deleteUser)
    api.POST("/upload",         uploadFile)

    // Graceful shutdown
    go func() {
        if err := e.Start(":8080"); err != nil && !errors.Is(err, http.ErrServerClosed) {
            e.Logger.Fatal(err)
        }
    }()

    quit := make(chan os.Signal, 1)
    signal.Notify(quit, os.Interrupt, syscall.SIGTERM)
    <-quit

    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    e.Shutdown(ctx)
}
```

---

## Key Takeaways

- **`c.Bind()` ≠ validation** — always follow binding with `c.Validate()` for input safety
- Register `go-playground/validator` via `e.Validator = &CustomValidator{v: validator.New()}`
- `validate` struct tags control rules: `required`, `email`, `min`, `max`, `gte`, `lte`, `oneof`
- Return **field-level errors** (not just a raw string) using `validator.ValidationErrors`
- **Cookies:** `c.SetCookie()` to write, `c.Cookie("name")` to read, set `MaxAge=-1` to delete
- **Headers:** `c.Request().Header.Get()` to read, `c.Response().Header().Set()` to write
- **File upload:** `c.FormFile("field")` for single, `c.MultipartForm()` for multiple
- **Static files:** `e.Static("/prefix", "dir")` serves a whole directory
- **JWT:** `echojwt.WithConfig()` protects route groups; extract claims with `c.Get("user")`
- **Graceful shutdown:** start in goroutine → listen for OS signal → call `e.Shutdown(ctx)` with timeout
- **Testing:** use `httptest.NewRequest` + `e.NewContext` + set params manually with `c.SetParamNames/Values`
