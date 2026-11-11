---
applyTo: "**/*.ts,**/*.tsx,**/*.mts,**/*.cts,**/*.js,**/*.jsx,**/*.mjs,**/*.cjs"
---
These rules apply the global "Modeling", "Software Engineering Style" and
"Design Judgement" sections to TypeScript and JavaScript. The snippets
show shape, not a template: reuse the repo's existing types and helpers
(`assertNever`, brands) rather than creating new ones.

In JavaScript, express every type-level rule below with JSDoc and have
the compiler enforce it: annotate with `@typedef`, `@type`, `@param` and
`@returns`, and add `// @ts-check` to each file unless the repo's
`jsconfig.json`/`tsconfig.json` already sets `checkJs`. Don't leave
exported functions or parsed values unannotated. Brands and discriminated
unions work through `@typedef` exactly as in TypeScript.

```js
// @ts-check
/** @typedef {string & { readonly __brand: "OrderId" }} OrderId */
/** @typedef {{ ok: true, value: number } | { ok: false, reason: string }} Parsed */

/** @param {unknown} raw @returns {Parsed} */
const parseQuantity = (raw) => { ... };
```

## Modeling

- Take external input as `unknown`, never `any`. Parse it once at the
  boundary into a typed value, then pass that value inward. After parsing,
  no `as` casts or `!` assertions.
- Parse with the repo's existing validator (zod, valibot, ajv, ...). If
  there is none, write a plain `parse(x: unknown)` function; ask before
  adding a dependency.
- Expected parse failures are a discriminated result, not a thrown error.
  Throw only for broken invariants and infrastructure failures.

```ts
type Parsed<T> = { ok: true; value: T } | { ok: false; reason: string };
const parseQuantity = (raw: unknown): Parsed<Quantity> => { ... };
```

- A value with several shapes is a discriminated union, not optional
  fields or flags.

```ts
// avoid: which fields are valid in which status?
type Order = { status: string; submittedAt?: Date; receipt?: Receipt };

// prefer
type Order =
  | { kind: "draft"; id: OrderId; lines: Line[] }
  | { kind: "submitted"; id: OrderId; lines: Line[]; at: Date }
  | { kind: "paid"; id: OrderId; receipt: Receipt };
```

- When operations are valid only in some states, name each state's type
  and make each transition a function from one state to the next.

```ts
type Draft = Extract<Order, { kind: "draft" }>;
type Submitted = Extract<Order, { kind: "submitted" }>;
const submit = (d: Draft, at: Date): Submitted => ({ ...d, kind: "submitted", at });
```

- Match unions exhaustively. A `default` is allowed only to assert
  `never`, so a new variant fails to compile.

```ts
const label = (o: Order): string => {
  switch (o.kind) {
    case "draft": return "draft";
    case "submitted": return "submitted";
    case "paid": return "paid";
    default: return assertNever(o);
  }
};
```

- Give distinct domain concepts distinct types (brands) when mixing them
  up would be a bug. Create a brand only in its parser or constructor,
  never with `as UserId` elsewhere.

```ts
type Brand<T, B extends string> = T & { readonly __brand: B };
type OrderId = Brand<string, "OrderId">;
```

- Use string-literal unions, not `enum`, for closed sets.
- Don't encode distinctions the code never acts on.
- Avoid: `any` (in JSDoc, `*` or `{any}`), `as`/`!` after parsing, boolean-flag combinations,
  optional fields that are valid only in some states.

## Software Engineering Style

- Use guard clauses and early returns; a plain `else` is fine for two
  symmetric branches. Write the happy path as straight-line flow.

```ts
// avoid
if (o) { if (o.kind === "paid") { ship(o); } }

// prefer
if (!o || o.kind !== "paid") return;
ship(o);
```

- Extract a helper only for multi-step validation, bulky detail, or when
  the name clarifies the caller. Otherwise inline it. Define helpers next
  to their one caller.
- Prefer map/filter/flatMap for loops that can be modeled as pure
  transforms. Use `for...of` when the body has side effects, sequential
  `await` invocations, or needs an early exit. Use a `for` loop in
  identified hot paths or over large collections.
- Never use map for side effects. Don't replace sequential `await`
  invocations with `map(async)` unless concurrency is intended.
- Where TypeScript doesn't infer the narrowing from a filter callback,
  add an explicit type guard so downstream types hold.

## Design Judgement

- Hardening at trust boundaries is not speculative; do it: parameterized
  queries, normalize and containment-check paths before use, no secrets in
  logs, no `innerHTML` or `eval` on untrusted input.
- Adding a dependency is speculative complexity. Ask first.
