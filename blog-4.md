# The Four Pillars of OOP in TypeScript: Writing Large-Scale Code That Stays Manageable

> **TL;DR** — Encapsulation, Abstraction, Inheritance, and Polymorphism are not academic concepts. They are four concrete tools that reduce complexity, prevent bugs, and keep large TypeScript codebases navigable as they grow.

---

## Introduction

Object-Oriented Programming (OOP) is built on four foundational principles. You have probably seen them listed in textbooks. What textbooks often miss is *why* each principle exists — what specific pain it was designed to eliminate — and how TypeScript's type system makes each one significantly more powerful than in dynamically typed languages.

This post walks through all four pillars with realistic TypeScript examples drawn from the kind of code you write in production: services, repositories, payment processors, and notification systems. By the end, each pillar should feel less like a theory and more like a tool you reach for instinctively.

---

## Pillar 1: Encapsulation — Protecting State from the Outside World

**Encapsulation** means bundling data and the methods that operate on that data together, and controlling what the outside world can see and touch. It is enforced in TypeScript through `private`, `protected`, and `public` access modifiers.

### The Problem Without It

```typescript
// Without encapsulation — everything is exposed
class BankAccount {
  balance: number = 0;
  transactionHistory: string[] = [];
}

const account = new BankAccount();
account.balance = -99999;       // 💥 No validation — direct mutation
account.transactionHistory = []; // 💥 History wiped with a single line
```

Nothing stops external code from putting the object into an invalid state.

### The Solution With Encapsulation

```typescript
class BankAccount {
  private balance: number;
  private transactionHistory: string[] = [];

  constructor(initialDeposit: number) {
    if (initialDeposit < 0) throw new Error("Initial deposit cannot be negative");
    this.balance = initialDeposit;
    this.log(`Account opened with $${initialDeposit}`);
  }

  deposit(amount: number): void {
    if (amount <= 0) throw new Error("Deposit amount must be positive");
    this.balance += amount;
    this.log(`Deposited $${amount}`);
  }

  withdraw(amount: number): void {
    if (amount > this.balance) throw new Error("Insufficient funds");
    this.balance -= amount;
    this.log(`Withdrew $${amount}`);
  }

  // Read-only public access to private state
  getBalance(): number {
    return this.balance;
  }

  getHistory(): ReadonlyArray<string> {
    return this.transactionHistory;
  }

  private log(entry: string): void {
    this.transactionHistory.push(`[${new Date().toISOString()}] ${entry}`);
  }
}

const account = new BankAccount(100);
account.deposit(50);
account.withdraw(30);

console.log(account.getBalance()); // 120

account.balance = -99999;  // ❌ Error: Property 'balance' is private
account.log("Hacked");     // ❌ Error: Property 'log' is private
```

The object now owns its state completely. External code can only interact with `BankAccount` through its public API — a controlled, validated interface.

### `protected` for Subclass Access

`protected` is a middle ground: invisible to external code, but accessible to subclasses:

```typescript
class Shape {
  protected color: string;

  constructor(color: string) {
    this.color = color;
  }
}

class Circle extends Shape {
  describe(): string {
    return `A ${this.color} circle`; // ✅ protected is accessible in subclass
  }
}
```

---

## Pillar 2: Abstraction — Hiding Complexity Behind Simple Interfaces

**Abstraction** means exposing only what is necessary and hiding the implementation details. In TypeScript, this is achieved with `abstract` classes and `interface` definitions. Consumers of your code work with a clean, intention-revealing API — they do not need to know how it works, only what it does.

### Abstract Classes

An `abstract` class defines the shape of a family of classes — including methods that must be implemented — without providing a complete implementation itself:

```typescript
abstract class Notification {
  // Subclasses must provide their own implementation
  abstract send(recipient: string, message: string): Promise<void>;

  // Shared concrete logic available to all subclasses
  protected formatMessage(message: string): string {
    return `[${new Date().toLocaleTimeString()}] ${message}`;
  }

  // High-level method that calls the abstract method
  async notify(recipient: string, message: string): Promise<void> {
    const formatted = this.formatMessage(message);
    await this.send(recipient, formatted);
    console.log(`Notification sent to ${recipient}`);
  }
}

class EmailNotification extends Notification {
  async send(recipient: string, message: string): Promise<void> {
    // Implementation detail hidden from callers
    console.log(`📧 Sending email to ${recipient}: ${message}`);
  }
}

class SmsNotification extends Notification {
  async send(recipient: string, message: string): Promise<void> {
    console.log(`📱 Sending SMS to ${recipient}: ${message}`);
  }
}

// Can't instantiate the abstract class itself
const n = new Notification(); // ❌ Error: Cannot create an instance of an abstract class

// Works with concrete subclasses
const email = new EmailNotification();
await email.notify("alice@example.com", "Your order has shipped.");
```

`notify` contains the shared orchestration logic. `send` contains the channel-specific detail. Callers never touch `send` directly — they call `notify` and let the subclass decide how to deliver.

### Interfaces for Pure Abstraction

An interface describes a contract with zero implementation:

```typescript
interface PaymentProcessor {
  charge(amount: number, currency: string): Promise<boolean>;
  refund(transactionId: string, amount: number): Promise<boolean>;
}

class StripeProcessor implements PaymentProcessor {
  async charge(amount: number, currency: string): Promise<boolean> {
    console.log(`Charging $${amount} ${currency} via Stripe`);
    return true;
  }

  async refund(transactionId: string, amount: number): Promise<boolean> {
    console.log(`Refunding $${amount} for transaction ${transactionId} via Stripe`);
    return true;
  }
}

class PayPalProcessor implements PaymentProcessor {
  async charge(amount: number, currency: string): Promise<boolean> {
    console.log(`Charging $${amount} ${currency} via PayPal`);
    return true;
  }

  async refund(transactionId: string, amount: number): Promise<boolean> {
    console.log(`Refunding $${amount} via PayPal`);
    return true;
  }
}
```

The rest of your application only depends on the `PaymentProcessor` interface — never on `StripeProcessor` or `PayPalProcessor` directly. Swapping providers is a one-line change.

---

## Pillar 3: Inheritance — Sharing Behaviour Across Related Classes

**Inheritance** lets a subclass acquire the properties and methods of a parent class, then extend or override them. It eliminates code duplication across related entities and creates natural hierarchies.

```typescript
class Vehicle {
  constructor(
    protected make: string,
    protected model: string,
    protected year: number,
  ) {}

  getInfo(): string {
    return `${this.year} ${this.make} ${this.model}`;
  }

  startEngine(): string {
    return `${this.getInfo()} engine started.`;
  }
}

class Car extends Vehicle {
  constructor(make: string, model: string, year: number, private doors: number) {
    super(make, model, year);
  }

  getInfo(): string {
    return `${super.getInfo()} (${this.doors}-door car)`;
  }
}

class ElectricCar extends Car {
  constructor(
    make: string,
    model: string,
    year: number,
    doors: number,
    private batteryKwh: number,
  ) {
    super(make, model, year, doors);
  }

  getInfo(): string {
    return `${super.getInfo()} — ${this.batteryKwh}kWh battery`;
  }

  startEngine(): string {
    return `${this.getInfo()} motor silently activated.`;
  }
}

const tesla = new ElectricCar("Tesla", "Model 3", 2024, 4, 75);
console.log(tesla.getInfo());
// → "2024 Tesla Model 3 (4-door car) — 75kWh battery"

console.log(tesla.startEngine());
// → "2024 Tesla Model 3 (4-door car) — 75kWh battery motor silently activated."
```

`Vehicle` defines the foundation. `Car` adds door count. `ElectricCar` adds battery information and overrides engine behaviour. Each level adds only what is new — nothing is duplicated.

### When Not to Use Inheritance

Inheritance models an **is-a** relationship. `ElectricCar` **is a** `Car`. Use **composition** (has-a) when the relationship is about capability rather than identity. A `Car` **has a** `Engine`, not **is an** `Engine`.

---

## Pillar 4: Polymorphism — One Interface, Many Behaviours

**Polymorphism** (from Greek: "many forms") means that different classes can be used interchangeably through a shared interface or base class, with each class providing its own implementation. It is what makes abstraction useful in practice.

```typescript
abstract class Employee {
  constructor(
    protected name: string,
    protected baseSalary: number
  ) {}

  abstract calculatePay(): number;

  abstract getRole(): string;

  getPaySummary(): string {
    return `${this.name} (${this.getRole()}): $${this.calculatePay().toFixed(2)}`;
  }
}

class FullTimeEmployee extends Employee {
  getRole(): string { return "Full-Time"; }

  calculatePay(): number {
    return this.baseSalary / 12; // Monthly salary
  }
}

class ContractEmployee extends Employee {
  constructor(
    name: string,
    private hourlyRate: number,
    private hoursWorked: number
  ) {
    super(name, 0);
  }

  getRole(): string { return "Contractor"; }

  calculatePay(): number {
    return this.hourlyRate * this.hoursWorked;
  }
}

class CommissionEmployee extends Employee {
  constructor(
    name: string,
    baseSalary: number,
    private salesAmount: number,
    private commissionRate: number
  ) {
    super(name, baseSalary);
  }

  getRole(): string { return "Sales"; }

  calculatePay(): number {
    return this.baseSalary / 12 + this.salesAmount * this.commissionRate;
  }
}

// Polymorphism in action — all treated identically through the shared base type
function runPayroll(employees: Employee[]): void {
  console.log("=== Monthly Payroll ===");
  let total = 0;

  for (const employee of employees) {
    console.log(employee.getPaySummary()); // Each calls its own calculatePay()
    total += employee.calculatePay();
  }

  console.log(`Total payout: $${total.toFixed(2)}`);
}

const team: Employee[] = [
  new FullTimeEmployee("Alice", 96000),
  new ContractEmployee("Bob", 85, 160),
  new CommissionEmployee("Carol", 48000, 50000, 0.05),
];

runPayroll(team);
// === Monthly Payroll ===
// Alice (Full-Time): $8000.00
// Bob (Contractor): $13600.00
// Carol (Sales): $6500.00
// Total payout: $28100.00
```

`runPayroll` knows nothing about `FullTimeEmployee`, `ContractEmployee`, or `CommissionEmployee`. It only knows `Employee`. Adding a fourth employee type — say `FreelanceEmployee` — requires zero changes to `runPayroll`. That is the power of polymorphism.

---

## All Four Pillars Together: A Complete Example

Here is a compact example that uses all four pillars in a single cohesive system:

```typescript
// Abstraction — defines the contract
interface Logger {
  log(level: "info" | "warn" | "error", message: string): void;
}

// Encapsulation — internal state is private
abstract class BaseService {
  private logs: string[] = [];

  constructor(protected readonly logger: Logger) {}

  protected recordLog(message: string): void {
    this.logs.push(message);
    this.logger.log("info", message);
  }

  getLogs(): ReadonlyArray<string> {
    return this.logs;
  }

  // Abstraction — forces subclasses to define their core behaviour
  abstract execute(input: string): string;
}

// Inheritance — shares BaseService's behaviour
class UpperCaseService extends BaseService {
  // Polymorphism — its own implementation of execute
  execute(input: string): string {
    const result = input.toUpperCase();
    this.recordLog(`UpperCaseService processed: "${input}" → "${result}"`);
    return result;
  }
}

class ReverseService extends BaseService {
  // Polymorphism — completely different implementation of execute
  execute(input: string): string {
    const result = input.split("").reverse().join("");
    this.recordLog(`ReverseService processed: "${input}" → "${result}"`);
    return result;
  }
}
```

Each pillar plays a distinct role: **Encapsulation** hides `logs`. **Abstraction** defines the `Logger` interface and `execute` contract. **Inheritance** gives both services access to `recordLog` and `getLogs`. **Polymorphism** means both services can be used anywhere a `BaseService` is expected, each behaving differently.

---

## Conclusion

The four pillars of OOP are not four separate ideas — they are four facets of the same underlying goal: **managing complexity by creating clear boundaries and responsibilities**.

- **Encapsulation** keeps objects in valid states by preventing unauthorized external access.
- **Abstraction** hides implementation detail so consumers only interact with what they need.
- **Inheritance** eliminates duplication by sharing behaviour across related classes.
- **Polymorphism** makes code extensible — you can add new behaviours without changing existing code.

TypeScript strengthens all four with its type system: `private`/`protected` enforce encapsulation at compile time, `interface` and `abstract class` formalize abstraction contracts, type checking validates inheritance hierarchies, and union types and generics expand what polymorphism can express.

In large-scale projects, these are not nice-to-have principles. They are the difference between a codebase that remains navigable at 100,000 lines and one that becomes impossible to reason about at 10,000.
