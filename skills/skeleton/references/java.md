# Java

Target the current LTS JDK (21 or later), so sealed types, records and
pattern-matching `switch` are available.

## Skeleton encoding

- **Placeholder.** The body is exactly
  `throw new UnsupportedOperationException("skeleton: not implemented");`
- **Contracts.** Javadoc on the declaration, with the sections from
  SKILL.md. On JDK 23 or later, Markdown doc comments (`///`, JEP 467)
  read better; either works.
- **Module description.** A one-line `package-info.java` Javadoc in
  each module's root package.
- **Type check:** `./gradlew compileJava` or `mvn -q compile`.
- **Remaining stubs:** `grep -rn "skeleton: not implemented" src/main | wc -l`

This example only illustrates the format. Don't copy its names or its
domain.

```java
// file: orders/OrderId.java
package orders;

public record OrderId(String value) {}

// file: orders/Line.java
package orders;

public record Line(String sku, int qty, long unitCents) {}

// file: orders/Order.java
package orders;

import java.util.List;

/** An order's lifecycle. Each state carries only its own data. */
public sealed interface Order permits Order.Draft, Order.Placed, Order.Cancelled {
    record Draft(OrderId id, List<Line> lines) implements Order {}
    record Placed(OrderId id, List<Line> lines, long totalCents) implements Order {}
    record Cancelled(OrderId id, String reason) implements Order {}
}

// file: orders/PlaceOutcome.java
package orders;

public sealed interface PlaceOutcome permits PlaceOutcome.Placed, PlaceOutcome.Empty {
    record Placed(Order.Placed order) implements PlaceOutcome {}
    record Empty() implements PlaceOutcome {}
}

// file: orders/Transitions.java
package orders;

public final class Transitions {
    private Transitions() {}

    /**
     * Ensures:
     * - WHEN draft.lines() is empty THEN THE CALLABLE SHALL return PlaceOutcome.Empty.
     * - WHEN draft.lines() is non-empty THEN THE CALLABLE SHALL return PlaceOutcome.Placed
     *   with the same id and lines, and totalCents equal to the sum of qty × unitCents.
     * Invariants: I1
     */
    public static PlaceOutcome place(Order.Draft draft) {
        throw new UnsupportedOperationException("skeleton: not implemented");
    }
}
```

The outcome is a sealed type per operation, so callers must handle
`Empty` in an exhaustive `switch`. Purity (no I/O) is stated once, as
an invariant over the whole class, rather than on every method.

## Extraction commands

- **Signatures:** `javap -p -cp build/classes/java/main <class>`
  (Maven: `target/classes`).
- **Dependency graph:** `jdeps -verbose:package build/classes/java/main`;
  `-verbose:class` for class level.
- **Cycles:** an ArchUnit test with
  `slices().matching("<root>.(*)..").should().beFreeOfCycles()`.
- **Complexity hotspots:**
  `lizard -l java -C 10 -w -s cyclomatic_complexity src/main/java`
  (install with `pip install lizard`).
- **Churn:**
  `git log --since=1.year --format= --name-only -- src/main | sort | uniq -c | sort -rn | head -40`

## Build

Gradle (Kotlin DSL) or Maven, whichever the user prefers; ask if
unstated. Compile with `-Xlint:all -Werror`.

## Static checks

- Error Prone, with NullAway configured for the project's packages, so
  nullness is checked at compile time. Annotate with JSpecify
  (`@NullMarked` on each package, `@Nullable` where absence is
  allowed).
- ArchUnit tests encoding the module map: no cycles between modules,
  domain packages import nothing from adapter packages or frameworks,
  and modules use only each other's public package.

## Modules and privacy

- One top-level package per module. Its public surface lives in the
  module's root package; everything else is package-private, which is
  the default when no modifier is written. Don't mark a class `public`
  unless another module needs it.
- Use JPMS (`module-info.java`) or Gradle subprojects when the map has
  enough modules that package-private alone isn't enough; `exports`
  then lists only public packages.

## State machines

A `sealed interface` with one `record` per state, each carrying only
that state's data. Transitions are static methods or methods on the
source state returning the target type. Exhaustive pattern-matching
`switch` expressions with no `default` branch, so adding a state
breaks the build at every unhandled site.

## Parsing and errors

- Parse at the edge into records with validating compact constructors
  or static `parse` factories. Domain types never carry framework
  annotations (JPA, Jackson); map at the adapter.
- Expected failures are values: a sealed result type defined once in
  the codebase (`sealed interface Result<T, E> permits Ok, Err`) or a
  domain-specific sealed outcome per operation. Checked exceptions are
  not the error model. Throw unchecked exceptions only for bugs and
  broken infrastructure.
- `Optional` for return values that may be absent; never as a field or
  parameter.

## Effects and wiring

Ports are interfaces in the domain module. The composition root is
`main`, constructing adapters with `new` and passing them to
constructors. If a framework requires Spring or similar, confine its
annotations and configuration to the composition root and adapters;
domain classes stay plain.

## Tests (for implementers)

JUnit 5 with AssertJ. Fakes are hand-written classes implementing the
port. Don't add Mockito unless a third-party API makes a fake
impractical, and never to mock in-process domain code.
