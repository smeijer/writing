---
description: Preserve meaningful error messages in production, stop using tools that strip them.
tags: code
date: 2025-11-12
draft: false
---

# Keep Your Error Messages

Some developers still run tools that strip their own error messages in production. That practice makes no sense. Good error messages aren't a debugging luxury, they're a fundamental part of your system's observability. Yet there are libraries that still throw `"Invariant failed"` instead of the message you wrote. Let's look at why that happens and how to avoid it.

## When helpers hide the message

Assertion helpers like [`tiny-invariant`](https://github.com/alexreardon/tiny-invariant) make it easy to check assumptions and satisfy TypeScript in a single line. They're small, popular, and convenient, the kind of utility you'd drop into any project without thinking twice.

But here's the catch: in production, `tiny-invariant` replaces your carefully written error message with the generic `"Invariant failed"`. Some bundler setups even remove the message entirely to shave a few bytes off the bundle.

That trade-off sounds clever until you need to debug a production issue. Your logs end up full of identical `"Invariant failed"` errors with no clue what actually failed. You save a handful of bytes and lose hours of context.

`tiny-invariant` isn't unique here, it's just the best-known example of a larger problem: tools that optimize away information that's critical when things go wrong. Assertions exist to explain failures, not hide them.

## Why that's wrong

An error message isn't optional. It tells you what failed, where it happened, and what to look at next. Removing it makes your logs useless. Several megabytes of JavaScript is fine, but you're trying to save a dozen characters of text? Network speeds and compression have made byte-counting on error strings irrelevant for years. You don't need to optimize that away.

## Do it properly

If you want a one-liner assertion that narrows types, just write one. TypeScript has an [`asserts` syntax](/articles/typescript-type-assertions/) for this (also see the [TypeScript handbook](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)). Here's a simple helper:

```ts
function ensure<T>(
  value: T | null | undefined, 
  message: string,
): asserts value is T {
  if (value == null) return;
  throw new Error(message);
}
```

You can use it like this:

```ts
function getUserName(user?: { name: string }) {
  ensure(user, 'User must be provided');
  // Here TypeScript knows that user is defined
  return user.name.toUpperCase();
}
```

It narrows the type like `tiny-invariant` does, but it never strips the message. In production, you get `Error: User must be provided`, which is exactly what you want.

## Keep your messages

Error messages have always been part of good code. Don't let your tools throw them away. Write assertion functions that preserve the message and narrow types. If you don't want to roll your own, check out [`ensured`](/projects/ensured/) or [install it from npm](https://npmjs.com/ensured). It provides the same assertion functionality and keeps your messages intact in production.

Discarding error text to save bytes is a relic of the past. Keep the information you need.