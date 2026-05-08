# Slicing Interfaces the Smart Way: `Pick`, `Omit`, and the DRY Principle in TypeScript

> **TL;DR** — Instead of writing multiple nearly-identical interfaces by hand, `Pick` and `Omit` let you derive precise, focused types from a single master interface. Your types stay in sync automatically, and your codebase stays DRY.

---

## Introduction

In any real-world application you will quickly find yourself with a central data model — a `User`, a `Product`, a `Post` — that different layers of the application need in slightly different forms. A registration form doesn't need `id` or `createdAt`. A public profile page shouldn't expose `password` or `internalNotes`. An update endpoint only needs the fields that are editable.

The naive solution is to write a separate interface for every scenario. It works, but it creates a maintenance nightmare: when the master model changes, every hand-written slice has to be updated manually. Miss one, and your types lie to you.

TypeScript's built-in utility types `Pick` and `Omit` solve this cleanly. They let you derive exactly the slice you need from one authoritative source of truth, so that changes propagate automatically.

---

## The Problem: Interface Duplication

Imagine a `User` master interface and what happens without utility types:

```typescript
// The master interface
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  role: "admin" | "user";
  createdAt: Date;
  updatedAt: Date;
}

// ❌ BAD — Written by hand, duplicating fields
interface UserRegistrationForm {
  name: string;     // copied from User
  email: string;    // copied from User
  password: string; // copied from User
}

// ❌ BAD — Another hand-written copy
interface PublicUserProfile {
  id: number;    // copied
  name: string;  // copied
  role: "admin" | "user"; // copied
}

// ❌ BAD — Yet another copy
interface UserUpdatePayload {
  name: string;   // copied
  email: string;  // copied
}
```

Every one of these interfaces is a liability. If `name` changes type in `User`, you have four places to update instead of one. If you add a `username` field to `User` and forget to add it to `UserRegistrationForm`, TypeScript won't warn you — the types have drifted silently.

---

## `Pick<T, K>` — Keep Only What You Need

`Pick<T, K>` constructs a new type by **selecting** a subset of keys `K` from type `T`. Everything not listed in `K` is discarded.

```typescript
type Pick<T, K extends keyof T> = {
  [P in K]: T[P];
};
```

In practice:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  role: "admin" | "user";
  createdAt: Date;
  updatedAt: Date;
}

// ✅ GOOD — Derived from User, not duplicated
type UserRegistrationForm = Pick<User, "name" | "email" | "password">;
// Equivalent to: { name: string; email: string; password: string }

type PublicUserProfile = Pick<User, "id" | "name" | "role">;
// Equivalent to: { id: number; name: string; role: "admin" | "user" }

type UserUpdatePayload = Pick<User, "name" | "email">;
// Equivalent to: { name: string; email: string }
```

Now add `username: string` to `User`, and `UserRegistrationForm` automatically includes it if you add `"username"` to its key list — or you deliberately choose not to include it. The decision is explicit and in one place.

---

## `Omit<T, K>` — Exclude What You Don't Want

`Omit<T, K>` is the inverse: it constructs a type by **removing** keys `K` from `T` and keeping everything else.

```typescript
type Omit<T, K extends keyof T> = Pick<T, Exclude<keyof T, K>>;
```

In practice:

```typescript
// ✅ GOOD — Everything except sensitive / server-managed fields
type PublicUser = Omit<User, "password" | "createdAt" | "updatedAt">;
// Result: { id: number; name: string; email: string; role: "admin" | "user" }

type UserWithoutDates = Omit<User, "createdAt" | "updatedAt">;
// Result: { id: number; name: string; email: string; password: string; role: "admin" | "user" }
```

`Omit` shines when the list of fields you want to **keep** is much longer than the list you want to **drop**. Instead of naming every field to keep, you name only the ones to exclude.

---

## Choosing Between `Pick` and `Omit`

| Scenario | Use |
|---|---|
| You want a **small** subset of a large interface | `Pick` — name the few fields you want |
| You want **most** of a large interface, minus a few fields | `Omit` — name the few fields to drop |
| Removing sensitive data (passwords, secrets) | `Omit` — explicit exclusion is safer |
| Building a focused form or DTO | `Pick` — only the fields the form touches |

---

## Combining with Other Utility Types

`Pick` and `Omit` compose naturally with `Partial`, `Required`, and `Readonly` to handle real API scenarios:

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  role: "admin" | "user";
  createdAt: Date;
  updatedAt: Date;
}

// A PATCH endpoint body: id is required, editable fields are all optional
type UserPatchPayload = { id: number } & Partial<Pick<User, "name" | "email">>;
// { id: number; name?: string; email?: string }

// A safe read-only snapshot to pass into a display component
type UserSnapshot = Readonly<Omit<User, "password">>;
// All fields except password, and none of them writable

// A create payload: no server-managed fields, password required
type CreateUserDto = Omit<User, "id" | "createdAt" | "updatedAt">;
// { name: string; email: string; password: string; role: "admin" | "user" }
```

---

## Real-World Pattern: A Layered API Service

Here is how these types flow through a complete service layer, each layer receiving exactly the data it should see:

```typescript
interface Post {
  id: number;
  title: string;
  body: string;
  authorId: number;
  isDraft: boolean;
  publishedAt: Date | null;
  internalNotes: string;
}

// The shape accepted when creating a new post
type CreatePostDto = Pick<Post, "title" | "body" | "authorId" | "isDraft">;

// The shape accepted when updating a post
type UpdatePostDto = Partial<Pick<Post, "title" | "body" | "isDraft">>;

// The shape returned to the public API — no internal fields
type PublicPost = Omit<Post, "internalNotes" | "authorId">;

// Service layer — each function is precisely typed
class PostService {
  create(dto: CreatePostDto): Post {
    return {
      ...dto,
      id: Date.now(),
      publishedAt: null,
      internalNotes: "",
    };
  }

  update(id: number, dto: UpdatePostDto): Post {
    // fetch post and merge dto fields
    const post = this.findById(id);
    return { ...post, ...dto };
  }

  getPublic(id: number): PublicPost {
    const { internalNotes, authorId, ...publicPost } = this.findById(id);
    return publicPost;
  }

  private findById(id: number): Post {
    // DB lookup — implementation omitted
    return {} as Post;
  }
}
```

`CreatePostDto`, `UpdatePostDto`, and `PublicPost` are all derived from `Post`. If `Post` gains a new field, the derived types update automatically with zero extra work.

---

## Why This Matters: The DRY Principle

**DRY — Don't Repeat Yourself** — is the principle that every piece of knowledge in a system should have a single, authoritative representation. When you duplicate interfaces by hand, you are violating DRY at the type level.

With `Pick` and `Omit`:

- There is **one source of truth** — the master interface.
- All derived types are **structurally linked** to it.
- **Renames and additions propagate automatically** — TypeScript errors point you to any derived type that needs attention.
- **Code reviews become cleaner** — a change to the data model is one diff, not five.

---

## Conclusion

`Pick` and `Omit` are small utility types with outsized impact. They turn your master interface into a flexible, composable building block rather than a template you copy and paste. The result is a codebase where your types are always in sync with your data model, changes are made in one place, and the TypeScript compiler does the work of catching inconsistencies for you.

The next time you find yourself writing an interface that looks almost like one you already have, stop and ask: can `Pick` or `Omit` derive what I need instead? In almost every case, the answer is yes — and your future self maintaining that code will be grateful.
