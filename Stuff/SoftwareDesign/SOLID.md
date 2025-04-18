### 🧩 Single Responsibility Principle (SRP)

> **“A class should have one, and only one, reason to change.”**

**Bad Example:**

```python
class UserManager:
    def __init__(self, username):
        self.username = username

    def save_to_db(self):
        # directly handles database logic
        print(f"Saving {self.username} to DB...")  # imagine actual DB code here

    def send_welcome_email(self):
        # directly handles email logic
        print(f"Sending welcome email to {self.username}...")
```
Here, `UserManager` does two things: persistence **and** email-sending.

**Good Example:**

```python
class User:
    def __init__(self, username):
        self.username = username

class UserRepository:
    def save(self, user: User):
        print(f"Saving {user.username} to DB...")

class EmailService:
    def send_welcome(self, user: User):
        print(f"📧 Sending welcome email to {user.username}...")

# Usage:
user = User("alice")
repo = UserRepository()
mailer = EmailService()

repo.save(user)                # Persistence
mailer.send_welcome(user)      # Notification
```

Now each class has a single responsibility—and a single reason to change.

---
### 🛡️ Open/Closed Principle (OCP)

> **“Software entities should be open for extension, but closed for modification.”**

**Bad Example:**

```python
class DiscountCalculator:
    def calculate(self, order, discount_type):
        if discount_type == "fixed":
            return 10
        elif discount_type == "percentage":
            return order.total * 0.1
```
Adding a new discount means editing `calculate()`, risking regressions.

**Good Example:**

```python
from abc import ABC, abstractmethod

class DiscountStrategy(ABC):
    @abstractmethod
    def compute(self, order):
        pass

class FixedDiscount(DiscountStrategy):
    def compute(self, order):
        return 10

class PercentageDiscount(DiscountStrategy):
    def compute(self, order):
        return order.total * 0.1

class DiscountCalculator:
    def __init__(self, strategy: DiscountStrategy):
        self.strategy = strategy

    def calculate(self, order):
        return self.strategy.compute(order)

# Usage:
class Order: 
    def __init__(self, total):
        self.total = total

order = Order(200)
calculator = DiscountCalculator(PercentageDiscount())
print(f"Discount applied ➕ {calculator.calculate(order)}")
```

You can add new `DiscountStrategy` subclasses without touching `DiscountCalculator`.

---
### 🐦 Liskov Substitution Principle (LSP)

> **“Subtypes must be substitutable for their base types.”**

**Bad Example:**

```python
class Rectangle:
    def __init__(self, w, h):
        self.width = w
        self.height = h

    def area(self):
        return self.width * self.height

class Square(Rectangle):
    def __init__(self, side):
        super().__init__(side, side)

# Client code expects to set width and height independently:
rect = Rectangle(2, 3)
rect.width = 5
rect.height = 4
print(rect.area())  # 20

sq = Square(5)
sq.width = 6      # oops—height stays 5, area breaks expectations
print(sq.area())  # 30 (but we might expect 36)
```

`Square` violates the expected behavior of `Rectangle`.

**Good Example:**

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self):
        pass

class Rectangle(Shape):
    def __init__(self, w, h):
        self.width = w
        self.height = h

    def area(self):
        return self.width * self.height

class Square(Shape):
    def __init__(self, side):
        self.side = side

    def area(self):
        return self.side * self.side

# Usage:
def print_area(shape: Shape):
    print(f"🔢 Area: {shape.area()}")

print_area(Rectangle(2, 3))
print_area(Square(5))
```

Now `Square` and `Rectangle` share only what makes sense—a common `Shape` interface.

---
### 📐 Interface Segregation Principle (ISP)

> **“Clients should not be forced to depend on interfaces they do not use.”**

**Bad Example:**

```python
class MultiTool:
    def cut(self): pass
    def screw(self): pass
    def drill(self): pass

class OnlyCutter(MultiTool):
    def cut(self):
        print("Cutting")
    def screw(self):
        raise NotImplementedError  # unwanted
    def drill(self):
        raise NotImplementedError
```

`OnlyCutter` must implement—or stub out—methods it doesn’t need.

**Good Example:**

```python
from abc import ABC, abstractmethod

class Cutter(ABC):
    @abstractmethod
    def cut(self): pass

class Screwdriver(ABC):
    @abstractmethod
    def screw(self): pass

class Drill(ABC):
    @abstractmethod
    def drill(self): pass

class MultiTool(Cutter, Screwdriver, Drill):
    def cut(self):    print("Cutting")
    def screw(self):  print("Screwing")
    def drill(self):  print("Drilling")

class OnlyCutter(Cutter):
    def cut(self):
        print("Cutting only")

# Usage:
tool = OnlyCutter()
tool.cut()
```

Each class depends only on the methods it actually uses.

---
### 🔌 Dependency Inversion Principle (DIP)

> **“High-level modules should not depend on low-level modules; both should depend on abstractions.”**

**Bad Example:**

```python
class MySQLDatabase:
    def connect(self): pass
    def query(self, q): pass

class UserService:
    def __init__(self):
        self.db = MySQLDatabase()     # direct dependency!

    def get_user(self, user_id):
        return self.db.query(f"SELECT * FROM users WHERE id={user_id}")
```

`UserService` is tightly coupled to `MySQLDatabase`.

**Good Example:**

```python
from abc import ABC, abstractmethod

class Database(ABC):
    @abstractmethod
    def connect(self): pass

    @abstractmethod
    def query(self, q): pass

class MySQLDatabase(Database):
    def connect(self): print("🔗 Connecting to MySQL")
    def query(self, q): print(f"📝 Executing SQL: {q}")

class UserService:
    def __init__(self, db: Database):
        self.db = db                # depends on abstraction

    def get_user(self, user_id):
        return self.db.query(f"SELECT * FROM users WHERE id={user_id}")

# Usage:
db = MySQLDatabase()
service = UserService(db)
service.get_user(42)
```

By injecting a `Database` abstraction, you can swap in PostgreSQL, MongoDB, a mock for testing, etc., without touching your high‑level logic.

---
🎉 **Wrap‑up:**

- **SRP** 🧩 → one reason to change
- **OCP** 🛡️ → extend without modifying
- **LSP** 🐦 → subclasses act like their base
- **ISP** 📐 → keep interfaces fine‑grained
- **DIP** 🔌 → depend on abstractions