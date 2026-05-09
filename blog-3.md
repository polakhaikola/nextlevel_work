# Write Once, Type Everywhere: Mastering TypeScript Generics

> **TL;DR** — Generics let you write a single function, class, or component that works correctly with many different types — without sacrificing the type safety that makes TypeScript valuable in the first place.

---

## Introduction

Every non-trivial codebase eventually runs into the same tension: you want to **reuse logic** across different data shapes, but you also want TypeScript to **catch type errors** specific to each shape. Without generics, you are forced to choose one or the other — either duplicate your logic for every type, or surrender type safety by falling back to `any`.

Generics resolve this tension entirely. They let you write logic once and parameterize the types it operates on, the same way function parameters let you write logic once and parameterize the *values* it operates on. This post walks through generics from first principles to practical, production-ready patterns.

---

## The Problem Generics Solve

Suppose you want a function that returns the first element of an array. Without generics, you face a hard choice:

**Option A — Duplicate for each type (violates DRY):**

```typescript
function firstNumber(arr: number[]): number   { return arr[0]; }
function firstString(arr: string[]): string   { return arr[0]; }
function firstBoolean(arr: boolean[]): boolean { return arr[0]; }
// ... one function for every type you will ever use
```

**Option B — Use `any` (destroys type safety):**

```typescript
function first(arr: any[]): any {
  return arr[0];
}

const result = first([1, 2, 3]);
result.toUpperCase(); // TypeScript says nothing — 💥 runtime crash
```

Neither option is acceptable. Option A does not scale. Option B is not type-safe. Generics give you Option C — one function, full type safety.

---

## Generic Syntax: The Type Parameter

A **type parameter** is written in angle brackets — `<T>` — and acts as a placeholder that TypeScript fills in based on how the function is called:

```typescript
function first<T>(arr: T[]): T {
  return arr[0];
}

const num  = first([1, 2, 3]);          // T = number  → num:  number
const str  = first(["a", "b", "c"]);   // T = string  → str:  string
const bool = first([true, false]);      // T = boolean → bool: boolean

num.toUpperCase();  // ❌ Error: Property 'toUpperCase' does not exist on type 'number'
str.toUpperCase();  // ✅ Fine — TypeScript knows str is a string
```

`T` is just a naming convention — short for "Type". You can name type parameters anything. `K`/`V` for key/value pairs, `E` for element type, and `T`/`U` for general types are all common conventions.

---

## Multiple Type Parameters

Functions can accept more than one type parameter. A practical example — transforming an array while keeping both the original and result, fully typed:

```typescript
function mapWithOriginal<T, U>(
  arr: T[],
  transform: (item: T) => U
): { original: T[]; transformed: U[] } {
  return {
    original: arr,
    transformed: arr.map(transform),
  };
}

const result = mapWithOriginal([1, 2, 3], (n) => n.toString());
// result.original:    number[]  ✅
// result.transformed: string[]  ✅

const lengths = mapWithOriginal(["hello", "world"], (s) => s.length);
// lengths.original:    string[]  ✅
// lengths.transformed: number[]  ✅
```

---

## Generic Constraints: `extends`

An unconstrained `T` can be anything. Sometimes you need to guarantee that `T` has certain properties. You express this with the `extends` keyword:

```typescript
// Without constraint — TypeScript doesn't know T has a .length property
function logLength<T>(value: T): void {
  console.log(value.length); // ❌ Error: Property 'length' does not exist on type 'T'
}

// With constraint — T must have at least a length: number property
function logLength<T extends { length: number }>(value: T): void {
  console.log(value.length); // ✅
}

logLength("hello");        // → 5   (string has .length)
logLength([1, 2, 3]);      // → 3   (arrays have .length)
logLength({ length: 10 }); // → 10  (plain objects work too)
logLength(42);             // ❌ Error: number has no .length
```

The canonical constraint is `keyof` — used to ensure a key actually exists on an object at compile time:

```typescript
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 1, name: "Alice", role: "admin" };

getProperty(user, "name");   // ✅ → "Alice"  (typed as string)
getProperty(user, "id");     // ✅ → 1         (typed as number)
getProperty(user, "email");  // ❌ Error: '"email"' is not a key of typeof user
```

---

## Generic Interfaces

Generics are not limited to functions. You can parameterize interfaces, type aliases, and classes too.

A common real-world pattern — every API response shares the same envelope structure, but the payload type differs per endpoint:

```typescript
interface ApiResponse<T> {
  data: T;
  status: number;
  message: string;
  timestamp: string;
}

interface User    { id: number; name: string; }
interface Product { id: number; title: string; price: number; }

type UserResponse    = ApiResponse<User>;
type ProductResponse = ApiResponse<Product>;

function handleUserResponse(res: UserResponse): string {
  return res.data.name; // ✅ TypeScript knows res.data is User
}

function handleProductResponse(res: ProductResponse): string {
  return `${res.data.title} — $${res.data.price}`; // ✅ TypeScript knows res.data is Product
}
```

One interface definition. Infinite type-safe variations.

---

## Generic Type Aliases — The `Result<T, E>` Pattern

A `Result<T, E>` type models success or failure explicitly, without throwing exceptions — a pattern increasingly common in TypeScript:

```typescript
type Result<T, E = Error> =
  | { success: true;  value: T }
  | { success: false; error: E };

function divide(a: number, b: number): Result<number, string> {
  if (b === 0) {
    return { success: false, error: "Division by zero is not allowed" };
  }
  return { success: true, value: a / b };
}

const result = divide(10, 2);

if (result.success) {
  console.log(result.value * 2); // ✅ TypeScript knows result.value is number
} else {
  console.log(result.error);     // ✅ TypeScript knows result.error is string
}
```

---

## Generic Classes

Classes benefit from generics just as much as functions. A generic `Stack<T>` enforces that you only push and pop values of a consistent, known type:

```typescript
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  peek(): T | undefined {
    return this.items[this.items.length - 1];
  }

  get size(): number {
    return this.items.length;
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }
}

const numberStack = new Stack<number>();
numberStack.push(1);
numberStack.push(2);
numberStack.push("three"); // ❌ Error: Argument of type 'string' is not assignable to 'number'

const stringStack = new Stack<string>();
stringStack.push("hello");
stringStack.push("world");
const top = stringStack.peek(); // ✅ TypeScript knows top is string | undefined
```

---

## Real-World Pattern: Generic Repository

One of the most powerful applications of generics in large TypeScript projects is a **generic data repository** — a class that abstracts CRUD operations and works for any entity type:

```typescript
interface Entity {
  id: number;
}

class Repository<T extends Entity> {
  private store: Map<number, T> = new Map();

  save(entity: T): T {
    this.store.set(entity.id, entity);
    return entity;
  }

  findById(id: number): T | undefined {
    return this.store.get(id);
  }

  findAll(): T[] {
    return Array.from(this.store.values());
  }

  delete(id: number): boolean {
    return this.store.delete(id);
  }
}

// Two fully typed repositories — from one class definition
interface User  extends Entity { name: string; email: string; }
interface Order extends Entity { userId: number; total: number; }

const userRepo  = new Repository<User>();
const orderRepo = new Repository<Order>();

userRepo.save({ id: 1, name: "Alice", email: "alice@example.com" });
const user = userRepo.findById(1);   // ✅ TypeScript knows: User | undefined

orderRepo.save({ id: 101, userId: 1, total: 49.99 });
const order = orderRepo.findById(101); // ✅ TypeScript knows: Order | undefined

// Cross-contamination is impossible
userRepo.save({ id: 2, userId: 1, total: 20 }); // ❌ Error: Object is not assignable to User
```

One class definition. Multiple fully typed repositories. No duplication. No `any`.

---

## Built-in Utility Types Are Generics

TypeScript's built-in utility types are all implemented with generics under the hood. Understanding generics helps you read and extend them:

```typescript
// How Partial<T> works — makes every property optional
type Partial<T> = {
  [K in keyof T]?: T[K];
};

// How Readonly<T> works — makes every property immutable
type Readonly<T> = {
  readonly [K in keyof T]: T[K];
};

// How ReturnType<T> works — extracts a function's return type
type ReturnType<T extends (...args: any) => any> =
  T extends (...args: any) => infer R ? R : never;

function buildConfig() {
  return { host: "localhost", port: 3000, ssl: false };
}

type Config = ReturnType<typeof buildConfig>;
// ✅ { host: string; port: number; ssl: boolean }
```

---

## Conclusion

Generics are the feature that transforms TypeScript from "JavaScript with type annotations" into a genuinely expressive, scalable type system. They let you write logic once, apply it broadly, and have the compiler enforce correctness at every call site — all without a single `any` in sight.

The mental model is straightforward: a **type parameter is a variable for types**. Just as a function parameter lets you reuse logic across different values, a type parameter lets you reuse logic across different types. Once that model clicks, generics stop feeling like magic and start feeling like the obvious tool for the job.

Start with generic utility functions. Move to generic interfaces for API boundaries. Graduate to generic classes for services and repositories. Each step builds on the same idea — **write once, type everywhere**.
