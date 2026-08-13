---
title: The Internal Folder Pattern
description: Separate public APIs from implementation details without TypeScript tricks by adopting an `internal` folder, inspired by Go.
tags: code
date: 2025-10-29
draft: true
---

We leak too much. Utilities seep into places they shouldn't, private helpers get imported across the app, and refactors stall because "something depends on it somewhere". We try to fix it with barrels, `export type`, path aliases, or complex visibility schemes in TypeScript. None of that draws a clear architectural boundary.

Go solved this years ago with one simple rule: anything under an `internal/` directory is private to its parent. Consumers can't import it. We can borrow the idea without gymnastics.

## The boundary: public at root, private in `internal`

Expose a tiny surface at the package or feature root. Put everything else in `internal/`.

```
src/
  user/
    index.ts                 # public API
    internal/
      repository.ts
      mapper.ts
      service.ts

  invoice/
    index.ts
    internal/
      repository.ts
      mapper.ts
```

`src/user/index.ts` is the boundary. It exports what the rest of the app may use. Implementation details live in `src/user/internal/*` and are off-limits to other features.

```ts
// src/user/index.ts
import * as repo from './internal/repository';
import { mapToUser } from './internal/mapper';

export type { User } from './internal/mapper';

export async function createUser(input: unknown) {
  const user = mapToUser(input);
  await repo.insert(user);
  return user;
}
```

Usage stays clean:

```ts
// elsewhere
import { createUser, type User } from '#/user';

await createUser(payload);
```

And this is what you avoid:

```ts
// don't do this
import { mapToUser } from '#/user/internal/mapper';
```

That import might work in plain JavaScript, but it erodes your architecture. Keep all cross-feature imports on the public surface.

## Enforcing without gymnastics

You can keep this a convention. If you want a light guardrail, add one small rule depending on what you’re building.

### Libraries: use `exports`

For published packages, block deep imports by explicitly listing entry points. Anything not exported isn’t reachable.

```json
// package.json
{
  "name": "@acme/domain",
  "type": "module",
  "exports": {
    "./user": "./dist/user/index.js",
    "./invoice": "./dist/invoice/index.js"
  }
}
```

This mirrors Go’s rule at the package boundary: consumers can import `@acme/domain/user`, not `@acme/domain/user/internal/mapper`.

### Apps/monorepos: one ESLint rule

If everything lives in one app, enforce the boundary with `eslint-plugin-internal-boundaries`.

```json
// .eslintrc.json
{
  "plugins": ["internal-boundaries"],
  "rules": {
    "internal-boundaries/no-outside-imports": "error"
  }
}
```

If you use path aliases, add an `import/resolver` setting so the rule can resolve them (via eslint-plugin-import resolvers).

Keep tests next to the code they verify and they won’t trip the rule. If you centralize tests, add a single override to relax the rule for your test directory.

## Why this works

The shape of your code becomes the policy. Index files publish a minimal API. The rest, placed under `internal/`, is free to change. You can refactor, rename, and reorganize internals without a repo-wide fear of breakage. Teams discover the intended surface by looking at `index.ts`, not by grepping for deep imports.

## Takeaway

Adopt the `internal` folder. Export a tiny public surface from `index.ts`. Treat everything else as private. If you ship a library, lock it with `exports`. If you ship an app, add one ESLint rule. No TypeScript magic, no ceremony—just a clear boundary that stays out of your way.
