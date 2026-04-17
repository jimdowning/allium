# Panel review: user journeys (v3)

**Proposal under review:** [`proposals/user-journeys.md`](./user-journeys.md) at SHA `fca0903`
**Protocol:** PROPOSE.md (default disposition: leave the language alone unless the case for change is strong)
**Panel:** nine-member review panel per TEAM.md

---

## Summary

One item was debated: the introduction of a `journey` top-level construct for
declaring actor-scoped outcomes and checking information sufficiency along
a DAG of steps realised by existing rules and surface actions. The debate
established that the problem (specifications that are necessary but not
sufficient) is real, recurring, and unreachable by any existing construct.
The proposed form is principled, with a formal denotation and clear
composition rules. **Verdict: Refine.** Three specific refinements would
close remaining objections: a more domain-facing spelling for parallel
fan-in, an explicit error-message catalogue, and a statement on the
overloading of the `for` keyword. A second cycle against the refined
proposal would very likely adopt.

---

## Item: user journeys

### Proposal

Introduce a `journey` construct that names an actor, declares an outcome,
and lists a DAG of steps — each realised by an existing surface action or
rule — with `captures` / `establishes` / `needs` declarations in typed
field identifiers. The validator enforces that accumulated captures at each
step satisfy that step's `needs` (a subset relation), that the step graph
is acyclic, and that all references resolve.

### Debate

#### Present

The proposal addresses the necessary-but-not-sufficient failure mode: a
spec can be internally consistent at every granularity — each rule, each
surface, each entity — and still fail to guarantee that a stated outcome
is reachable with the information captured along the way. The most trivial
form is a missing process surface between collection and action; the same
failure mode afflicts UI mockups. No existing construct in Allium sees
across rules, surfaces and captures simultaneously, so no existing
construct can catch this gap. The proposed `journey` is the minimum
structure that makes sufficiency a checkable subset relation.

#### Respond

**Simplicity advocate.** I concede the core question: sufficiency needs
validator support, and that support requires the whole path with its
outcome declared. I have two residual concerns. First, the keyword
surface: `journey`, `step`, `achieves`, `via`, `captures`, `establishes`,
`after`, `needs`, `includes`, `all_of`, `diverges`, plus an overloaded
`for` — that is a lot to introduce in one stroke. Second, the hedged
prototype-generation motivation feels like a smuggled requirement; either
the language supports it or it does not. Strip the hedge.

**Machine reasoning advocate.** The denotation is the cleanest shape I
could ask for: subset relations over typed identifier sets, with explicit
join semantics (intersection by default, union via `all_of`). One concrete
worry: `after: a, b` defaulting to intersection is counter-intuitive for
English readers, who read commas as "and". The default is semantically
right — intersection is conservative — but the notation hides the choice.
Consider an explicit `any_of(...)` mirror to `all_of(...)`, with the bare
list reserved for the single-predecessor case only.

**Composability advocate.** `includes` with dotted step references is
the right composition primitive. My remaining concern is cross-module
composition: journeys can `includes` another journey today, but the
proposal defers what happens when that other journey lives behind a
`use` declaration. I will accept deferral if the proposal states
explicitly that cross-module `includes` is not yet supported — so
authors do not discover the limitation by accident.

**Readability advocate.** The domain-facing keywords (`achieves`,
`captures`, `needs`) are excellent. `all_of(...)` is not — it reads as a
programmer's expression, not a domain phrase. Something like
`after: a and b` or `after: both a and b` would carry the same meaning
in language a product owner already uses. The distinction between
`captures` and `establishes` is subtle; I would want the distinction
explained in plain prose in authoring guidance, not just by rule.

**Rigour advocate.** The denotation section is honest and gives me what
I need: captured-set recursion, join semantics, sufficiency as a subset
check, what the check does and does not guarantee. One gap: the
cross-validation of `establishes` against the rule's `ensures` clause is
named but its algorithm is not specified. Does the validator require
every `establishes` field to appear as a field-assignment in `ensures`?
What about fields set via trigger emission or via entity creation? The
check needs to be precise before adoption.

**Domain modelling advocate.** No substantive objection. The multi-actor
support via per-step `actor:` is a real domain win. One request: ensure
the proposal addresses non-human actors (scheduled jobs, external systems
with identities) explicitly, because some domain journeys are kicked off
by a cron trigger rather than a user action. I think this is already
handled via existing actor declarations but it deserves one sentence.

**Developer experience advocate.** The biggest commitment in the proposal
is tend's restructuring mode, and it is sketched in prose without an
error-message catalogue. DX lives and dies on error messages. A
sufficiency failure must name the step, the field, and the predecessors
that did not provide it — all three, in one message, with a concrete
example shape. Add a short error-catalogue section. Separately: how does
a user know whether tend is in restructuring mode versus additive mode?
Surface it to the user explicitly.

**Creative advocate.** The prototype-generation motivation is the most
interesting thing in this proposal and simplicity wants it struck. Keep
it. Hedged motivation is not a smuggled requirement; it is an honest
statement of where the construct points. Every language with a runtime
began with "hedged" motivations that later became concrete. Do not
preemptively trim the ceiling.

**Backward compatibility advocate.** Clean on the installed base: no
existing spec breaks. One compatibility concern the proposal does not
name: the `for` keyword now has three contextual meanings — iteration
inside rules (`for x in collection`), iteration inside invariants (`for
account in Accounts`), and actor binding on a journey (`for Candidate`).
I can read the third in context, but a formally-minded reader or a
machine grammar may find the overload awkward. Flag it, or rename.

#### Rebut

**Readability → Machine reasoning.** On your `any_of(...)` suggestion:
agreed in spirit, but the spelling should be domain-facing. I would
prefer `after: any of a, b` or `after: either a or b` over another
programmer's-expression wrapper. Marrying your concern about the default
to my concern about `all_of` spelling: the solution is the same. Replace
both wrappers with phrasal syntax.

**Machine reasoning → Readability.** Phrasal syntax complicates the
grammar — "any of" and "either…or" introduce multi-word keywords that
the parser must disambiguate. Function-call form is unambiguous.
Compromise: keep the function-call form (`all_of`, `any_of`) but rename
to phrases that read more naturally out of domain conversation, e.g.
`after: both(a, b)` and `after: either(a, b)`. Single-word, domain-facing,
unambiguous. This is a small refinement both of us can accept.

**Rigour → the proposal.** On `establishes` cross-validation: the
cleanest algorithm is "every name in `establishes` must appear as the
target of an assignment in the referenced rule's `ensures`, or as a
field on an entity the rule creates via `.created()`". Trigger emissions
do not establish fields; they emit events. State this in the denotation
section.

**Simplicity → Creative.** I accept the hedged motivation if it is
stated as explicit direction rather than implicit obligation. The
current wording ("is not a requirement the construct must satisfy") is
enough. Concern withdrawn.

**DX → the proposal.** Error-message catalogue: I can draft it in the
refinement. Two modes of sufficiency failure (missing capture, missing
predecessor link) plus acyclicity violation plus identifier-hygiene
failure = four shapes. Ten sentences total. Not a heavy ask.

**Backward compatibility → the proposal.** On `for` overloading: the
cheapest fix is to name the journey actor without `for` — use a colon:
`journey CandidateApplies { actor: Candidate, achieves: ... }`. This
mirrors the per-step `actor:` declaration and removes the overload. The
cost is that `for Candidate` reads a touch more naturally; the benefit
is grammatical regularity and one fewer special case for the validator
and for new readers.

**Composability → the proposal.** Cross-module `includes` deferral is
acceptable if named. Record as a deferred item in the proposal's open
questions.

**Domain modelling → the proposal.** The one-sentence note on non-human
actors: acceptable. Add it.

#### Synthesise

The panel converged substantially during rebuttals. Simplicity withdrew
its motivation-hedge objection. Readability and machine reasoning
converged on a refinement (rename `all_of` / introduce `any_of` as
domain-phrased single-word wrappers like `both(...)` / `either(...)`).
Rigour proposed a specific algorithm for the `establishes`↔`ensures`
check that the proposal can adopt. DX committed to drafting a short
error-message catalogue. Backward compatibility identified the `for`
keyword overload and proposed a concrete fix (rename to `actor:` for
regularity). Composability accepted cross-module deferral if named.
Domain modelling asked for one added sentence.

Three outstanding items have concrete fixes that the proposal does not
yet incorporate. None are large.

### Verdict: Refine

The proposal clears the bar for change (sufficiency is real, recurring,
and unreachable by existing constructs). The shape is principled. The
remaining objections are local and have agreed fixes. Three refinement
items must be applied, after which the refined proposal goes through one
more cycle of the protocol.

#### Refinement items

1. **Parallel / alternative fan-in spelling.** Replace `all_of(...)`
   with a single-word domain-facing wrapper (suggested: `both(...)`).
   Introduce an explicit `either(...)` wrapper so the single-predecessor
   default is not the only unmarked case and authors can be explicit
   when they mean alternative. Bare-list `after: a, b` should either be
   deprecated in favour of explicit `either(a, b)`, or kept as shorthand
   — the panel is open either way but wants the ambiguity removed.

2. **`establishes` cross-validation algorithm, stated in the denotation.**
   A field name appearing in `establishes` must appear as the target of
   an assignment in the referenced rule's `ensures` clause, or as a
   field on an entity created in that rule via `.created()`. Trigger
   emissions do not establish fields. State this precisely.

3. **`for` keyword overloading.** Rename the journey-level actor
   binding from `journey Name for Actor { ... }` to `journey Name {
   actor: Actor, ... }`, mirroring the per-step `actor:` clause and
   eliminating the contextual triple-meaning of `for`. This is a small
   syntactic cost for grammatical regularity.

#### Minor additions agreed during rebuttal

Apply in the same refinement pass:

- **Error-message catalogue.** Short section (~ten sentences) covering
  the four failure modes: missing capture on a sufficiency path, missing
  predecessor link, acyclicity violation, identifier-hygiene failure.
  Each names the step, the field (where applicable), and the offending
  references.
- **Cross-module `includes` deferred, named.** Add to the proposal's
  deferred-items list.
- **Non-human actors.** One sentence confirming that journeys can bind
  to any declared actor type, including integration actors whose
  identity derives from a cron or external-system trigger.

### Key tensions

- **Simplicity vs. Creative** on prototype generation — resolved by
  hedging the motivation.
- **Readability vs. Machine reasoning** on fan-in spelling — resolved
  by phrasing convergence.
- **Rigour vs. the current text** on `establishes` algorithm — resolved
  by specifying the algorithm.
- **Backward compatibility vs. current grammar** on `for` overloading —
  resolved by renaming to `actor:`.
- **Composability vs. current scope** on cross-module composition —
  resolved by explicit deferral.

All tensions are resolvable on a single refinement cycle. None surfaced
an irreducible disagreement.

---

## Deferred items

Carried over from the proposal's own deferred list (no new additions
from the debate, except where noted):

1. **Conditional captures.** Surface actions that capture a field only
   under a specific option. Sufficiency under branching is more subtle
   than under DAG alternatives.
2. **Cross-module journeys.** Journeys that `includes` a journey from
   another module via `use`. *(Added by composability advocate during
   rebuttal; agreed deferral.)*
3. **Validator warnings for speculative structure.** Whether any of the
   authoring rules-of-thumb in `patterns.md` should rise to validator
   warnings, revisited after initial adoption.

---

## Process notes

- The panel read the proposal at `fca0903`, TEAM.md, PROPOSE.md and
  SKILL.md in full. The language reference and patterns file were
  consulted for keyword-conflict and prior-art checks (no conflicts
  found; surface `related:` clauses and the v3 `transitions status {
  ... }` graph are adjacent prior art but do not overlap
  functionally).
- Per the protocol, a second cycle will run after the refinement items
  are applied. Expected cycle-2 verdict: consensus adopt, with one or
  two reservations noted rather than carried.
