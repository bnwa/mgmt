---
name: skeleton
description: Design or extract a code skeleton with the user, one increment per session. A skeleton is types, module boundaries, signatures, contracts and invariants, with placeholder bodies. Use when asked to design architecture, start or scaffold a codebase, or map an existing codebase's structure before changing it. Not for implementing, fixing or reviewing code.
---

# Code skeleton

A skeleton states the decisions an implementer must not get to make on
its own: data types, module boundaries, which modules may call which,
function signatures, and the contracts and invariants they satisfy.
Bodies are placeholders. Tests and examples can't pin these decisions
down, because how many tests a program needs depends on a structure
the tests can't see. A skeleton states the structure directly, in the
language the code will be written in, so the compiler checks it from
the first commit. (Basis: Golatkar et al., "Skeleton Coding: Program
Structure as a Specification for Agents", arXiv:2610.38214, 2026.)

The skeleton is written with the user, not for them. You recover or
decide decisions together; you never guess them. A skeleton the user
hasn't signed off is a proposal.

## What a skeleton contains

- **Layout.** The module tree. Each module carries a one-line
  description of what it owns, in its entry file (see the language
  reference for where).
- **Types.** Domain types, state machines as sum types, the parsed
  form of each external input, branded or wrapped IDs. Most decisions
  live here.
- **Signatures.** Each module's public callables, and one port per
  external system, each with a contract in its doc comment.
- **Invariants.** `INVARIANTS.md` at the repository root holds rules
  that span several callables. Each entry names its participants, and
  each participant cites the entry back by ID.
- **Placeholder bodies.** Every body is exactly the language's
  placeholder, so remaining stubs can be counted with grep.

It leaves out private helpers, the internals of any module not yet
designed, and anything whose shape the user hasn't decided. Partial is
the default. An unstated region is free for the implementer, which is
right whenever the decision doesn't matter yet.

## Contracts

Write contracts in EARS phrasing (Mavin et al., "Easy Approach to
Requirements Syntax", 2009):

```
Requires: THE CALLER SHALL <precondition>.
Ensures:
- WHEN <condition> THEN THE CALLABLE SHALL <outcome>.
- THE CALLABLE SHALL NOT <forbidden effect>.
Invariants: I<n>, I<m>
Debt: <intended shape, and why the code differs>
```

`Debt:` appears only in extraction (see `references/extraction.md`).

An `INVARIANTS.md` entry uses the same phrasing:

```
## I1 — Order transitions are pure
THE CALLABLES SHALL NOT perform I/O, read the clock, or mutate their arguments.
Participants: orders/transitions: place, cancel
```

- **Types first.** Before writing a clause, try to move it into a
  type: a narrower parameter, a sum type, a branded ID. A doc comment
  holds only what the type system can't say. A `Requires:` clause that
  could be a parameter type should become one.
- **Contract where the complexity is.** Complex or widely called
  callables get full contracts. Trivial ones get a signature only.
- **Each component implementable alone.** A contract is complete when
  someone filling that file needs no other file's bodies. Complete
  local contracts are what let separate, cheaper agents fill
  components in parallel.
- **Stop before over-specifying.** Cut clauses that restate the
  signature, describe how instead of what, or pin behavior nobody
  depends on. Extra correct detail can make implementers worse,
  especially small models.
- **One source.** The doc comment is the contract. Don't restate it in
  prose anywhere else.

## One increment per session

Each session adds or amends one set of decisions that can be closed:
usually one layer or one boundary. Tightly coupled decisions, such as
a data model and the port that persists it, belong in the same
increment. Split increments along narrow interfaces.

1. **Load the current state from the repository**, not from earlier
   conversations: the stubs and contracts in code, `INVARIANTS.md`, the
   project `AGENTS.md`, and recent commit messages with their
   `Constraint:` and `Rejected:` trailers. Count the remaining stubs
   (command in the language reference) to see what is still unfilled.
2. **Name the increment.** Propose what this session will decide and
   what it will leave out. Agree on it before writing anything.
3. **Work the mode.** With no source code yet, follow
   `references/greenfield.md`. With existing source code, follow
   `references/extraction.md`. Load `references/<language>.md` for the
   encoding. If no reference exists for the language, apply the same
   intent with that language's equivalents and say which you chose.
4. **Ask; don't guess.** Each question names specific code or a
   specific decision, gives your recommendation, and gives the
   alternative. Ask a few at a time, most consequential first.
5. **Write the increment.** Run the type checker. It must pass.
6. **Present it for sign-off.** Show the diff split into additions and
   amendments. For each amendment to a signed-off signature, contract
   or invariant, name everything that depends on it. List what you
   deliberately left unstated and any question still open.
7. **Commit on sign-off**, with trailers that carry what code can't:

   ```
   skeleton: <increment>

   Constraint: <requirement the user stated> — <reason>
   Rejected: <alternative considered> — <why>
   ```

8. **Hand off and stop.** End with the stubs now ready to fill, in
   dependency order, each as a file path plus the invariants it
   participates in. The next increment starts in a fresh session after
   this one is filled and its tests pass, unless the user says
   otherwise.

If no one is there to answer, stop after step 2 with the proposal
written out. Never write a skeleton on structure the user hasn't
approved.

## Rules

- Write no implementation: no logic, no helpers, no tests. Bodies are
  placeholders only. The one exception: in a new codebase, write the
  build and enforcement config the skeleton needs to compile.
- Don't change a signed-off signature, contract or invariant except as
  a listed amendment.
- In an existing codebase, change no behavior. Adding contracts to doc
  comments and adding new stub files is fine. Moving or rewriting
  existing code is the implementer's job, after the user signs off the
  target shape.
- The user decides what is a requirement. You propose and ask.

## Done

- The type checker passes.
- Every body written this session is exactly the placeholder.
- The diff touches only types, signatures, doc comments, new stub
  files, `INVARIANTS.md`, module descriptions and, in a new codebase,
  build and enforcement config.
- Every invariant lists its participants, and every participant cites
  it.

Report these four checks with the sign-off.
