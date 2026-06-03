# Low Level Design & Design Patterns

## Progress Tracker
- [x] APIE — Four Pillars of OOP
- [x] SOLID Principles
- [x] Creational Patterns
  - [x] Singleton
  - [x] Factory Method
  - [x] Builder
  - [x] Prototype
- [ ] Structural Patterns
  - [ ] Decorator
  - [ ] Adapter
  - [ ] Facade
  - [ ] Proxy
- [ ] Behavioral Patterns
  - [ ] Observer
  - [ ] Strategy
  - [ ] Command
  - [ ] Iterator
  - [ ] Template Method
- [ ] LLD Case Studies
  - [ ] Parking Lot
  - [ ] Library System
  - [ ] Elevator
  - [ ] Chess
  - [ ] Snake & Ladder

---

## APIE — Four Pillars of OOP

> SOLID is built on top of these. These are the fundamental capabilities of OOP languages.

| Pillar | One line |
|---|---|
| **Abstraction** | Show only what's necessary, hide complexity underneath |
| **Polymorphism** | Same interface, different behaviour depending on the object |
| **Inheritance** | A class acquires properties and behaviour of another class |
| **Encapsulation** | Bundle data + methods, control access to internal state |

### Abstraction
Hide complexity behind a clean interface. Callers don't need to know *how*, only *what*.
```python
class Car:
    def start(self):
        self._ignite_fuel_system()
        self._start_engine()
        self._engage_transmission()
        print("Car started")
    def _ignite_fuel_system(self): pass
    def _start_engine(self): pass
    def _engage_transmission(self): pass
```

### Polymorphism
Same call, different behaviour. Two forms in Python:
- **Subtype polymorphism** — via inheritance and method overriding
- **Duck typing** — if it has the method, it works, no inheritance needed

```python
class Shape(ABC):
    @abstractmethod
    def area(self) -> float: pass

class Circle(Shape):
    def area(self): return 3.14 * self.r ** 2

class Rectangle(Shape):
    def area(self): return self.w * self.h

for shape in [Circle(5), Rectangle(4, 6)]:
    print(shape.area())   # same call, different result
```

### Inheritance
Use for **is-a** relationships with shared behaviour. Prefer **composition** for code reuse.
```python
# Prefer composition (HAS-A) over deep inheritance (IS-A)
class Engine:
    def start(self): print("Engine started")

class Car:
    def __init__(self):
        self.engine = Engine()   # Car HAS-A Engine
```

### Encapsulation
Control internal state through methods, not direct access.

| Syntax | Level | Meaning |
|---|---|---|
| `balance` | public | anyone can access |
| `_balance` | protected | internal use by convention |
| `__balance` | private | name-mangled, hard to access externally |

---

## SOLID Principles

### S — Single Responsibility
> One class, one reason to change.

**Mental test:** ask "why would this class change?" — more than one answer = violation.

```python
# Violation — three reasons to change
class User:
    def save_to_db(self): ...
    def send_welcome_email(self): ...
    def generate_report(self): ...

# Fixed — one reason each
class UserRepository:
    def save(self, user: User): ...

class EmailService:
    def send_welcome(self, user: User): ...

class ReportService:
    def generate(self, user: User): ...
```

### O — Open/Closed
> Open for extension, closed for modification.

**Mental test:** when a new requirement comes in, are you editing an existing class or adding a new one?

```python
class PaymentMethod(ABC):
    @abstractmethod
    def pay(self, amount: float): pass

class CreditCard(PaymentMethod):
    def pay(self, amount): print(f"Credit card: {amount}")

class UPI(PaymentMethod):           # new type — nothing else touched
    def pay(self, amount): print(f"UPI: {amount}")

class PaymentProcessor:
    def process(self, method: PaymentMethod, amount: float):
        method.pay(amount)          # never changes
```

### L — Liskov Substitution
> Subclass must be fully substitutable for its parent without breaking behaviour.

**Mental test:** swap every subclass in wherever the parent is used — does anything break?

```python
# Violation — Penguin breaks Bird's fly() contract
class Bird:
    def fly(self): print("Flying")

class Penguin(Bird):
    def fly(self): raise Exception("Can't fly")  # 💥

# Fixed — restructure hierarchy around behaviour
class FlyingBird(Bird):
    @abstractmethod
    def fly(self): pass

class FlightlessBird(Bird):
    @abstractmethod
    def move(self): pass
```

### I — Interface Segregation
> No class should be forced to implement methods it doesn't use.

**Mental test:** does any class implement a method as `raise NotImplementedError`? Fat interface — split it.

```python
# Violation
class Worker(ABC):
    def work(self): pass
    def eat(self): pass    # Robot doesn't eat
    def sleep(self): pass  # Robot doesn't sleep

# Fixed — focused interfaces
class Workable(ABC):
    def work(self): pass

class Eatable(ABC):
    def eat(self): pass

class Robot(Workable):           # only what it actually does
    def work(self): print("Robot working")
```

### D — Dependency Inversion
> High-level modules should not depend on low-level modules. Both should depend on abstractions.

**Mental test:** can you swap the dependency (e.g. for a mock in tests) without touching the high-level class?

```python
class Database(ABC):
    @abstractmethod
    def save(self, data: str): pass

class MySQLDatabase(Database):
    def save(self, data): print(f"MySQL: {data}")

class MockDatabase(Database):
    def save(self, data): print(f"[MOCK]: {data}")

class UserService:
    def __init__(self, db: Database):   # injected, not created
        self.db = db

# Production
UserService(MySQLDatabase())

# Testing — no real DB needed
UserService(MockDatabase())
```

### LSP vs ISP — the key distinction
| | LSP | ISP |
|---|---|---|
| Problem | Subclass *behaviour* breaks substitution | Interface forces dead methods |
| Root cause | Wrong inheritance hierarchy | Interface is too fat |
| Fix | Restructure hierarchy | Split interface |
| Question | "Can I substitute safely?" | "Does this class need all these methods?" |

### How APIE enables SOLID
- **Abstraction** → enables OCP, DIP
- **Polymorphism** → enables OCP, LSP, ISP
- **Inheritance** → enables LSP (and causes its violations)
- **Encapsulation** → enables SRP

---

## Creational Patterns

> How objects are created. Prevent scattered `SomeClass()` calls all over the codebase.

| Pattern | Problem | Mechanic |
|---|---|---|
| Singleton | Only one instance should exist | Intercept `__new__`, return cached instance |
| Factory | Caller shouldn't know which class to create | Delegate instantiation to factory |
| Builder | Complex object with many optional params | Method chaining + single `.build()` |
| Prototype | Expensive construction, many similar objects | `deepcopy` an existing instance |

---

### Singleton

```python
class Config:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.settings = {}
            print("Config loaded")   # runs exactly once
        return cls._instance

# Pythonic alternative — decorator
def singleton(cls):
    instances = {}
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance

@singleton
class Logger:
    def __init__(self):
        self.logs = []
```

**Warning:** Singleton + hidden dependencies = untestable code. Always combine with DI.

```python
# Wrong — hidden dependency
class UserService:
    def get_user(self, id):
        db = DatabaseConnection()   # untestable

# Right — Singleton + DI
class UserService:
    def __init__(self, db: DatabaseConnection):
        self.db = db
```

**Use when:** truly one instance makes sense (logger, config, connection pool) + combined with DI.
**Avoid when:** you just want global access to avoid passing things around.

---

### Factory Method

```python
class Report(ABC):
    @abstractmethod
    def generate(self): pass

class PDFReport(Report):
    def generate(self): print("PDF report")

class ExcelReport(Report):
    def generate(self): print("Excel report")

# Simple Factory — 80% case
class ReportFactory:
    @staticmethod
    def create(report_type: str) -> Report:
        factories = {
            "pdf":   PDFReport,
            "excel": ExcelReport,
        }
        cls = factories.get(report_type)
        if not cls:
            raise ValueError(f"Unknown: {report_type}")
        return cls()

report = ReportFactory.create("pdf")
report.generate()
```

**SOLID connection:** OCP (new type = new class, nothing edited) + DIP (caller depends on `Report`, not `PDFReport`)

---

### Builder

```python
class ServerBuilder:
    def __init__(self):
        self._server = Server()

    def host(self, host: str):
        self._server.host = host
        return self              # return self = method chaining

    def port(self, port: int):
        self._server.port = port
        return self

    def with_ssl(self, cert: str, key: str):
        self._server.use_ssl = True
        self._server.ssl_cert = cert
        self._server.ssl_key = key
        return self

    def debug(self):
        self._server.debug_mode = True
        return self

    def build(self) -> Server:
        self._validate()
        return self._server

    def _validate(self):
        if self._server.use_ssl and not self._server.ssl_cert:
            raise ValueError("SSL enabled but cert not provided")

# Usage
server = (ServerBuilder()
    .host("0.0.0.0")
    .port(443)
    .with_ssl("/certs/cert.pem", "/certs/key.pem")
    .build())
```

**In the wild:** Django ORM QuerySet, SQLAlchemy queries, requests sessions — all Builder.

**Factory vs Builder:**
- Factory → *what* to create (type varies)
- Builder → *how* to create it (configuration varies)

---

### Prototype

```python
import copy

class DatabaseConfig:
    def __init__(self):
        print("Loading from disk...")   # expensive — runs once
        self.host = "prod-db.internal"
        self.port = 5432
        self.options = {"charset": "utf8"}

    def clone(self):
        return copy.deepcopy(self)      # always deepcopy if nested objects exist

# Build once, clone many
base = DatabaseConfig()        # disk read — once

analytics = base.clone()
analytics.host = "analytics-db.internal"
analytics.options["charset"] = "utf16"  # doesn't affect base — deep copy

print(base.options)   # {"charset": "utf8"} — untouched
```

**Shallow vs Deep copy:**
- `copy.copy` — primitives copied, nested objects *shared* (dangerous)
- `copy.deepcopy` — fully independent, nested objects also copied (use this)

**Prototype Registry:**
```python
class PrototypeRegistry:
    def __init__(self):
        self._registry = {}

    def register(self, name: str, prototype):
        self._registry[name] = prototype

    def clone(self, name: str):
        obj = self._registry.get(name)
        if not obj:
            raise ValueError(f"No prototype: '{name}'")
        return copy.deepcopy(obj)
```

---

## Interview One-Liners

- **Singleton:** "Ensures a class has only one instance by intercepting `__new__` — best combined with DI to stay testable."
- **Factory:** "Centralises object creation so callers depend on interfaces not concrete classes — open for extension without modification."
- **Builder:** "Separates complex object construction using method chaining, keeping client code readable and construction logic centralised."
- **Prototype:** "Clones existing objects instead of building from scratch — essential when construction is expensive and you need many similar variants."

