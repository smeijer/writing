---
title: Decorators without the magic — HOFs, the standard, and TS legacy
description: Start with HOFs/HOCs you already know, then see how ECMAScript decorators solve it, what TypeScript’s experimental decorators did differently, and how to write one decorator that works in both worlds.
tags: code
date: 2025-10-29
draft: true
---

If decorators ever felt like magic, it's time to look behind the curtains. It's just function wrapping. If you’ve written a higher‑order function or a React HOC, you already understand decorators. The rest is syntax and timing.

## The background: HOFs and HOCs

A higher‑order function (HOF) takes a function and returns a function to add behavior around the original.

```js
function withLogging(fn, label) {
  return function (...args) {
    console.log(`[${label}] args:`, args)
    const result = fn.call(this, ...args)
    console.log(`[${label}] result:`, result)
    return result
  }
}

const add = (a, b) => a + b
const loggedAdd = withLogging(add, 'add')

loggedAdd(1, 2)
```

That wrapper adds behavior “around” `add` without touching `add` itself.

React HOCs are the same pattern for components.

```jsx
const withSpinner = (Component) => (props) => {
  if (props.loading) return <div className="spinner" />
  return <Component {...props} />
}

const User = ({ user }) => <div>{user.name}</div>
export const UserWithSpinner = withSpinner(User)
```

Same pattern: take something, return a wrapped version. Keep that model in your head. We’ll translate it to decorators next.

## JavaScript decorators

EcmaScript decorators are Stage 3, meaning the spec is written and browsers should implement it. TypeScript 5+ implements the current semantics, but native engine support is still limited. In practice, you’ll use a compiler (TS/Babel) that down‑emits them for today’s targets. They run at class definition time and let you wrap methods/fields where they live.

Method example — logging (same idea as the HOF):

```ts
function logged(value: any, context: ClassMethodDecoratorContext) {
  return function (this: any, ...args: unknown[]) {
    console.log(`[${context.name}] args:`, args)
    const result = value.call(this, ...args)
    console.log(`[${context.name}] result:`, result)
    return result
  }
}

class Mathy {
  @logged
  add(a: number, b: number) {
    return a + b
  }
}
```

Field example — clamp negative values to zero via an initializer:

```ts
function nonNegative(_value: unknown, context: ClassFieldDecoratorContext) {
  if (context.kind !== 'field') return undefined;
  return function (initialValue) {
    return Math.max(0, initialValue)
  }
}

class Wallet {
  @nonNegative
  balance = -5 // becomes 0
}
```

The mental model stays the same as the HOF: take a value, return a wrapped value or an initializer. The only new thing is the `context` object that tells you what you’re decorating.

## TypeScript’s experimental decorators (legacy)

Before the standard landed, TypeScript had “experimental” decorators. They look similar in code, but the API is different and descriptor‑based.

```ts
// Legacy/experimental method decorator in TypeScript
function loggedLegacy(_target: any, key: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value
  descriptor.value = function (...args: unknown[]) {
    console.log(`[${key}] args:`, args)
    const result = original.call(this, ...args)
    console.log(`[${key}] result:`, result)
    return result
  }
  return descriptor
}

class Mathy {
  @loggedLegacy
  add(a: number, b: number) {
    return a + b
  }
}
```

Notes worth knowing:
- API shape is `(target, key, descriptor)`.
- You mutate/return a `PropertyDescriptor`.
- Parameter decorators exist here (they do not in the standard).
- Many libs used `emitDecoratorMetadata` + `reflect-metadata` to tack on runtime type info.

### What changed vs the standard?

- Standard: `(value, context)` and you return a replacement/initializer.
- Legacy: `(target, key, descriptor)` and you return a descriptor.
- Parameter decorators only exist in legacy.
- Metadata via `emitDecoratorMetadata` is legacy‑oriented; the standard doesn’t bake it in.

## One decorator, both environments

You can keep one source of truth and adapt it to both APIs. The trick: write the core as a tiny wrapper factory, then expose two thin adapters.

```ts
// Core logic — pure HOF
function wrapLogged(fn: Function, name: string) {
  return function (this: any, ...args: unknown[]) {
    console.log(`[${name}] args:`, args)
    const result = fn.call(this, ...args)
    console.log(`[${name}] result:`, result)
    return result
  }
}

// Standard decorator adapter
export function logged(value: any, context: ClassMethodDecoratorContext) {
  const name = String(context.name)
  return wrapLogged(value, name)
}

// Legacy decorator adapter
export function loggedLegacy(_target: any, key: string, descriptor: PropertyDescriptor) {
  descriptor.value = wrapLogged(descriptor.value, key)
  return descriptor
}
```

Use `@logged` when you compile for the standard decorator pipeline. Use `@loggedLegacy` in older TS setups that still rely on the experimental pipeline. Same behavior, one implementation.

## Migration tips (legacy → standard)

- Replace descriptor logic with return‑a‑function (for methods) or initializer objects (for fields).
- Remove reliance on `emitDecoratorMetadata`; consider explicit schemas or runtime validators instead.
- Re‑think parameter decorators. Push validation to the method body or use explicit factories/HOFs.
- Test behavior around class fields — the standard model’s initializers often simplify old patterns.

## Function decorators (ECMAScript proposal)

Class element decorators are on the standards track and usable today via TS/Babel transforms; engine support remains spotty. Decorating standalone functions (e.g. `@memoize function fib…`) is a separate proposal. Expect similar `(value, context)` ergonomics, but don’t ship code that depends on it yet. Until then, use HOFs:

```ts
function memoize(fn) {
  const cache = new Map()
  return function (...args) {
    const key = JSON.stringify(args)
    if (cache.has(key)) return cache.get(key)
    const result = fn.call(this, ...args)
    cache.set(key, result)
    return result
  }
}

const fib = memoize(function fib(n) {
  return n <= 1 ? n : fib(n - 1) + fib(n - 2)
})
```

## When to use decorators

- Cross‑cutting concerns: logging, tracing, timing.
- Contract enforcement: invariants, validate inputs, clamp ranges.
- Composition: capability flags, auto‑binding, dependency injection (with care).

If the logic reads better “next to” the class member than above/below it, a decorator can improve clarity.

## Takeaway

This isn’t magic. It’s the HOF/HOC pattern, moved to class definition time. Learn the standard model first, then translate your legacy decorators, and keep a single core so you don’t maintain two behaviors. Function decorators will likely join the party—until then, reach for HOFs.

Missing something specific from your codebase? Point me to it, and I’ll add a concrete migration.
