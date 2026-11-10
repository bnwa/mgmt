# Extraction: recovering a skeleton from existing code

Existing code mixes requirements, accidents and workarounds, and
nothing in the code says which is which. The user holds that
knowledge. Extraction recovers it through questions about specific
code. The mechanical pass is cheap; the classification is the work.

## Scope

Don't skeleton the whole codebase. Each increment covers the region
about to change: the module or boundary the user's next piece of work
touches, plus the interfaces it calls across. Coverage grows where the
work happens.

## 1. Mechanical pass

Gather, using the commands in the language reference:
- The public signatures and types in the region.
- The dependency graph between modules, including cycles.
- Complexity per callable, and change frequency per file from git
  history. High complexity combined with high churn marks a hotspot
  (Tornhill, *Your Code as a Crime Scene*). Start there.
- Disagreements: places where names, doc comments, tests and code
  contradict each other; tests that assert implementation details;
  optional fields standing in for states.

This pass is read-only. It can go to a read-only exploration subagent
if one is available. Give it the region and the list above, and ask
for raw output (paths, signatures, counts), not a summary.

## 2. Classify

For each element the increment covers, propose a class and ask the
user to confirm it:

- **Requirement.** Callers or users depend on it. It goes into the
  skeleton with a contract.
- **Accident.** It's how the code happens to work, and nothing depends
  on it. Leave it out. Omission frees implementers to change it.
- **Debt.** It should be shaped differently. Add a `Debt:` clause to
  its contract stating the intended shape and why. If the target needs
  new types or signatures, declare them in new stub files and point
  the `Debt:` clause at them. The gap between the code and the
  skeleton is the refactor plan.

Ask about specific code, with your proposal. For example:
"`OrderService.apply` silently ignores unknown discount codes. I think
this is debt: callers can't tell a typo from a valid code. Requirement,
accident, or debt?"

Only the user can settle these, so always ask about them:
- whether an observable behavior is relied on outside the codebase;
- whether the cause of a workaround still exists;
- which of two disagreeing sources (a doc and the code, a test and the
  code) is right.

## 3. Record

- Contracts go into the existing doc comments, in place, within the
  repository's doc-comment conventions (for example, keep `@param` and
  `@return` tags). This is the one sanctioned edit to existing
  comments: keep anything the old comment says that the contract
  doesn't, such as a non-obvious why. Don't move or rewrite code.
- Rules that span callables go into `INVARIANTS.md`, with
  participants.
- New target types and signatures go into new stub files with
  placeholder bodies.
- Anything still unclassified at the end of the session stays out of
  the skeleton and is listed as open in the sign-off.

Once a region is extracted, it is in the state greenfield code starts
in. The contracts live in the code, so extracting it again later is
mechanical, and anything without a contract is an accident by
default.
