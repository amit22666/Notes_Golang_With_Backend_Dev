# Abstract Factory Pattern — Explained For Beginners (Go)

**Source:** https://www.youtube.com/watch?v=F9tQ46YkQLU&list=PL7g1jYj15RUO-crQOgDV0_dVp2OaoSgye&index=2  
**Playlist:** Design Patterns (Visualized)  
**Category:** Creational Patterns

---

## Prerequisites

- Go interfaces, structs, embedding
- Factory Method pattern (previous video)
- Basic dependency injection

---

## Timeline

| Timestamp | Topic |
|-----------|-------|
| 0:00 | Problem: families of related objects must stay compatible |
| 0:30 | Abstract Factory — definition and intent |
| 1:00 | Five participants: AbsFactory, ConcreteFactory, AbsProduct, ConcreteProduct, Client |
| 1:40 | Sports equipment example — Adidas/Nike walkthrough |
| 3:30 | GUI cross-platform example — Windows/Mac |
| 5:00 | Registry-based factory variant |
| 5:30 | Functional factory variant |
| 6:00 | Abstract Factory vs Factory Method vs Builder |
| 6:30 | When to use / when to avoid |

---

## 1. The Problem — Families Must Stay Compatible

Without Abstract Factory, creation logic scatters and products can be mixed incorrectly:

```go
// BAD: client picks concrete types directly — can accidentally mix families
func buildUI(os string) {
    var btn Button
    var menu Menu

    if os == "windows" {
        btn = &WindowsButton{}
    } else {
        btn = &MacButton{}
    }

    if os == "windows" {
        menu = &WindowsMenu{} // OK
    } else {
        menu = &MacMenu{} // OK — but what stops this?
    }

    // Nothing prevents: WindowsButton + MacMenu — visual inconsistency!
}
```

**The core question:** How do we guarantee that a `WindowsButton` only ever appears alongside a `WindowsMenu` and `WindowsCheckbox`?

Abstract Factory enforces family consistency by making the factory — not the client — responsible for creating every product in a family.

---

## 2. What Is the Abstract Factory Pattern?

> "Provides an interface for creating **families of related objects** without specifying their concrete classes."
> — Gang of Four

**Key distinction from Factory Method:**

| | Factory Method | Abstract Factory |
|--|--|--|
| Creates | **One** product type | **Multiple** related product types |
| Focus | Single object variation | Whole family consistency |
| Structure | One `Create()` method | Multiple `CreateX()` methods |

**Analogy — Furniture store:**  
A `ModernFactory` produces `ModernChair` + `ModernTable` + `ModernSofa`.  
A `VictorianFactory` produces `VictorianChair` + `VictorianTable` + `VictorianSofa`.  
You pick *one factory* and get a *coordinated set*. You can never accidentally mix Modern and Victorian.

---

## 3. Five Participants

```
┌─────────────────────────────────────────────────────────┐
│                   Abstract Factory                       │
├──────────────────┬──────────────────────────────────────┤
│ Participant      │ Role                                  │
├──────────────────┼──────────────────────────────────────┤
│ AbstractFactory  │ Interface: declares Create methods    │
│ ConcreteFactory  │ Implements AbstractFactory per family │
│ AbstractProduct  │ Interface: declares product behaviour │
│ ConcreteProduct  │ Implements AbstractProduct per family │
│ Client           │ Uses only AbstractFactory + products  │
└──────────────────┴──────────────────────────────────────┘
```

---

## 4. Full Example — Sports Equipment (Adidas / Nike)

This is the canonical Go implementation from refactoring.guru.

### iSportsFactory.go — Abstract Factory

```go
type ISportsFactory interface {
    makeShoe()  IShoe
    makeShirt() IShirt
}

// Client-facing factory function — picks the concrete factory by brand
func GetSportsFactory(brand string) (ISportsFactory, error) {
    switch brand {
    case "adidas":
        return &Adidas{}, nil
    case "nike":
        return &Nike{}, nil
    default:
        return nil, fmt.Errorf("unknown brand: %s", brand)
    }
}
```

### iShoe.go — Abstract Product 1

```go
type IShoe interface {
    setLogo(logo string)
    setSize(size int)
    getLogo() string
    getSize() int
}

// Base struct — shared behaviour via embedding
type Shoe struct {
    logo string
    size int
}

func (s *Shoe) setLogo(logo string) { s.logo = logo }
func (s *Shoe) getLogo() string     { return s.logo }
func (s *Shoe) setSize(size int)    { s.size = size }
func (s *Shoe) getSize() int        { return s.size }
```

### iShirt.go — Abstract Product 2

```go
type IShirt interface {
    setLogo(logo string)
    setSize(size int)
    getLogo() string
    getSize() int
}

type Shirt struct {
    logo string
    size int
}

func (s *Shirt) setLogo(logo string) { s.logo = logo }
func (s *Shirt) getLogo() string     { return s.logo }
func (s *Shirt) setSize(size int)    { s.size = size }
func (s *Shirt) getSize() int        { return s.size }
```

### Concrete Products — Adidas family

```go
// adidasShoe.go
type AdidasShoe struct{ Shoe } // embedding provides all IShoe methods

// adidasShirt.go
type AdidasShirt struct{ Shirt }
```

### Concrete Products — Nike family

```go
// nikeShoe.go
type NikeShoe struct{ Shoe }

// nikeShirt.go
type NikeShirt struct{ Shirt }
```

### adidas.go — Concrete Factory (Adidas family)

```go
type Adidas struct{}

func (a *Adidas) makeShoe() IShoe {
    return &AdidasShoe{
        Shoe: Shoe{logo: "adidas", size: 14},
    }
}

func (a *Adidas) makeShirt() IShirt {
    return &AdidasShirt{
        Shirt: Shirt{logo: "adidas", size: 14},
    }
}
```

### nike.go — Concrete Factory (Nike family)

```go
type Nike struct{}

func (n *Nike) makeShoe() IShoe {
    return &NikeShoe{
        Shoe: Shoe{logo: "nike", size: 14},
    }
}

func (n *Nike) makeShirt() IShirt {
    return &NikeShirt{
        Shirt: Shirt{logo: "nike", size: 14},
    }
}
```

### main.go — Client (works only with interfaces)

```go
func main() {
    adidasFactory, _ := GetSportsFactory("adidas")
    nikeFactory, _   := GetSportsFactory("nike")

    // Each factory guarantees brand-consistent products
    adidasShoe  := adidasFactory.makeShoe()
    adidasShirt := adidasFactory.makeShirt()
    nikeShoe    := nikeFactory.makeShoe()
    nikeShirt   := nikeFactory.makeShirt()

    printShoe(adidasShoe)
    printShirt(adidasShirt)
    printShoe(nikeShoe)
    printShirt(nikeShirt)
}

func printShoe(s IShoe) {
    fmt.Printf("Shoe  — logo: %s, size: %d\n", s.getLogo(), s.getSize())
}

func printShirt(s IShirt) {
    fmt.Printf("Shirt — logo: %s, size: %d\n", s.getLogo(), s.getSize())
}
// Output:
// Shoe  — logo: adidas, size: 14
// Shirt — logo: adidas, size: 14
// Shoe  — logo: nike,   size: 14
// Shirt — logo: nike,   size: 14
```

To add a **Puma** family: create `Puma` factory + `PumaShoe` + `PumaShirt` — zero changes to client code or `GetSportsFactory`'s callers.

---

## 5. GUI Cross-Platform Example (Windows / macOS)

The textbook motivation: a UI toolkit must look native on every OS.

### Abstract Products

```go
type Button interface {
    Paint()
}

type Menu interface {
    Show()
}

type Checkbox interface {
    Check()
}
```

### Abstract Factory

```go
type UIFactory interface {
    CreateButton()   Button
    CreateMenu()     Menu
    CreateCheckbox() Checkbox
}
```

### Windows Family

```go
type WindowsButton struct{}
func (WindowsButton) Paint() { fmt.Println("Rendering Windows button") }

type WindowsMenu struct{}
func (WindowsMenu) Show() { fmt.Println("Showing Windows menu") }

type WindowsCheckbox struct{}
func (WindowsCheckbox) Check() { fmt.Println("Checking Windows checkbox") }

type WindowsFactory struct{}
func (WindowsFactory) CreateButton()   Button   { return WindowsButton{} }
func (WindowsFactory) CreateMenu()     Menu     { return WindowsMenu{} }
func (WindowsFactory) CreateCheckbox() Checkbox { return WindowsCheckbox{} }
```

### macOS Family

```go
type MacButton struct{}
func (MacButton) Paint() { fmt.Println("Rendering Mac button") }

type MacMenu struct{}
func (MacMenu) Show() { fmt.Println("Showing Mac menu") }

type MacCheckbox struct{}
func (MacCheckbox) Check() { fmt.Println("Checking Mac checkbox") }

type MacFactory struct{}
func (MacFactory) CreateButton()   Button   { return MacButton{} }
func (MacFactory) CreateMenu()     Menu     { return MacMenu{} }
func (MacFactory) CreateCheckbox() Checkbox { return MacCheckbox{} }
```

### Client — platform-agnostic

```go
func renderUI(factory UIFactory) {
    btn      := factory.CreateButton()
    menu     := factory.CreateMenu()
    checkbox := factory.CreateCheckbox()
    btn.Paint()
    menu.Show()
    checkbox.Check()
}

func main() {
    var factory UIFactory
    switch runtime.GOOS {
    case "windows":
        factory = WindowsFactory{}
    default:
        factory = MacFactory{}
    }
    renderUI(factory) // correct family automatically
}
```

Adding a `LinuxFactory` requires only new concrete types — `renderUI` never changes.

---

## 6. Notification × Platform Example

A factory per **platform** (iOS / Android), each creating the full set of notification types.

```go
// Abstract products
type SMSNotification interface {
    SendSMS(to, msg string) error
}

type PushNotification interface {
    SendPush(deviceToken, msg string) error
}

// Abstract factory — one factory per platform
type NotificationFactory interface {
    CreateSMS()  SMSNotification
    CreatePush() PushNotification
}

// iOS family
type iOSSMS struct{}
func (i *iOSSMS) SendSMS(to, msg string) error {
    fmt.Printf("[iOS SMS] %s → %s\n", msg, to); return nil
}

type iOSPush struct{}
func (i *iOSPush) SendPush(token, msg string) error {
    fmt.Printf("[iOS Push] %s → %s\n", msg, token); return nil
}

type iOSFactory struct{}
func (f *iOSFactory) CreateSMS()  SMSNotification  { return &iOSSMS{} }
func (f *iOSFactory) CreatePush() PushNotification { return &iOSPush{} }

// Android family
type AndroidSMS struct{}
func (a *AndroidSMS) SendSMS(to, msg string) error {
    fmt.Printf("[Android SMS] %s → %s\n", msg, to); return nil
}

type AndroidPush struct{}
func (a *AndroidPush) SendPush(token, msg string) error {
    fmt.Printf("[Android Push] %s → %s\n", msg, token); return nil
}

type AndroidFactory struct{}
func (f *AndroidFactory) CreateSMS()  SMSNotification  { return &AndroidSMS{} }
func (f *AndroidFactory) CreatePush() PushNotification { return &AndroidPush{} }

// Factory selector
func GetNotificationFactory(platform string) (NotificationFactory, error) {
    switch platform {
    case "ios":
        return &iOSFactory{}, nil
    case "android":
        return &AndroidFactory{}, nil
    default:
        return nil, fmt.Errorf("unknown platform: %s", platform)
    }
}
```

---

## 7. Registry-Based Factory Variant

Register factories at startup; look them up by name at runtime — avoids the switch statement.

```go
var factoryRegistry = map[string]UIFactory{}

func RegisterFactory(name string, f UIFactory) {
    factoryRegistry[name] = f
}

func GetFactory(name string) (UIFactory, error) {
    f, ok := factoryRegistry[name]
    if !ok {
        return nil, fmt.Errorf("no factory registered for %q", name)
    }
    return f, nil
}

func init() {
    RegisterFactory("windows", WindowsFactory{})
    RegisterFactory("mac",     MacFactory{})
}

// Third-party packages can call RegisterFactory() to plug in new families
// without modifying any existing code
```

---

## 8. Functional Factory Variant

Replace the factory interface with a function type — lighter weight.

```go
// Factory is just a function that returns a coordinated set of products
type UIFactory func() (Button, Menu)

var WindowsUIFactory UIFactory = func() (Button, Menu) {
    return WindowsButton{}, WindowsMenu{}
}

var MacUIFactory UIFactory = func() (Button, Menu) {
    return MacButton{}, MacMenu{}
}

func renderUI(factory UIFactory) {
    btn, menu := factory()
    btn.Paint()
    menu.Show()
}

renderUI(WindowsUIFactory)
renderUI(MacUIFactory)
```

---

## 9. Testing with Abstract Factory

Inject a mock factory in tests — no real GUI, network, or platform code runs.

```go
// Mock products
type mockButton   struct{ painted bool }
func (m *mockButton)   Paint() { m.painted = true }

type mockMenu     struct{ shown bool }
func (m *mockMenu)     Show()  { m.shown = true }

type mockCheckbox struct{ checked bool }
func (m *mockCheckbox) Check() { m.checked = true }

// Mock factory
type mockUIFactory struct {
    btn *mockButton
    men *mockMenu
    chk *mockCheckbox
}
func (f *mockUIFactory) CreateButton()   Button   { return f.btn }
func (f *mockUIFactory) CreateMenu()     Menu     { return f.men }
func (f *mockUIFactory) CreateCheckbox() Checkbox { return f.chk }

func TestRenderUI(t *testing.T) {
    mock := &mockUIFactory{
        btn: &mockButton{},
        men: &mockMenu{},
        chk: &mockCheckbox{},
    }
    renderUI(mock)

    if !mock.btn.painted { t.Error("button not painted") }
    if !mock.men.shown   { t.Error("menu not shown") }
}
```

---

## 10. Abstract Factory vs Factory Method vs Builder

| | Factory Method | Abstract Factory | Builder |
|--|--|--|--|
| **Creates** | One product, one type | Multiple related products | One complex product |
| **Method count** | One `Create()` | Many `CreateX()` | Many `SetX()` + `Build()` |
| **Goal** | Vary a single product | Ensure family consistency | Step-by-step construction |
| **Switching** | Change the creator | Change the whole factory | Change the builder |
| **Example** | `NewLogger("json")` | `WindowsFactory{}` | `QueryBuilder` |

---

## 11. When to Use

**Use Abstract Factory when:**
- You need families of objects that **must work together** (e.g., OS-specific UI components)
- You want to swap entire families at runtime with a single change
- You want client code completely decoupled from concrete types
- You'll have multiple parallel product families (more than one "brand")

**Avoid Abstract Factory when:**
- You only have a **single** product family (use Factory Method instead)
- You're unlikely to add more families (premature abstraction)
- The number of product types per family is small and stable
- The added boilerplate outweighs the benefit

---

## 12. Pros and Cons

**Pros:**
- **Consistency** — impossible to mix products from different families
- **Open/Closed** — add new families without modifying client code
- **Encapsulation** — concrete types are hidden behind interfaces
- **Testability** — inject mock factories in tests

**Cons:**
- **Boilerplate** — many interfaces and structs for every product × family combination
- **Rigidity** — adding a new *product type* (e.g., `Checkbox`) to the abstract factory requires updating *every* concrete factory
- **Over-engineering** — overkill if you only have one or two implementations

---

## 13. Quick Reference

```go
// 1. Abstract products
type ProductA interface { DoA() }
type ProductB interface { DoB() }

// 2. Abstract factory
type AbstractFactory interface {
    CreateA() ProductA
    CreateB() ProductB
}

// 3. Concrete products — Family 1
type ConcreteA1 struct{}; func (ConcreteA1) DoA() { fmt.Println("A1") }
type ConcreteB1 struct{}; func (ConcreteB1) DoB() { fmt.Println("B1") }

// 4. Concrete factory — Family 1
type Factory1 struct{}
func (Factory1) CreateA() ProductA { return ConcreteA1{} }
func (Factory1) CreateB() ProductB { return ConcreteB1{} }

// 5. Concrete products — Family 2
type ConcreteA2 struct{}; func (ConcreteA2) DoA() { fmt.Println("A2") }
type ConcreteB2 struct{}; func (ConcreteB2) DoB() { fmt.Println("B2") }

// 6. Concrete factory — Family 2
type Factory2 struct{}
func (Factory2) CreateA() ProductA { return ConcreteA2{} }
func (Factory2) CreateB() ProductB { return ConcreteB2{} }

// 7. Client — never mentions concrete types
func client(f AbstractFactory) {
    a := f.CreateA()
    b := f.CreateB()
    a.DoA()
    b.DoB()
}

// 8. Wire up
client(Factory1{}) // A1, B1 — consistent family
client(Factory2{}) // A2, B2 — consistent family
```

---

## 14. File Layout Convention

```
/sportsfactory
  isportsfactory.go   ← AbstractFactory interface + GetSportsFactory()
  adidas.go           ← Adidas ConcreteFactory
  nike.go             ← Nike ConcreteFactory
  ishoe.go            ← IShoe AbstractProduct + Shoe base
  ishirt.go           ← IShirt AbstractProduct + Shirt base
  adidasshoe.go       ← AdidasShoe ConcreteProduct
  adidasshirt.go      ← AdidasShirt ConcreteProduct
  nikeshoe.go         ← NikeShoe ConcreteProduct
  nikeshirt.go        ← NikeShirt ConcreteProduct
  main.go             ← Client
```

One file per type keeps the pattern navigable as families grow.

---

## Further Resources

- [Refactoring.guru — Abstract Factory in Go](https://refactoring.guru/design-patterns/abstract-factory/go/example)
- [Refactoring.guru — Abstract Factory vs Factory Method](https://refactoring.guru/design-patterns/factory-comparison)
- [Refactoring.guru — All Go Design Patterns](https://refactoring.guru/design-patterns/go)
- [DEV — Abstract Factory using GoLang](https://dev.to/jayaprasanna_roddam/abstract-factory-method-3djd)
- [Matthias Bruns — Golang Abstract Factory Pattern](https://blog.matthiasbruns.com/golang-abstract-factory-pattern)
