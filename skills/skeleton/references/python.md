# Python

Target Python 3.12 or later. Use uv for the project, environment and
lockfile unless the user prefers otherwise.

## Skeleton encoding

- **Placeholder.** The body is exactly
  `raise NotImplementedError("skeleton: not implemented")`, after the
  docstring.
- **Where.** Write the skeleton in `.py` files, not `.pyi` stubs, so the
  contracts stay in the code that gets filled.
- **Contracts.** The function's docstring, with the sections from
  SKILL.md.
- **Module description.** A one-line docstring in each package's
  `__init__.py`.
- **Type check:** `pyright` (strict).
- **Remaining stubs:** `grep -rn "skeleton: not implemented" src | wc -l`

This example only illustrates the format. Don't copy its names or its
domain.

```python
# file: orders/types.py
from dataclasses import dataclass
from typing import NewType

OrderId = NewType("OrderId", str)
Cents = NewType("Cents", int)  # >= 0


@dataclass(frozen=True, slots=True)
class Line:
    sku: str
    qty: int
    unit: Cents


@dataclass(frozen=True, slots=True)
class Draft:
    id: OrderId
    lines: tuple[Line, ...]


@dataclass(frozen=True, slots=True)
class Placed:
    id: OrderId
    lines: tuple[Line, *tuple[Line, ...]]  # non-empty
    total: Cents


@dataclass(frozen=True, slots=True)
class Cancelled:
    id: OrderId
    reason: str


@dataclass(frozen=True, slots=True)
class EmptyOrder:
    """Placing failed because the draft had no lines."""


type Order = Draft | Placed | Cancelled

# file: orders/transitions.py
from orders.types import Cancelled, Draft, EmptyOrder, Placed


def place(draft: Draft) -> Placed | EmptyOrder:
    """
    Ensures:
    - WHEN draft.lines is empty THEN THE CALLABLE SHALL return EmptyOrder().
    - WHEN draft.lines is non-empty THEN THE CALLABLE SHALL return Placed with the
      same id and lines, and total equal to the sum of qty * unit.
    Invariants: I1
    """
    raise NotImplementedError("skeleton: not implemented")


def cancel(order: Draft | Placed, reason: str) -> Cancelled:
    """
    Ensures: THE CALLABLE SHALL return Cancelled with the same id and the given reason.
    Invariants: I1
    """
    raise NotImplementedError("skeleton: not implemented")
```

`cancel` accepts only `Draft | Placed`, so "a cancelled order can't be
cancelled again" needs no contract clause, as long as pyright runs.
Python enforces none of this at runtime, so contracts carry more
weight here than in TypeScript or Java.

## Extraction commands

- **Signatures:** `stubgen src/<package> -o <scratch>` (ships with mypy;
  the path form works without importing the package).
- **Dependency graph and cycles:**
  `pydeps <package> --show-deps --no-output` for the graph, and
  `pydeps <package> --show-cycles --no-output` for cycles.
- **Complexity hotspots:**
  `lizard -l python -C 10 -w -s cyclomatic_complexity src`
  (install with `pip install lizard`).
- **Churn:**
  `git log --since=1.year --format= --name-only -- src | sort | uniq -c | sort -rn | head -40`

## Type checking

pyright (or basedpyright) in `strict` mode over all source and tests,
run in CI. mypy with `--strict` is acceptable if the user prefers it.

## Lint and format

Ruff for both, with at least the `ANN`, `B`, `RET`, `SIM`, `PL` and
`TID` rule sets. Use `flake8-tidy-imports` banned-api rules (under
`TID`) or import-linter contracts to encode the module map: no
cycles, domain packages import nothing from adapters, and modules
import only each other's public names.

## Modules and privacy

- One package per module. Its `__init__.py` re-exports the public
  surface and sets `__all__`; internal modules and names start with
  `_`. Python can't enforce privacy itself, so the import-linter or
  banned-api contract is what enforces it.
- Use a `src/` layout so tests import the installed package, not the
  working directory.

## State machines

One frozen dataclass (`@dataclass(frozen=True, slots=True)`) per
state, each carrying only that state's data, joined in a type alias
(`type Order = Pending | Paid | Shipped`). Transitions are functions
annotated with their source and target types. Exhaustive `match` with
a final `case _: assert_never(x)` from `typing`, so pyright flags any
unhandled state.

## Parsing and errors

- Parse at the edge into frozen dataclasses with hand-written parse
  functions, or Pydantic models confined to the adapter and converted
  into domain dataclasses. Domain code never sees `dict[str, Any]` or
  Pydantic models.
- Expected failures are values: a small result union defined once
  (`type Result[T, E] = Ok[T] | Err[E]`) or a domain-specific union per
  operation. Raise exceptions only for bugs and broken infrastructure.
- `NewType` for primitive IDs that could be swapped.

## Effects and wiring

Ports are `typing.Protocol` classes in the domain package. The
composition root constructs adapters and passes them as arguments. No
DI frameworks or module-level singletons; read the environment once
at startup into a frozen settings dataclass.

## Tests (for implementers)

pytest. Fakes are plain classes satisfying the port `Protocol`. Don't
use `unittest.mock.patch` on in-process code; fixtures build the real
domain objects and fakes.
