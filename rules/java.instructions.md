---
applyTo: "**/*.java"
---
These rules apply the global "Modeling", "Software Engineering Style" and
"Design Judgement" sections to Java. The snippets show shape, not a
template: reuse the repo's existing types and helpers rather than copying
them, and don't create new ones where equivalents exist.

Check the repo's Java level (pom or Gradle toolchain) first. Records need
16+, sealed types 17+, pattern-matching `switch` 21+. Below that, fall back
to an enum with a `switch` expression.

## Modeling

- Parse input once, at the boundary that owns it, into records, enums, or
  sealed types. Don't pass `null` inward. Use `Optional` for return values
  only, never for fields or parameters.
- Expected parse failures (bad user or wire input) are a sealed result
  value, not an exception. Reserve exceptions for broken invariants and
  infrastructure failures.

```java
sealed interface Parsed<T> {
  record Ok<T>(T value) implements Parsed<T> {}
  record Invalid<T>(String reason) implements Parsed<T> {}
}
static Parsed<Quantity> parse(String raw) { ... }
```

- A value with several shapes is a sealed interface of records, not one
  class with nullable fields or a status flag.

```java
// avoid: which fields are valid in which status?
class Order { Status status; Instant submittedAt; Receipt receipt; }

// prefer
sealed interface Order permits Draft, Submitted, Paid {}
record Draft(OrderId id, List<Line> lines) implements Order {}
record Submitted(OrderId id, List<Line> lines, Instant at) implements Order {}
record Paid(OrderId id, Receipt receipt) implements Order {}
```

- When operations are valid only in some states, each state is its own
  record and each transition is a function from one state to the next.

```java
static Submitted submit(Draft d, Instant at) { return new Submitted(d.id(), d.lines(), at); }
static Paid pay(Submitted s, Receipt r) { return new Paid(s.id(), r); }
```

- Match sealed types and enums with a `switch` expression and no
  `default`, so a new variant fails to compile. `default ->` or
  `instanceof` chains defeat the check.

```java
String label(Order o) {
  return switch (o) {
    case Draft d -> "draft";
    case Submitted s -> "submitted";
    case Paid p -> "paid";
  };
}
```

- Give distinct domain concepts distinct types when mixing them up would
  be a bug. Enforce invariants in the compact constructor.

```java
record OrderId(UUID value) {}
record Quantity(int value) { Quantity { if (value <= 0) throw new IllegalArgumentException(); } }
```

- Use an enum for a closed set with no per-variant data; a sealed
  interface once variants carry data.
- Don't encode distinctions the code never acts on. A wrapper record that
  a serializer or JPA must map needs its mapping verified first.
- Avoid: boolean-flag combinations, builders where every field is
  optional, `Optional` fields or parameters.

## Software Engineering Style

- Use guard clauses and early returns; a plain `else` is fine for two
  symmetric branches. Write the happy path as straight-line flow.

```java
// avoid
if (o != null) { if (o.isPaid()) { ship(o); } }

// prefer
if (o == null || !o.isPaid()) return;
ship(o);
```

- Extract a helper only for multi-step validation, bulky detail, or when
  the name clarifies the caller. Otherwise inline it. Define helpers next
  to their one caller.
- Prefer streams (map, filter, flatMap, collect) for pure transforms and
  collection building. Use a `for` loop when the body has side effects,
  throws checked exceptions, mutates local state, or needs an early exit,
  and in hot paths or code over large collections.
- No side effects in stream lambdas. Collect results; don't build a list
  inside `forEach`.
- Use `Stream.toList()` when an unmodifiable list is acceptable.
  Otherwise use Collectors.

## Design Judgement

- Hardening at trust boundaries is not speculative; do it: parameterized
  queries (`PreparedStatement`, never string-built SQL), normalize and
  containment-check paths before use, no secrets in logs.
- Adding a dependency is speculative complexity. Ask first.
