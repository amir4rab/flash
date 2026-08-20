# 02 — TypeScript

This document defines the TypeScript rules every change must follow. The goal is a codebase where the type system catches as many errors as possible at compile time.

## Non-negotiable rules

1. **`any` is banned.** Never use `any`. If you are tempted to use it, the type is not yet understood — go understand it.
2. **No implicit `any`.** Every function parameter, return value, and variable is explicitly typed or inferred from an explicitly typed source.
3. **No untyped `as` casts.** `as` is only allowed when the cast is provably safe and the reason is documented in a comment. Prefer narrowing, discriminated unions, or type guards.
4. **Explicit return types on exported functions.** Every exported function declares its return type. This keeps the public API surface of a module readable and stable.
5. **Strict compiler settings.** The `strict` flag and its family are always on. See the tsconfig section below.

## Replacing `any`

| Situation | Use instead |
| --- | --- |
| Unknown external input (API payload, JSON, user input) | `unknown` + runtime validation with Zod, then narrow |
| Function that accepts multiple types | Generics |
| A value with a specific meaning (e.g. an ID) | Branded types |
| A value that is one of several known shapes | Discriminated unions |
| Optional data | `T \| undefined` with explicit handling |

### Example: `unknown` + narrowing

```ts
import { z } from "zod";

const UserPayload = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
});

type UserPayload = z.infer<typeof UserPayload>;

export function parseUser(raw: unknown): UserPayload {
  return UserPayload.parse(raw); // throws a typed error on invalid input
}
```

### Example: branded type

```ts
export type UserId = string & { readonly __brand: "UserId" };

export function toUserId(value: string): UserId {
  return value as UserId; // single, documented cast at the boundary
}
```

### Example: discriminated union

```ts
export type ApiResult<T> =
  | { readonly kind: "success"; readonly data: T }
  | { readonly kind: "error"; readonly error: ApiError };

export function handle(result: ApiResult<number>): void {
  if (result.kind === "success") {
    // result.data is number
  } else {
    // result.error is ApiError
  }
}
```

## Interfaces and types

- **Prefer `interface` for object shapes** that are extended or implemented; prefer `type` for unions, intersections, and mapped types.
- **Every interface is explicitly declared.** No structural types written inline for public boundaries.
- **Use `readonly`** on properties that must not change after construction.
- **Use `satisfies`** instead of `as` when you want to check a value against a type without widening it.
- **`exactOptionalPropertyTypes` is on.** Optional properties are `prop?: T`, never `prop?: T | undefined`.

## tsconfig baseline

Every package extends a shared base config. The base config sets:

```jsonc
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "noImplicitReturns": true,
    "noUncheckedIndexedAccess": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "forceConsistentCasingInFileNames": true,
    "verbatimModuleSyntax": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "target": "es2022",
    "lib": ["es2022"],
    "skipLibCheck": true,
    "isolatedModules": true
  }
}
```

Notes:

- `noUncheckedIndexedAccess` forces handling of `undefined` when indexing arrays and records — do not disable it.
- `verbatimModuleSyntax` requires `import type` for type-only imports. Use it consistently.
- Framework-specific settings (JSX, DOM libs) are added in the app's own tsconfig, never in the shared base.

## Type-only imports

Use `import type` for anything that is only a type:

```ts
import type { User } from "./user";
import { toUserId } from "./user";
```

## Generics

- Use generics to preserve type information across functions and components.
- Constrain generics with `extends`; never default to `any`.
- Name type parameters with a single uppercase letter or a descriptive `T`-prefixed name (`TData`, `TError`).

## Error handling

- Errors are typed. Define an `ApiError` shape (see `docs/04-api-design.md`) and use it consistently.
- Do not catch and swallow errors. If a catch is needed, narrow the caught value (`unknown`) and rethrow or handle explicitly.
- Use `Result`-style discriminated unions for fallible operations that are not exceptions.

## What to avoid

- `any`, `unknown` used as a permanent escape hatch, `@ts-ignore`, `@ts-expect-error` without justification.
- Casting through `as unknown as X`.
- Optional chaining used to hide missing data instead of handling it explicitly.
- Non-null assertion `!` on values that are not provably present.
