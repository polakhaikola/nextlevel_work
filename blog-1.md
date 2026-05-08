# Plugging the Type Safety Hole: Why `unknown` Beats `any` in TypeScript

> **TL;DR** — `any` silences the TypeScript compiler entirely. `unknown` keeps the compiler on your side while still letting you handle unpredictable data. The bridge between them is **type narrowing**.

---

## Introduction

One of TypeScript's greatest promises is that it catches bugs at compile time, before your code ever runs. But TypeScript also ships with an escape hatch — the `any` type — that can quietly dissolve every guarantee it just made.

Developers often reach for `any` when they don't know the shape of incoming data: an API response, a parsed JSON blob, a third-party library return value. It feels pragmatic in the moment. It almost always causes pain later.

Since TypeScript 3.0, there has been a better answer: `unknown`. This post explains exactly why `any` is dangerous, how `unknown` fixes the problem, and how **type narrowing** lets you write safe, expressive code without sacrificing flexibility.

---

## The Problem with `any`

When you annotate a value as `any`, TypeScript stops type-checking that value **completely**. You can call methods on it, index into it, pass it anywhere, and assign it to any other type — and the compiler will never complain.

```typescript
function processResponse(data: any) {
  // TypeScript raises zero errors on any of these
  console.log(data.user.name);     // 💥 runtime crash if data has no .user
  const result: number = data;     // assigned to number — no error raised
  data.nonExistentMethod();        // silent failure waiting to happen
}
```

The variable `data` is typed as `any`, so TypeScript has opted out entirely. Every bug above becomes a **runtime error** — exactly what a type system exists to prevent.

Worse, `any` is **contagious**: once it touches another variable, that variable silently becomes `any` too.

```typescript
function double(x: any): number {
  return x * 2; // x could be a string — this returns NaN at runtime
}

const result = double("oops");
// TypeScript says result is `number`. It is actually NaN. No error raised.
```

---

## The Solution: `unknown`

`unknown` is the **type-safe counterpart** to `any`. Like `any`, it can hold a value of any shape. Unlike `any`, TypeScript **refuses to let you use that value** until you prove what it is.

```typescript
function processResponse(data: unknown) {
  console.log(data.user.name);   // ❌ Error: Object is of type 'unknown'
  const result: number = data;   // ❌ Error: Type 'unknown' not assignable to 'number'
  data.nonExistentMethod();      // ❌ Error: Object is of type 'unknown'
}
```

Every one of those previously silent bugs is now a **compile-time error**. TypeScript is forcing you to verify what you have before you use it. That verification process is called **type narrowing**.

---

## Type Narrowing: Proving What You Have

Type narrowing is the act of taking a wide type — like `unknown` — and refining it into something more specific inside a conditional block. TypeScript tracks the narrowing and adjusts what operations are legal accordingly.

### 1. `typeof` Guards — for Primitives

```typescript
function formatInput(value: unknown): string {
  if (typeof value === "string") {
    // ✅ TypeScript knows value is `string` inside this block
    return value.toUpperCase();
  }

  if (typeof value === "number") {
    // ✅ TypeScript knows value is `number` inside this block
    return value.toFixed(2);
  }

  return "Unsupported type";
}

formatInput("hello");  // → "HELLO"
formatInput(3.14159);  // → "3.14"
formatInput(true);     // → "Unsupported type"
```

### 2. `instanceof` Guards — for Classes

```typescript
function handleError(error: unknown): string {
  if (error instanceof Error) {
    // ✅ TypeScript now knows error is an `Error` instance
    return `Error caught: ${error.message}`;
  }

  return "An unknown error occurred";
}
```

### 3. Custom Type Guard Functions (Type Predicates)

For complex objects — like an API response — you write a **type predicate** function. The return type `value is SomeType` tells TypeScript to narrow the type when the function returns `true`.

```typescript
interface ApiUser {
  id: number;
  name: string;
  email: string;
}

function isApiUser(value: unknown): value is ApiUser {
  return (
    typeof value === "object" &&
    value !== null &&
    "id"    in value &&
    "name"  in value &&
    "email" in value
  );
}

async function fetchUser(id: number): Promise<string> {
  const response = await fetch(`/api/users/${id}`);
  const data: unknown = await response.json();

  if (isApiUser(data)) {
    // ✅ TypeScript now knows data is ApiUser — full autocomplete & safety
    return `Welcome, ${data.name} (${data.email})`;
  }

  throw new Error("Unexpected API response shape");
}
```

### 4. Assertion Functions

TypeScript also supports **assertion functions** that narrow a type or throw at runtime:

```typescript
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new TypeError(`Expected string, got ${typeof value}`);
  }
}

const rawConfig: unknown = loadConfig();
assertIsString(rawConfig);
// ✅ After this line, TypeScript knows rawConfig is a string
console.log(rawConfig.trim());
```

---

## `any` vs `unknown` — Side by Side

| Behaviour | `any` | `unknown` |
|---|---|---|
| Can hold any value | ✅ | ✅ |
| Access properties freely | ✅ (unsafe) | ❌ compile error |
| Assign to typed variable without a check | ✅ (unsafe) | ❌ compile error |
| Requires type check before use | ❌ | ✅ |
| Contagious to other variables | ✅ | ❌ |
| Ideal for boundary data (API, JSON, I/O) | ⚠️ Risky | ✅ Safe |

---

## Real-World Pattern: Safe API Response Parsing

```typescript
interface Product {
  id: number;
  title: string;
  price: number;
}

function isProduct(value: unknown): value is Product {
  return (
    typeof value === "object" &&
    value !== null &&
    typeof (value as Record<string, unknown>).id    === "number" &&
    typeof (value as Record<string, unknown>).title === "string" &&
    typeof (value as Record<string, unknown>).price === "number"
  );
}

async function getProduct(id: number): Promise<Product> {
  const res = await fetch(`/api/products/${id}`);

  if (!res.ok) {
    throw new Error(`HTTP ${res.status}`);
  }

  const json: unknown = await res.json();

  if (!isProduct(json)) {
    throw new Error("Response did not match expected Product shape");
  }

  // ✅ TypeScript knows json is Product from here on
  return json;
}
```

The boundary between your trusted TypeScript world and the unpredictable outside world is managed in exactly one place — the type guard. Everything downstream stays fully typed.

---

## Conclusion

`any` is an escape hatch that disables TypeScript's type system entirely. Every use of `any` is a bet that you'll remember — perfectly, forever — exactly what that value can and cannot be. That bet rarely pays off in a growing codebase.

`unknown` makes the same promise to hold any value, but enforces one condition: **you must prove what it is before you use it**. That proof is written with type narrowing — `typeof`, `instanceof`, custom type predicates, and assertion functions.

The type safety TypeScript promises is only as strong as your commitment to not punch holes in it. Reaching for `unknown` instead of `any` at your application's data boundaries is one of the highest-leverage habits you can build as a TypeScript developer.
