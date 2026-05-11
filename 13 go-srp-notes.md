# Single Responsibility Principle — A Real World Example in Go

**Source:** https://www.youtube.com/watch?v=u7UhIiG2gNk&list=PL7g1jYj15RUP6aXNe66M91sB-ek9AC3w_&index=1  
**Playlist:** SOLID Principles in Go (new playlist)

---

## Prerequisites

- Go structs, interfaces, methods
- Packages and import system
- Basic dependency injection pattern

---

## Timeline

| Timestamp | Topic |
|-----------|-------|
| 0:00 | What is the Single Responsibility Principle? |
| 0:30 | "One reason to change" — what that means |
| 1:00 | The "and" test for spotting violations |
| 1:30 | Holding vs delegating responsibility |
| 2:00 | EmailService violation — real-world walkthrough |
| 3:30 | Refactored: Repository + Sender + Service |
| 5:00 | Function-level SRP — JWT extraction |
| 6:30 | Struct-level: Active Record, multi-purpose structs |
| 8:00 | Package-level SRP in Go |
| 9:00 | Benefits: testability, maintainability, team independence |

---

## 1. What Is the Single Responsibility Principle?

> "Each software module should have one and only one reason to change."
> — Robert C. Martin

A struct, function, or package should do **one thing**. When requirements in that one area change, only that component needs updating — nothing else.

SRP is about **cohesion**: keeping things that change together in the same place, and separating things that change for different reasons.

---

## 2. "One Reason to Change" — In Practice

"Reason to change" maps to a **stakeholder or concern**:

| Component | Whose requirement drives change? |
|-----------|----------------------------------|
| SMTP sender | Email infrastructure team |
| Database repository | Database / schema team |
| Business rules | Product / domain team |
| API response shape | Frontend / API consumers |

If a single struct changes for two different stakeholders, it has two responsibilities.

---

## 3. The "And" Test

> If describing what a struct does requires the word **"and"**, it violates SRP.

- `EmailService` saves emails to the database **and** sends them via SMTP ❌
- `User` holds user data **and** persists itself to the database ❌
- `Transaction` maps to the DB schema **and** serialises as JSON **and** validates API requests ❌

---

## 4. Holding vs Delegating Responsibility

This distinction is key in Go:

- **Holding**: If you delete the struct, the entire responsibility disappears with it.
- **Delegating**: If you delete the struct, the responsibility still exists in other components.

An orchestrator that *delegates* to a repository and a sender is fine — it has one job: coordinate the flow.

---

## 5. EmailService — The Classic SRP Violation

### Before: Two Responsibilities in One Struct

```go
// BAD: saves to DB AND sends via SMTP — two reasons to change
type EmailService struct {
    db           *gorm.DB
    smtpHost     string
    smtpPassword string
    smtpPort     int
}

func (s *EmailService) Send(from, to, subject, message string) error {
    // Responsibility 1: persist to database
    email := EmailGorm{From: from, To: to, Subject: subject, Message: message}
    if err := s.db.Create(&email).Error; err != nil {
        log.Println(err)
        return err
    }

    // Responsibility 2: transmit via SMTP
    auth := smtp.PlainAuth("", from, s.smtpPassword, s.smtpHost)
    server := fmt.Sprintf("%s:%d", s.smtpHost, s.smtpPort)
    if err := smtp.SendMail(server, auth, from, []string{to}, []byte(message)); err != nil {
        log.Println(err)
        return err
    }
    return nil
}
```

**Five consequences of this violation:**
1. A database schema change forces editing SMTP code
2. Switching from SMTP to Mailgun means touching DB persistence code
3. Every integration that needs email storage duplicates the DB logic
4. DB team and email team cannot work independently
5. Unit testing is practically impossible without spinning up a real DB and SMTP server

---

### After: Three Focused Components

**Step 1 — Repository: only persistence**

```go
type EmailRepository interface {
    Save(from, to, subject, message string) error
}

type EmailDBRepository struct {
    db *gorm.DB
}

func NewEmailRepository(db *gorm.DB) EmailRepository {
    return &EmailDBRepository{db: db}
}

func (r *EmailDBRepository) Save(from, to, subject, message string) error {
    email := EmailGorm{From: from, To: to, Subject: subject, Message: message}
    if err := r.db.Create(&email).Error; err != nil {
        log.Println(err)
        return err
    }
    return nil
}
```

*Reason to change: database schema or ORM changes.*

---

**Step 2 — Sender: only transmission**

```go
type EmailSender interface {
    Send(from, to, subject, message string) error
}

type EmailSMTPSender struct {
    smtpHost     string
    smtpPassword string
    smtpPort     int
}

func NewEmailSender(host, password string, port int) EmailSender {
    return &EmailSMTPSender{smtpHost: host, smtpPassword: password, smtpPort: port}
}

func (s *EmailSMTPSender) Send(from, to, subject, message string) error {
    auth := smtp.PlainAuth("", from, s.smtpPassword, s.smtpHost)
    server := fmt.Sprintf("%s:%d", s.smtpHost, s.smtpPort)
    if err := smtp.SendMail(server, auth, from, []string{to}, []byte(message)); err != nil {
        log.Println(err)
        return err
    }
    return nil
}
```

*Reason to change: SMTP provider or protocol changes.*

---

**Step 3 — Service: only orchestration (delegation, not holding)**

```go
type EmailService struct {
    repository EmailRepository
    sender     EmailSender
}

func NewEmailService(repo EmailRepository, sender EmailSender) *EmailService {
    return &EmailService{repository: repo, sender: sender}
}

func (s *EmailService) Send(from, to, subject, message string) error {
    if err := s.repository.Save(from, to, subject, message); err != nil {
        return err
    }
    return s.sender.Send(from, to, subject, message)
}
```

*Reason to change: orchestration logic (e.g., save before or after send, add retry).*

If you delete `EmailService`, persistence and transmission still exist — it **delegates**, not holds.

---

**Wiring in main:**

```go
func main() {
    db := connectDB()
    repo   := NewEmailRepository(db)
    sender := NewEmailSender("smtp.example.com", "secret", 587)
    svc    := NewEmailService(repo, sender)

    svc.Send("alice@example.com", "bob@example.com", "Hello", "World")
}
```

To swap to Mailgun later, only `NewEmailSender` changes — zero changes to `EmailService` or `EmailDBRepository`.

---

## 6. Function-Level SRP — JWT Extraction

### Before: One Function, Three Jobs

```go
// BAD: extracts header AND parses JWT AND reads claim
func extractUsername(header http.Header) string {
    raw := header.Get("Authorization")

    parser := &jwt.Parser{}
    token, _, err := parser.ParseUnverified(raw, jwt.MapClaims{})
    if err != nil {
        return ""
    }

    claims, ok := token.Claims.(jwt.MapClaims)
    if !ok {
        return ""
    }

    return claims["username"].(string)
}
```

### After: Each Function Has One Job

```go
func extractUsername(header http.Header) string {
    raw    := extractRawToken(header)
    claims := extractClaims(raw)
    if claims == nil {
        return ""
    }
    return claims["username"].(string)
}

// Reason to change: header key name changes
func extractRawToken(header http.Header) string {
    return header.Get("Authorization")
}

// Reason to change: JWT library or token format changes
func extractClaims(raw string) jwt.MapClaims {
    parser := &jwt.Parser{}
    token, _, err := parser.ParseUnverified(raw, jwt.MapClaims{})
    if err != nil {
        return nil
    }
    claims, ok := token.Claims.(jwt.MapClaims)
    if !ok {
        return nil
    }
    return claims
}
```

Each function is independently testable and has exactly one reason to change.

---

## 7. Struct-Level Violations

### Active Record Pattern — Business Logic + Persistence

```go
// BAD: IsAdult (business rule) AND Save (persistence) in same struct
type User struct {
    db        *gorm.DB
    Username  string
    Firstname string
    Lastname  string
    Birthday  time.Time
}

func (u User) IsAdult() bool {
    return u.Birthday.AddDate(18, 0, 0).Before(time.Now())
}

func (u *User) Save() error {
    return u.db.Exec(
        "INSERT INTO users (username, firstname, lastname, birthday) VALUES (?, ?, ?, ?)",
        u.Username, u.Firstname, u.Lastname, u.Birthday,
    ).Error
}
```

Fix — separate entity from repository:

```go
// Entity: pure domain data + business rules
type User struct {
    Username  string
    Firstname string
    Lastname  string
    Birthday  time.Time
}

func (u User) IsAdult() bool {
    return u.Birthday.AddDate(18, 0, 0).Before(time.Now())
}

// Repository: only persistence
type UserRepository struct{ db *gorm.DB }

func (r *UserRepository) Save(u *User) error {
    return r.db.Exec(
        "INSERT INTO users (username, firstname, lastname, birthday) VALUES (?, ?, ?, ?)",
        u.Username, u.Firstname, u.Lastname, u.Birthday,
    ).Error
}
```

---

### Multi-Purpose Struct — Three Reasons to Change

```go
// BAD: DB mapping AND JSON serialisation AND request validation — 3 responsibilities
type Transaction struct {
    gorm.Model
    Amount     int       `gorm:"column:amount"      json:"amount"      validate:"required"`
    CurrencyID int       `gorm:"column:currency_id" json:"currency_id" validate:"required"`
    Time       time.Time `gorm:"column:time"        json:"time"        validate:"required"`
}
```

Fix — three separate types:

```go
// DB model: only database mapping
type TransactionDB struct {
    gorm.Model
    Amount     int
    CurrencyID int
    Time       time.Time
}

// API response: only JSON shape
type TransactionResponse struct {
    Amount     int       `json:"amount"`
    CurrencyID int       `json:"currency_id"`
    Time       time.Time `json:"time"`
}

// Request body: only validation
type CreateTransactionRequest struct {
    Amount     int       `json:"amount"      validate:"required,gt=0"`
    CurrencyID int       `json:"currency_id" validate:"required"`
    Time       time.Time `json:"time"        validate:"required"`
}
```

---

### Trade — Validator and Repository Separation

```go
// Entity
type Trade struct {
    TradeID  int
    Symbol   string
    Quantity float64
    Price    float64
}

// Repository: only persistence — reason to change: DB schema
type TradeRepository struct{ db *sql.DB }

func (r *TradeRepository) Save(t *Trade) error {
    _, err := r.db.Exec(
        "INSERT INTO trades (trade_id, symbol, quantity, price) VALUES (?, ?, ?, ?)",
        t.TradeID, t.Symbol, t.Quantity, t.Price,
    )
    return err
}

// Validator: only business rules — reason to change: domain rules
type TradeValidator struct{}

func (v *TradeValidator) Validate(t *Trade) error {
    if t.Quantity <= 0 {
        return errors.New("quantity must be greater than zero")
    }
    if t.Price <= 0 {
        return errors.New("price must be greater than zero")
    }
    return nil
}
```

---

## 8. Order Processing — Chef/Waiter Analogy

Think of SRP like a restaurant: the **chef** prepares food, the **waiter** serves it. One person doing both causes delays and errors.

```go
// BAD: payment AND printing in one method
type Order struct{ Amount float64 }

func (o *Order) ProcessOrder() {
    // charges card
    fmt.Printf("Charging $%.2f\n", o.Amount)
    // prints receipt
    fmt.Printf("Receipt: $%.2f paid\n", o.Amount)
}

// GOOD: separated concerns
type PaymentProcessor struct{}

func (p *PaymentProcessor) Charge(amount float64) error {
    fmt.Printf("Charging $%.2f\n", amount)
    return nil
}

type ReceiptPrinter struct{}

func (r *ReceiptPrinter) Print(amount float64) {
    fmt.Printf("Receipt: $%.2f paid\n", amount)
}

type OrderService struct {
    payment *PaymentProcessor
    printer *ReceiptPrinter
}

func (s *OrderService) ProcessOrder(amount float64) error {
    if err := s.payment.Charge(amount); err != nil {
        return err
    }
    s.printer.Print(amount)
    return nil
}
```

---

## 9. Package-Level SRP in Go

In Go, **packages** are also units of responsibility. Dave Cheney's framing:

> "A well-designed package has a single purpose and a name that reflects it."

**Signs of package-level SRP violation:**
- Package named `util`, `common`, `helpers`, `misc`, `shared`
- Package that imports many unrelated external libraries
- Functions in the package have nothing to do with each other

**Good package names** announce exactly one responsibility:
- `encoding/json` — JSON marshalling only
- `net/http` — HTTP protocol only
- `database/sql` — SQL database abstraction only

**Anti-pattern:**
```
// Bad: one dumping-ground package
package utils

func FormatDate(t time.Time) string { ... }
func HashPassword(s string) string { ... }
func SendEmail(to, body string) error { ... }
func ValidatePhone(s string) bool { ... }
```

**Fix:** split by cohesion:
```
package dateutil
package auth
package notification
package validation
```

---

## 10. Testing Benefit

SRP makes unit tests trivial — each component can be tested in isolation.

```go
func TestEmailDBRepository_Save(t *testing.T) {
    db := setupTestDB(t) // only DB, no SMTP
    repo := NewEmailRepository(db)
    err := repo.Save("a@a.com", "b@b.com", "hi", "hello")
    assert.NoError(t, err)
}

func TestEmailSMTPSender_Send(t *testing.T) {
    // use a mock SMTP server, no DB involved
    sender := NewEmailSender("localhost", "", 1025)
    err := sender.Send("a@a.com", "b@b.com", "hi", "hello")
    assert.NoError(t, err)
}

func TestEmailService_Send(t *testing.T) {
    // mock both dependencies — tests orchestration logic only
    repo   := &mockRepo{}
    sender := &mockSender{}
    svc    := NewEmailService(repo, sender)
    err    := svc.Send("a@a.com", "b@b.com", "hi", "hello")
    assert.NoError(t, err)
    assert.True(t, repo.saved)
    assert.True(t, sender.sent)
}
```

---

## 11. Signs of SRP Violation — Checklist

| Signal | Example |
|--------|---------|
| Description uses "and" | "saves **and** sends email" |
| Struct has both db and smtp fields | `{db *gorm.DB, smtpHost string}` |
| Method does I/O + business logic | `Validate()` + `Save()` in one method |
| Struct has gorm + json + validate tags | `Transaction` example above |
| Package named `util` / `common` | Unrelated functions coexist |
| Changing DB schema breaks email code | Tight coupling |
| Can't test without spinning up infra | No abstraction between concerns |

---

## 12. Quick Reference

```
SRP: one reason to change per struct / function / package

Diagnostic:
  - Does description need "and"? → split it
  - Who drives change? → one stakeholder per component
  - Can you delete it without losing a responsibility? → you're delegating (good)

Pattern:
  Entity       → pure domain data + business rules (no DB, no I/O)
  Repository   → only persistence
  Sender/Client→ only external communication
  Validator    → only business rule checking
  Service      → only orchestration (delegates, never holds)

Package naming:
  Bad:  util, common, helpers
  Good: auth, notification, payment, validation
```

---

## Further Resources

- [Dave Cheney — SOLID Go Design](https://dave.cheney.net/2016/08/20/solid-go-design)
- [Ompluscator — Practical SOLID: SRP in Go](https://www.ompluscator.com/article/golang/practical-solid-single-responsibility/)
- [Relia Software — SRP in Golang](https://reliasoftware.com/blog/single-responsibility-principle-in-golang)
- [Robert C. Martin — Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
