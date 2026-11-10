# Greenfield: designing a skeleton

Structure drawn on day one is cheap; structure drawn later costs a
migration. Still, design only what this increment needs. Later
increments add the rest, on ground that has been filled and tested.

## Before the first increment

Establish the problem before the structure. Get, from the request or
by asking: what the system does, its inputs and outputs, every
external system it touches (network services, storage, clock,
filesystem, queues), and the language. Ask about gaps; don't fill them
with assumptions.

## The first increment: the outer shell

Propose the following, in this order, short enough to read in one
pass. Wait for sign-off before writing any of it.

- **Module map.** Modules named for capabilities (`billing`, `ingest`,
  `catalog`), never for technical layers (`services`, `models`,
  `controllers`, `utils`). One line of responsibility each.
- **Dependency direction.** Which modules may depend on which. No
  cycles. Domain modules depend on nothing external; adapters depend
  on the domain, never the reverse.
- **Core types.** The parsed form of each external input, the domain
  types, and every state machine with its states and transitions,
  as sum types.
- **Ports.** One interface per external system, phrased in the
  domain's terms (`Invoices.save`, not `PostgresClient.query`). Each
  port gets a stub in-memory fake whose contract is "satisfies the
  port's contract, in memory".
- **Error model.** Which failures are values (parse failures, domain
  rejections, expected absences) and which are exceptions or panics
  (bugs, broken infrastructure). One convention for the whole
  codebase.
- **Walking skeleton.** The signatures along the thinnest end-to-end
  path, from raw input to observable output, including the
  composition root.
- **Left out.** What you are deliberately not deciding yet, one line
  each, with why.

After sign-off, write: the build and enforcement config, the module
layout with its descriptions, the types, the ports and their fake
stubs, the walking-skeleton signatures with contracts, and
`INVARIANTS.md`.

## Later increments

Each later increment widens one capability or descends into one
module's internals. The walking skeleton is filled, with its
end-to-end test passing, before any second capability is designed.

When a new capability doesn't fit the module map, propose a change to
the map as an amendment. Don't bend an existing module to absorb it.

## The shape to aim for

The Modeling rules in your instructions (parse at the boundary, sum
types, typestates, exhaustive matching) apply throughout. These add
the module-level shape:

- **Boundaries.** A module's public surface is small and deliberate.
  Make the language enforce privacy; a naming convention alone is not
  enforcement. No shared `common` or `utils` module at the start: code
  that two modules need goes in the module that owns the concept, or
  waits until a third use shows what the concept is.
- **Immutable values.** Transitions return a new state rather than
  mutating the old one.
- **Effects at the edges.** Domain logic is pure functions over domain
  types. I/O, time, randomness and environment access live in
  adapters behind ports. One composition root constructs the adapters
  and passes them in explicitly. No DI container, service locator or
  global singleton unless a framework forces one, and then only in the
  composition root. Configuration is parsed once at startup into a
  typed value and passed in like any other dependency.

## What implementers will build against it

You don't write tests, but design so that these are possible:
- Every port has a working in-memory fake (a map, a list, a
  controllable clock), not a mock with expectations.
- Tests target each module's public surface, using real domain code
  and fakes for ports. The contracts are what those tests check.
- The walking skeleton gets one end-to-end test through the
  composition root, with fakes wired in.

## Enforcement in config

Encode rules where the next agent can't miss them: strict compiler
settings, exhaustiveness checks, visibility, and import or module
boundary rules. Prefer a failing build over a sentence in a document.
The language reference lists the settings.

## Leave a map

After the first increment, write a short project `AGENTS.md` with only
what code and config can't say:
- The skeleton is authoritative. Implementers fill placeholders and
  don't change signatures, contracts or invariants. A wrong or missing
  contract is reported, not worked around.
- The placeholder string, and the commands for the type check, the
  remaining-stub count and the tests.
- The error-model convention.

Don't restate the module map, the contracts, or anything the tooling
enforces. Those are in the code.
