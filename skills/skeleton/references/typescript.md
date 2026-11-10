# TypeScript / JavaScript

Write TypeScript. For a JavaScript-only codebase, ask before choosing
between TypeScript and JSDoc-typed JavaScript checked with
`tsc --checkJs`.

## Skeleton encoding

- **Placeholder.** The body is exactly
  `throw new Error("skeleton: not implemented");`. It typechecks for
  any return type, because `throw` has type `never`.
- **Contracts.** A `/** */` comment on the declaration, with the
  sections from SKILL.md.
- **Module description.** A one-line `/** */` comment at the top of
  each module's `index.ts`.
- **Type check:** `npx tsc --noEmit`.
- **Remaining stubs:** `grep -rn "skeleton: not implemented" src | wc -l`
- Lint runs on filled code. Unused-parameter warnings on stubs are
  expected; don't weaken the lint config to silence them.

This example only illustrates the format. Don't copy its names or its
domain.

```ts
// file: orders/types.ts
type Brand<T, B extends string> = T & { readonly __brand: B };
export type OrderId = Brand<string, "OrderId">;
export type Cents = Brand<number, "Cents">; // integer, >= 0
export type NonEmpty<T> = readonly [T, ...T[]];
export type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

export type Line = { readonly sku: string; readonly qty: number; readonly unit: Cents };

export type Draft = { readonly kind: "draft"; readonly id: OrderId; readonly lines: readonly Line[] };
export type Placed = {
  readonly kind: "placed";
  readonly id: OrderId;
  readonly lines: NonEmpty<Line>;
  readonly total: Cents;
};
export type Cancelled = { readonly kind: "cancelled"; readonly id: OrderId; readonly reason: string };
export type Order = Draft | Placed | Cancelled;

// file: orders/transitions.ts
import type { Cancelled, Draft, Placed, Result } from "./types.js";

/**
 * Ensures:
 * - WHEN draft.lines is empty THEN THE CALLABLE SHALL return { ok: false, error: "empty" }.
 * - WHEN draft.lines is non-empty THEN THE CALLABLE SHALL return Placed with the
 *   same id and lines, and total equal to the sum of qty × unit.
 * Invariants: I1
 */
export function place(draft: Draft): Result<Placed, "empty"> {
  throw new Error("skeleton: not implemented");
}

/**
 * Ensures: THE CALLABLE SHALL return Cancelled with the same id and the given reason.
 * Invariants: I1
 */
export function cancel(order: Draft | Placed, reason: string): Cancelled {
  throw new Error("skeleton: not implemented");
}
```

`cancel` accepts only `Draft | Placed`, so "a cancelled order can't be
cancelled again" needs no contract clause. Purity (no I/O) is stated
once, as an invariant over the whole module, rather than on every
function.

## Extraction commands

- **Signatures:** `npx tsc --declaration --emitDeclarationOnly --outDir <scratch>`
  (add `--allowJs` for JavaScript).
- **Dependency graph and cycles:** `npx madge --circular --extensions ts src`,
  and `npx madge --json --extensions ts src` for the full graph.
- **Complexity hotspots:**
  `lizard -l typescript -C 10 -w -s cyclomatic_complexity src`
  (install with `pip install lizard`).
- **Churn:**
  `git log --since=1.year --format= --name-only -- src | sort | uniq -c | sort -rn | head -40`

## Compiler

`tsconfig.json`, at minimum:
- `"strict": true`
- `"noUncheckedIndexedAccess": true`
- `"exactOptionalPropertyTypes": true`
- `"noImplicitOverride": true`
- `"noFallthroughCasesInSwitch": true`
- `"verbatimModuleSyntax": true`
- `"noPropertyAccessFromIndexSignature": true`

## Lint

typescript-eslint with type-aware rules, including:
- `@typescript-eslint/switch-exhaustiveness-check`
- `@typescript-eslint/no-floating-promises`
- `@typescript-eslint/no-explicit-any`
- `@typescript-eslint/strict-boolean-expressions`
- Boundary rules that forbid importing another module's internal
  files: `no-restricted-imports` patterns, or `eslint-plugin-boundaries`
  when the map has more than a few modules.

## Modules and privacy

- One directory per module, with an `index.ts` that exports the public
  surface. Everything else is imported only from inside the directory,
  and the lint boundary rules enforce that.
- In a monorepo, each module is a workspace package whose
  `package.json` `exports` field lists only its public entry points;
  Node then refuses deep imports.

## State machines

Discriminated unions on a literal `kind` (or `status`) field, one
member per state carrying only that state's data. Transitions are
functions typed `(s: Pending, ...) => Approved | Rejected`. Exhaustive
`switch` with a `never` default:

```ts
default: { const unreachable: never = s; throw new Error(String(unreachable)); }
```

## Parsing and errors

- Parse at the edge with a schema library (Zod, Valibot) or
  hand-written parsers returning a result type. Domain types are plain
  types, not schema-inferred types that leak a library into the
  domain; convert at the edge.
- Expected failures are values: a `Result`-shaped discriminated union
  (`{ ok: true, value } | { ok: false, error }`) defined once in the
  codebase, or a small library if the user prefers. Throw only for
  bugs and broken infrastructure.
- Brand primitive IDs (`type UserId = string & { readonly __brand: "UserId" }`)
  when two kinds of ID could be swapped.

## Effects and wiring

Ports are TypeScript `interface`s or function types. The composition
root constructs adapters and passes them as arguments or a typed
context object. No DI decorators or reflection-based containers.

## Tests (for implementers)

Vitest or `node:test`. Fakes are classes or closures implementing the
port interface. Don't add `vi.mock`-style module mocking for
in-process code.
