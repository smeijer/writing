---
description: "`accessor` gives you a property with an auto‑generated getter/setter and a private backing field. Cleaner than fields, lighter than hand‑written accessors, and ready for decorators."
tags: code
date: 2025-10-29
draft: true
---

# Auto accessors in TypeScript — less boilerplate, real properties

TypeScript 4.9 added auto accessors. You write `accessor name: string`, and TypeScript gives you a real property: a getter, a setter, and a private backing field. Same usage as a field at the call site, but more capability.

If you’ve ever written a “field + private slot + manual get/set” for cleanliness or future‑proofing, this replaces it.

## Fields vs accessors

At the call site both read the same:

```ts
class A { name = 'Ada' }          // field
class B { accessor name = 'Ada' }  // accessor

new A().name // 'Ada'
new B().name // 'Ada'
```

The differences are about control and evolution:

- Interception: a field can’t intercept reads/writes. An accessor can be wrapped (now or later) to validate, normalize, log, or notify.
- Encapsulation: accessors hide storage behind a getter/setter; callers don’t write the storage directly.
- Evolvability: start simple today, add behavior tomorrow without changing `user.name` everywhere.
- Subclassing: subclasses can override an accessor to extend behavior; you can’t “override” a field to intercept writes.

If you truly never need those hooks, a field is fine. Accessors buy you options with almost no extra code.

## The idea

Declare an accessor and use it like a normal property.

```ts
class Person {
  accessor name: string

  constructor(name: string) {
    this.name = name
  }
}

const p = new Person('Ada')
console.log(p.name) // Ada
p.name = 'Grace'
```

## What TypeScript emits

Conceptually, this is what the compiler creates:

```ts
class Person {
  #name
  get name() { return this.#name }
  set name(v) { this.#name = v }

  constructor(name) { this.name = name }
}
```

You still read/write `person.name`. The storage is private, the API stays stable.

## Types, visibility, and initializers

You can annotate and initialize like any field. Visibility modifiers affect type‑checking access.

```ts
class Counter {
  accessor value: number = 0
}

class Session {
  private accessor token!: string
  getToken() { return this.token }
}
```

## When to reach for it

- Replace boilerplate “get/set + private field” that only forwards.
- Keep the option to add validation/logging later without changing usage.
- Hide storage details while keeping a clean instance API.

If you need custom logic in the getter/setter today, you can still hand‑write them. Auto accessors exist to remove boilerplate, not to block control.

## Decorators play nicely

Because there’s a real `get`/`set` under the hood, standard decorators can wrap reads/writes without refactors. Example: trim on write and warn on empty.

```ts
function trimmed(value: { set: (v: string) => void }, context: ClassAccessorDecoratorContext) {
  return {
    set(this: any, v: string) {
      const next = v.trim()
      if (next.length === 0) console.warn(`${String(context.name)} set to empty`)
      value.set.call(this, next)
    },
  }
}

class Person {
  @trimmed
  accessor name = ''
}
```

If decorators are your thing, I explain them in more detail here: /articles/decorators-explained/

## Takeaway

Auto accessors give you field‑level ergonomics with property‑level control. Use `accessor` for properties you care about, drop to manual getters/setters only when you need custom bodies. Smaller classes, cleaner APIs, future‑proof for decorators.
