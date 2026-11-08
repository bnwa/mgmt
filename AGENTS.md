## Modeling
Goal: types should declare the domain's concepts and which
interactions are legal, so misuse fails to compile, not at runtime.
- Parse unknown input once, at the boundary that owns it, into a
  typed value. Pass that value inward; don't re-check it.
- Make illegal states unrepresentable. A value with several
  possible shapes is a sum type (discriminated union, sealed
  hierarchy), not optional fields or flag combinations.
- When an operation is valid only in some states, or the code
  has an implicit state machine (signs: boolean flags that
  combine, fields valid only in some phases, status values
  checked ad hoc across call sites), give each state its own type
  carrying only the data valid in it, and make each transition a
  function from one state to the next (typestate). If the
  language makes that awkward, use an explicit state enum with
  one transition function. A single independent flag stays a flag.
- Match sum types exhaustively. No default branch that silently
  absorbs a new variant.
- Give distinct domain concepts distinct types (branded IDs,
  newtypes, units) when confusing them would be a bug.
- Stop when the type stops paying: don't encode distinctions the
  code never acts on.

## Software Engineering Style
- Use guard clauses and early returns instead of nested else. A
plain else is fine for two symmetric branches.
- Write a function as the happy path: guards first, then
straight-line flow. Move multi-step validation or bulky
supporting detail into small helpers either defined within the
function or as near to the function as possible.
- Don't extract a single-use helper unless it hides a genuinely
complex boundary, is complex enough to test directly, provides
multi-step validation or bulky supporting detail, or has a name
that makes the caller clearer. Otherwise inline it.

## Design Judgement
- Before adding complexity for a speculative case, ask me, with
one line each on the failure mode, likelihood, and cost. If you
can't ask, skip it and list it in your summary. Standard hardening
at trust boundaries (parameterized queries, path checks, no
secrets in logs) or reifying state the code already has isn't
speculative; do it.
- Apply the guidance in the sections 'Modeling', 'Software
Engineering Style', 'Design Judgement', and 'Tests' to code
you're writing or materially changing. Elsewhere, don't fix
pre-existing problems unasked; flag them in your summary.

## Tests
- Test behavior through the public API of the boundary that owns
it: the module, package, or service seam its callers depend on.
A class or function that exists only to serve one caller is an
implementation detail; cover it through that caller.
- Assert on observable outcomes: return values, resulting state,
effects at a real boundary. If a behavior-preserving refactor
would break a test, it's a change detector (mock-and-verify is
the usual form). If your change breaks one, rewrite it against
behavior, or delete it if that behavior is already covered,
and note it in your summary.
- Replace a dependency only where it crosses a process or
ownership boundary (network, clock, filesystem, randomness,
another team's service), and prefer a fake over a mock. Use real
instances for everything in-process.
- Don't test inputs the boundary already rules out. Inner code
receives parsed, typed values; its tests use those types and skip
cases the parser rejects.
- Test an implementation detail directly only when failures
through the public API wouldn't localize the defect: an
algorithm, a parser, a state machine's transition functions. Keep
its visibility as narrow as the language allows (package-private,
module-internal, unexported).
- If existing tests near your change are change detectors, don't
extend the pattern and don't rewrite them unasked; flag them in
your summary.

## Comments
- Comments don't narrate what code does. Comment only what code
can't say: a non-obvious why (constraint, workaround, invariant)
or the how of a genuinely intricate algorithm.
- Never put session content in code: requests, instructions,
review feedback, etc. Change history belongs in the commit
message.
- Leave existing comments alone unless your change invalidates
one; then correct it.
- Doc comments on public API (Javadoc, docstrings, TSDoc, etc)
follow the repo's conventions and state the contract, not the
implementation.

## Git Commits
- Follow the Conventional Commits 1.0.0 standard
(https://www.conventionalcommits.org/en/v1.0.0/).
- For `fix` and `feat` commits, the body should state the
problem, then the change that solves it. Don’t mention tests,
linting, or other routine checks.
