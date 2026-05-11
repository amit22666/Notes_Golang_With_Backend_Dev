# DAY-10 | JWT (JSON Web Tokens) — Study Notes

**Source:** https://www.youtube.com/watch?v=c-tJRpAOJhk&list=PLxBfKGlYrg-Hudnm0odvZZDn-J6BEmr1y&index=13  
**Playlist:** 30 Days Golang Backend MasterClass — From Beginner to Production  
**Category:** Backend / Authentication / Security

---

## Prerequisites

- Go structs, interfaces, error handling
- REST APIs and HTTP headers
- Basic understanding of authentication vs authorisation

---

## Timeline

| Timestamp | Topic |
|-----------|-------|
| 0:00 | Session-based vs token-based auth |
| 3:00 | What is JWT? |
| 6:00 | JWT structure — header, payload, signature |
| 11:00 | Standard (registered) claims |
| 15:00 | Signing algorithms — HS256 vs RS256 |
| 20:00 | Installation — golang-jwt/jwt |
| 22:00 | Generate a token (HS256) |
| 28:00 | Parse and validate a token |
| 33:00 | Custom claims struct |
| 38:00 | MapClaims vs custom struct |
| 42:00 | All JWT error types |
| 46:00 | Access token + refresh token pattern |
| 52:00 | JWT middleware for Echo |
| 57:00 | Token revocation and blacklisting |
| 60:00 | Security best practices |

---

## 1. Session-based vs Token-based Auth

### Session-based (stateful)

```
Client → POST /login → Server creates session → stores in DB/Redis
Client stores session_id in cookie
Every request → Server looks up session_id in DB → slow at scale
```

**Problem:** Server must store session for every user → hard to scale horizontally.

### Token-based / JWT (stateless)

```
Client → POST /login → Server creates signed token → returns to client
Client stores token (localStorage / cookie)
Every request → Server verifies signature locally → no DB lookup needed
```

**Benefits:**
- Stateless — no server-side storage required
- Scalable — any server can verify (no shared session store)
- Cross-domain — works for mobile apps, SPAs, microservices
- Self-contained — token carries user info

**Trade-off:** Cannot invalidate a token before expiry without a blacklist.

---

## 2. What Is JWT?

**JWT** = **J**SON **W**eb **T**oken — a compact, URL-safe, self-contained token format defined in [RFC 7519](https://tools.ietf.org/html/rfc7519).

A JWT is three Base64URL-encoded strings joined by dots:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJ1c2VyX2lkIjoxLCJyb2xlIjoiYWRtaW4iLCJleHAiOjE3MDAwMDAwMDB9
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

  └─── HEADER ──┘ └───────────── PAYLOAD ─────────────┘ └── SIGNATURE ──┘
```

> The header and payload are **readable by anyone** (just Base64 decoded).  
> The signature proves they **have not been tampered with**.

---

## 3. JWT Structure — Deep Dive

### 3.1 Header

Specifies the **token type** and the **signing algorithm**.

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Base64URL encoded → `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9`

### 3.2 Payload (Claims)

Contains the **claims** — statements about the user + metadata.

```json
{
  "sub":  "1",
  "iss":  "my-app",
  "aud":  ["web", "mobile"],
  "iat":  1700000000,
  "exp":  1700003600,
  "nbf":  1700000000,
  "jti":  "a1b2c3d4",
  "user_id": 42,
  "role": "admin",
  "email": "amit@example.com"
}
```

> **Never put passwords, credit cards, or secrets in the payload — it is readable.**

### 3.3 Signature

Created by the server using:

```
HMACSHA256(
  base64url(header) + "." + base64url(payload),
  secret_key
)
```

The signature **verifies authenticity**. If anyone changes even one character in the payload, the signature will not match.

### How verification works

```
Incoming token:  header.payload.signature

Server recomputes: HMAC(header + "." + payload, secret)
Compare:           recomputed == signature from token?
  Yes → token is valid
  No  → token was tampered with → reject
```

---

## 4. Standard (Registered) Claims

Defined in RFC 7519. All optional but widely used.

| Claim | JSON key | Meaning |
|-------|----------|---------|
| **Issuer** | `iss` | Who issued the token (e.g. `"my-api"`) |
| **Subject** | `sub` | Who the token is about (usually user ID) |
| **Audience** | `aud` | Who the token is intended for |
| **Expires At** | `exp` | Unix timestamp — token is invalid after this |
| **Not Before** | `nbf` | Token is invalid before this time |
| **Issued At** | `iat` | When the token was created |
| **JWT ID** | `jti` | Unique token identifier (used for revocation) |

```go
// In golang-jwt/jwt/v5
type RegisteredClaims struct {
    Issuer    string       `json:"iss,omitempty"`
    Subject   string       `json:"sub,omitempty"`
    Audience  ClaimStrings `json:"aud,omitempty"`
    ExpiresAt *NumericDate `json:"exp,omitempty"`
    NotBefore *NumericDate `json:"nbf,omitempty"`
    IssuedAt  *NumericDate `json:"iat,omitempty"`
    ID        string       `json:"jti,omitempty"`
}
```

---

## 5. Signing Algorithms

### HS256 — HMAC SHA-256 (Symmetric)

- **One shared secret key** used to both sign and verify
- Same key on all servers → all servers can verify AND create tokens
- Simpler to set up, fast

```
Sign:   HMAC(header.payload, secret)  → signature
Verify: HMAC(header.payload, secret)  → compare
```

**Use when:** Single server or all servers are trusted.

### RS256 — RSA SHA-256 (Asymmetric)

- **Private key** signs the token (only auth server has it)
- **Public key** verifies the token (any service can verify)
- Other services cannot forge tokens

```
Sign:   RSA_sign(header.payload, private_key)  → signature
Verify: RSA_verify(header.payload, public_key) → compare
```

**Use when:** Microservices, third-party token issuers (OAuth2, OIDC).

### Algorithm Comparison

| | HS256 | RS256 | ES256 |
|--|-------|-------|-------|
| Key type | Shared secret | RSA key pair | ECDSA key pair |
| Key size | 32+ bytes | 2048+ bits | 256 bits |
| Speed | Fastest | Slower | Fast |
| Security | Good | Strong | Strong + compact |
| Use case | Single service | Microservices / OAuth | Modern APIs |

> **Security rule:** Always explicitly validate the algorithm in your key function. Never accept `"none"` algorithm.

---

## 6. Installation

```bash
go get github.com/golang-jwt/jwt/v5
```

```go
import "github.com/golang-jwt/jwt/v5"
```

> This is the maintained successor to the archived `dgrijalva/jwt-go`.

---

## 7. Generating a Token (HS256)

```go
package main

import (
    "fmt"
    "time"
    "github.com/golang-jwt/jwt/v5"
)

var secretKey = []byte("super-secret-key-store-in-env")

type Claims struct {
    UserID int    `json:"user_id"`
    Role   string `json:"role"`
    Email  string `json:"email"`
    jwt.RegisteredClaims
}

func generateToken(userID int, role, email string) (string, error) {
    now := time.Now()

    claims := Claims{
        UserID: userID,
        Role:   role,
        Email:  email,
        RegisteredClaims: jwt.RegisteredClaims{
            Subject:   fmt.Sprintf("%d", userID),
            Issuer:    "my-golang-app",
            Audience:  jwt.ClaimStrings{"web", "mobile"},
            IssuedAt:  jwt.NewNumericDate(now),
            NotBefore: jwt.NewNumericDate(now),
            ExpiresAt: jwt.NewNumericDate(now.Add(15 * time.Minute)),
            ID:        generateUUID(), // unique jti for revocation
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(secretKey)
}

func main() {
    tokenStr, err := generateToken(42, "admin", "amit@example.com")
    if err != nil {
        panic(err)
    }
    fmt.Println("Token:", tokenStr)
}
```

---

## 8. Parsing and Validating a Token

```go
func parseToken(tokenStr string) (*Claims, error) {
    claims := &Claims{}

    token, err := jwt.ParseWithClaims(
        tokenStr,
        claims,
        func(token *jwt.Token) (interface{}, error) {
            // IMPORTANT: always validate the algorithm
            if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
            }
            return secretKey, nil
        },
        jwt.WithValidMethods([]string{"HS256"}), // whitelist algorithms
        jwt.WithIssuer("my-golang-app"),          // validate issuer
        jwt.WithAudience("web"),                  // validate audience
    )

    if err != nil {
        return nil, err
    }

    if !token.Valid {
        return nil, fmt.Errorf("token is not valid")
    }

    return claims, nil
}
```

---

## 9. Handling JWT Errors

```go
import "errors"

token, err := jwt.ParseWithClaims(tokenStr, claims, keyFunc)
if err != nil {
    switch {
    case errors.Is(err, jwt.ErrTokenExpired):
        // 401 — token expired, client must refresh
        return echo.NewHTTPError(http.StatusUnauthorized, "token expired")

    case errors.Is(err, jwt.ErrTokenSignatureInvalid):
        // 401 — token was tampered with
        return echo.NewHTTPError(http.StatusUnauthorized, "invalid token signature")

    case errors.Is(err, jwt.ErrTokenMalformed):
        // 400 — not even a valid JWT format
        return echo.NewHTTPError(http.StatusBadRequest, "malformed token")

    case errors.Is(err, jwt.ErrTokenNotValidYet):
        // 401 — nbf claim is in the future
        return echo.NewHTTPError(http.StatusUnauthorized, "token not yet valid")

    case errors.Is(err, jwt.ErrTokenInvalidIssuer):
        return echo.NewHTTPError(http.StatusUnauthorized, "invalid issuer")

    case errors.Is(err, jwt.ErrTokenInvalidAudience):
        return echo.NewHTTPError(http.StatusUnauthorized, "invalid audience")

    default:
        return echo.NewHTTPError(http.StatusUnauthorized, "invalid token")
    }
}
```

### All JWT error types (golang-jwt/jwt v5)

| Error | Cause |
|-------|-------|
| `ErrTokenMalformed` | Not even a valid JWT format |
| `ErrTokenUnverifiable` | Cannot verify (missing/wrong key) |
| `ErrTokenSignatureInvalid` | Signature doesn't match |
| `ErrTokenExpired` | `exp` claim is in the past |
| `ErrTokenNotValidYet` | `nbf` claim is in the future |
| `ErrTokenUsedBeforeIssued` | `iat` is in the future |
| `ErrTokenRequiredClaimMissing` | A required claim is absent |
| `ErrTokenInvalidAudience` | `aud` claim mismatch |
| `ErrTokenInvalidIssuer` | `iss` claim mismatch |
| `ErrTokenInvalidSubject` | `sub` claim mismatch |
| `ErrTokenInvalidId` | `jti` validation failed |
| `ErrTokenInvalidClaims` | General claim validation failure |

---

## 10. MapClaims vs Custom Struct Claims

### MapClaims — quick and flexible

```go
token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
    "user_id": 42,
    "role":    "admin",
    "exp":     time.Now().Add(15 * time.Minute).Unix(),
    "iat":     time.Now().Unix(),
})
tokenStr, _ := token.SignedString(secretKey)

// Parse with MapClaims
token, _ = jwt.Parse(tokenStr, keyFunc)
claims := token.Claims.(jwt.MapClaims)
userID := int(claims["user_id"].(float64))  // ← type assertion needed, fragile
```

**Problem:** All values come out as `interface{}` — you must type-assert, which is error-prone.

### Custom struct — safer, recommended

```go
type Claims struct {
    UserID int    `json:"user_id"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

token, err := jwt.ParseWithClaims(tokenStr, &Claims{}, keyFunc)
claims := token.Claims.(*Claims)
userID := claims.UserID  // ← direct field access, type-safe
```

> **Always prefer a custom struct** — it is type-safe and self-documenting.

---

## 11. Generating Tokens with RS256

RS256 requires a private key to sign and a public key to verify.

### Generate RSA key pair (one-time)

```bash
# Generate private key
openssl genrsa -out private.pem 2048

# Extract public key
openssl rsa -in private.pem -pubout -out public.pem
```

### Sign with private key

```go
import (
    "os"
    "github.com/golang-jwt/jwt/v5"
)

func loadPrivateKey() (*rsa.PrivateKey, error) {
    data, _ := os.ReadFile("private.pem")
    return jwt.ParseRSAPrivateKeyFromPEM(data)
}

func generateRS256Token(userID int) (string, error) {
    privateKey, err := loadPrivateKey()
    if err != nil {
        return "", err
    }

    claims := Claims{
        UserID: userID,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(15 * time.Minute)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }

    token := jwt.NewWithClaims(jwt.SigningMethodRS256, claims)
    return token.SignedString(privateKey)
}
```

### Verify with public key

```go
func loadPublicKey() (*rsa.PublicKey, error) {
    data, _ := os.ReadFile("public.pem")
    return jwt.ParseRSAPublicKeyFromPEM(data)
}

func parseRS256Token(tokenStr string) (*Claims, error) {
    publicKey, err := loadPublicKey()
    if err != nil {
        return nil, err
    }

    claims := &Claims{}
    token, err := jwt.ParseWithClaims(tokenStr, claims,
        func(token *jwt.Token) (interface{}, error) {
            if _, ok := token.Method.(*jwt.SigningMethodRSA); !ok {
                return nil, fmt.Errorf("unexpected algorithm: %v", token.Header["alg"])
            }
            return publicKey, nil
        },
        jwt.WithValidMethods([]string{"RS256"}),
    )
    if err != nil {
        return nil, err
    }
    if !token.Valid {
        return nil, fmt.Errorf("invalid token")
    }
    return claims, nil
}
```

---

## 12. Access Token + Refresh Token Pattern

Short-lived access tokens + long-lived refresh tokens provide security without forcing frequent logins.

```
┌─────────────────────────────────────────────────────────────┐
│                    Authentication Flow                        │
│                                                               │
│  POST /login                                                  │
│  ─────────────────────────────────────────►                  │
│                               Create:                         │
│                               access_token  (15 min)          │
│                               refresh_token (7 days)          │
│  ◄─────────────────────────────────────────                  │
│  { access_token, refresh_token }                              │
│                                                               │
│  GET /api/profile                                             │
│  Authorization: Bearer <access_token>                         │
│  ─────────────────────────────────────────►                  │
│  Validate access_token → serve response    ◄─────────────── │
│                                                               │
│  [access_token expires]                                       │
│  POST /auth/refresh                                           │
│  { refresh_token: "..." }                                     │
│  ─────────────────────────────────────────►                  │
│  Validate refresh_token → issue new access_token             │
│  ◄─────────────────────────────────────────                  │
│  { access_token }                                             │
└─────────────────────────────────────────────────────────────┘
```

### Implementation

```go
const (
    accessTokenDuration  = 15 * time.Minute
    refreshTokenDuration = 7 * 24 * time.Hour
)

type TokenPair struct {
    AccessToken  string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
}

func generateTokenPair(userID int, role string) (TokenPair, error) {
    // Access token — short-lived, contains user info
    accessClaims := Claims{
        UserID: userID,
        Role:   role,
        RegisteredClaims: jwt.RegisteredClaims{
            Subject:   fmt.Sprintf("%d", userID),
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(accessTokenDuration)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }
    accessToken := jwt.NewWithClaims(jwt.SigningMethodHS256, accessClaims)
    accessStr, err := accessToken.SignedString(secretKey)
    if err != nil {
        return TokenPair{}, err
    }

    // Refresh token — long-lived, minimal claims
    refreshClaims := jwt.RegisteredClaims{
        Subject:   fmt.Sprintf("%d", userID),
        ExpiresAt: jwt.NewNumericDate(time.Now().Add(refreshTokenDuration)),
        IssuedAt:  jwt.NewNumericDate(time.Now()),
        ID:        uuid.New().String(), // jti — for revocation
    }
    refreshToken := jwt.NewWithClaims(jwt.SigningMethodHS256, refreshClaims)
    refreshStr, err := refreshToken.SignedString(refreshSecretKey) // different key!
    if err != nil {
        return TokenPair{}, err
    }

    return TokenPair{AccessToken: accessStr, RefreshToken: refreshStr}, nil
}
```

### Refresh token handler

```go
func refreshHandler(c echo.Context) error {
    var req struct {
        RefreshToken string `json:"refresh_token" validate:"required"`
    }
    if err := c.Bind(&req); err != nil {
        return echo.ErrBadRequest
    }

    // Parse the refresh token
    claims := &jwt.RegisteredClaims{}
    token, err := jwt.ParseWithClaims(req.RefreshToken, claims,
        func(t *jwt.Token) (interface{}, error) {
            return refreshSecretKey, nil
        },
        jwt.WithValidMethods([]string{"HS256"}),
    )
    if err != nil || !token.Valid {
        return echo.NewHTTPError(http.StatusUnauthorized, "invalid refresh token")
    }

    // TODO: check if jti is in the revocation list (Redis)
    userID, _ := strconv.Atoi(claims.Subject)

    // Issue new access token only
    newAccessClaims := Claims{
        UserID: userID,
        RegisteredClaims: jwt.RegisteredClaims{
            Subject:   claims.Subject,
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(accessTokenDuration)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }
    newToken := jwt.NewWithClaims(jwt.SigningMethodHS256, newAccessClaims)
    newTokenStr, err := newToken.SignedString(secretKey)
    if err != nil {
        return echo.ErrInternalServerError
    }

    return c.JSON(http.StatusOK, map[string]string{"access_token": newTokenStr})
}
```

---

## 13. JWT Middleware for Echo

### Option A — Using `echo-jwt` package (recommended)

```go
import echojwt "github.com/labstack/echo-jwt/v4"

e := echo.New()

// Apply to a group
api := e.Group("/api")
api.Use(echojwt.WithConfig(echojwt.Config{
    SigningKey:   secretKey,
    TokenLookup: "header:Authorization:Bearer ,cookie:access_token",
    NewClaimsFunc: func(c echo.Context) jwt.Claims {
        return new(Claims) // tells echo-jwt which struct to use
    },
    ErrorHandler: func(c echo.Context, err error) error {
        return echo.NewHTTPError(http.StatusUnauthorized, "invalid or missing token")
    },
}))
```

### Extracting claims from Echo context

```go
func getProfile(c echo.Context) error {
    // echo-jwt stores token at context key "user"
    token := c.Get("user").(*jwt.Token)
    claims := token.Claims.(*Claims)

    return c.JSON(http.StatusOK, map[string]interface{}{
        "user_id": claims.UserID,
        "role":    claims.Role,
        "email":   claims.Email,
    })
}
```

### Option B — Custom middleware (full control)

```go
func JWTMiddleware(next echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        authHeader := c.Request().Header.Get("Authorization")
        if authHeader == "" {
            return echo.ErrUnauthorized
        }

        // Strip "Bearer " prefix
        parts := strings.SplitN(authHeader, " ", 2)
        if len(parts) != 2 || parts[0] != "Bearer" {
            return echo.NewHTTPError(http.StatusUnauthorized, "invalid authorization format")
        }
        tokenStr := parts[1]

        claims, err := parseToken(tokenStr)
        if err != nil {
            switch {
            case errors.Is(err, jwt.ErrTokenExpired):
                return echo.NewHTTPError(http.StatusUnauthorized, "token expired")
            default:
                return echo.NewHTTPError(http.StatusUnauthorized, "invalid token")
            }
        }

        // Store claims in context for handlers to use
        c.Set("claims", claims)
        return next(c)
    }
}

// In handler:
func getProfile(c echo.Context) error {
    claims := c.Get("claims").(*Claims)
    return c.JSON(http.StatusOK, claims)
}
```

---

## 14. Token Revocation — The Stateless Problem

JWTs are valid until expiry. You cannot "delete" them like a session.

### Solutions

#### 1. Short access tokens (simplest)
Keep access tokens to 15 minutes. Worst case: attacker has 15 min.

#### 2. Redis blacklist (most common)

```go
import "github.com/redis/go-redis/v9"

var rdb *redis.Client

// On logout — blacklist the token's jti
func logout(c echo.Context) error {
    token := c.Get("user").(*jwt.Token)
    claims := token.Claims.(*Claims)

    jti := claims.ID
    ttl := time.Until(claims.ExpiresAt.Time)

    // Store jti in Redis until expiry
    err := rdb.Set(ctx, "blacklist:"+jti, "1", ttl).Err()
    if err != nil {
        return echo.ErrInternalServerError
    }
    return c.NoContent(http.StatusNoContent)
}

// In parseToken — check blacklist
func isBlacklisted(jti string) bool {
    result, err := rdb.Exists(ctx, "blacklist:"+jti).Result()
    return err == nil && result > 0
}
```

#### 3. Token rotation (refresh tokens only)

Each use of a refresh token issues a new one and invalidates the old one. Store used refresh token JTIs in Redis.

---

## 15. Complete Login / Logout Flow (Echo)

```go
var users = map[string]string{
    "amit@example.com": "hashedpassword123",
}

// POST /auth/login
func loginHandler(c echo.Context) error {
    var req struct {
        Email    string `json:"email"    validate:"required,email"`
        Password string `json:"password" validate:"required"`
    }
    if err := c.Bind(&req); err != nil {
        return echo.ErrBadRequest
    }
    if err := c.Validate(&req); err != nil {
        return echo.NewHTTPError(http.StatusUnprocessableEntity, err.Error())
    }

    // TODO: look up user in DB, verify password hash
    stored, ok := users[req.Email]
    if !ok || !checkPasswordHash(req.Password, stored) {
        return echo.ErrUnauthorized
    }

    pair, err := generateTokenPair(42, "user")
    if err != nil {
        return echo.ErrInternalServerError
    }

    return c.JSON(http.StatusOK, pair)
}

// POST /auth/refresh
func refreshHandler(c echo.Context) error { /* see section 12 */ }

// POST /auth/logout
func logoutHandler(c echo.Context) error { /* blacklist jti in Redis */ }

func main() {
    e := echo.New()

    // Public
    e.POST("/auth/login",   loginHandler)
    e.POST("/auth/refresh", refreshHandler)

    // Protected
    api := e.Group("/api")
    api.Use(JWTMiddleware)
    api.GET("/profile", getProfile)
    api.POST("/auth/logout", logoutHandler)

    e.Logger.Fatal(e.Start(":8080"))
}
```

---

## 16. Security Best Practices

| Practice | Why |
|----------|-----|
| **Store secret in env var, not code** | Prevent accidental key exposure in source control |
| **Use short access token TTL (15 min)** | Limit damage if token is stolen |
| **Always validate algorithm in keyfunc** | Prevent "alg: none" attack |
| **Use `WithValidMethods()`** | Whitelist algorithms — block algorithm confusion attacks |
| **Never put sensitive data in payload** | Payload is base64-decoded by anyone |
| **Transmit only over HTTPS** | Prevent token interception via network sniffing |
| **Use RS256 in microservices** | Auth service signs; other services only verify with public key |
| **Generate unique `jti`** | Enables per-token revocation |
| **Separate signing keys for access/refresh** | Compromise of one does not affect the other |
| **Use `exp` + `nbf` together** | Narrow valid window, prevent pre-use |
| **Implement refresh token rotation** | Detect token theft (if old token is used, revoke all) |
| **Avoid `localStorage` for tokens in browser** | Vulnerable to XSS; prefer `HttpOnly` cookies |

### Algorithm confusion attack (must know)

```go
// VULNERABLE — attacker can switch HS256→RS256 and use your public key as HMAC secret
token, _ := jwt.ParseWithClaims(tokenStr, claims, func(token *jwt.Token) (interface{}, error) {
    return secretKey, nil  // No algorithm check!
})

// SAFE — always check the method type
token, _ := jwt.ParseWithClaims(tokenStr, claims, func(token *jwt.Token) (interface{}, error) {
    if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
        return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
    }
    return secretKey, nil
})

// ALSO SAFE — use option
jwt.ParseWithClaims(tokenStr, claims, keyFunc,
    jwt.WithValidMethods([]string{"HS256"}),
)
```

---

## 17. Testing JWT Logic

```go
func TestGenerateAndParseToken(t *testing.T) {
    tokenStr, err := generateToken(42, "admin", "amit@example.com")
    assert.NoError(t, err)
    assert.NotEmpty(t, tokenStr)

    claims, err := parseToken(tokenStr)
    assert.NoError(t, err)
    assert.Equal(t, 42, claims.UserID)
    assert.Equal(t, "admin", claims.Role)
}

func TestExpiredToken(t *testing.T) {
    // Generate token already expired
    claims := Claims{
        UserID: 1,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(-1 * time.Hour)), // past
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    tokenStr, _ := token.SignedString(secretKey)

    _, err := parseToken(tokenStr)
    assert.ErrorIs(t, err, jwt.ErrTokenExpired)
}

func TestTamperedToken(t *testing.T) {
    tokenStr, _ := generateToken(1, "user", "u@u.com")

    // Tamper with the token (change last char)
    tampered := tokenStr[:len(tokenStr)-1] + "X"

    _, err := parseToken(tampered)
    assert.True(t, errors.Is(err, jwt.ErrTokenSignatureInvalid) ||
        errors.Is(err, jwt.ErrTokenMalformed))
}
```

---

## Key Takeaways

- JWT = **header.payload.signature** — all Base64URL encoded, dot-separated
- Payload is **readable** — never store passwords, secrets, or PII
- Signature **proves integrity** — if tampered, verification fails
- `exp`, `iat`, `sub`, `iss`, `jti` are standard registered claims — always set `exp`
- **HS256** = symmetric (one shared secret); **RS256** = asymmetric (private signs, public verifies)
- Always **validate the signing algorithm** in the keyfunc — prevents algorithm confusion attacks
- Use **custom struct claims** over `MapClaims` — type-safe, no fragile type assertions
- **Access token** (15 min) + **refresh token** (7 days) — best-practice pattern
- JWTs are **stateless by nature** — revocation requires a Redis blacklist or short TTLs
- Store secrets in **environment variables**, never hardcode in source
- Transmit tokens **only over HTTPS**; prefer `HttpOnly` cookies over `localStorage` in browsers
