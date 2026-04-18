# Proposal: User journeys

**Status:** v4 — refinements from cycle 1 of the PROPOSE.md protocol applied, entering cycle 2
**Scope:** new top-level section `journeys`, with a `journey` construct that declares a named outcome for an actor and asserts that the surfaces and rules along a path gather enough information to realise that outcome

---

## Motivation

Working with Allium in anger surfaces a failure mode the current language
cannot catch: specifications that are necessary but not sufficient. Each rule
is well-formed in isolation, each surface contract is internally consistent,
each entity carries the fields its own operations need — and yet the spec as
a whole does not guarantee that a user's intended outcome can be achieved.
A surface collects four of the five fields a downstream rule requires; a rule
needs a score no earlier step establishes; an escalation policy depends on a
field that is never captured at a user-facing boundary. The checker cannot see
these gaps because no construct ties the relevant surfaces, captures and rules
into a single path with a stated goal.

The most trivial form of this failure is a missed surface. An author writes a
surface for collecting information and another for acting on it, each
internally consistent, but forgets the intermediate surfaces that bridge them
— a review step, a triage handoff, a reviewer-facing queue. The collection
surface is well-formed, the action surface is well-formed, and yet no path
between them exists. Per-surface review does not catch this because each
surface is fine in isolation; the gap is relational. A journey with a declared
outcome names the relation, and the checker flags the missing link as an
unsatisfiable step.

This is the failure mode that afflicts UI mockups: a wireframe can look
complete while silently omitting fields required by downstream logic. The
problem is general to artefacts that describe the parts of a system but not
the flow of information across those parts toward an outcome.

A secondary motivation, held as a hedge rather than a normative claim:
journeys compose surfaces in a way that makes generation of a runnable
skeleton tractable. A spec whose journeys are complete — declared surfaces,
declared captures, declared rules, declared outcomes — describes enough of
the IO structure of an application that a clickable prototype can be produced
from it, with business-logic rule bodies remaining to be implemented. This is
not a requirement the construct must satisfy. It is a direction that
justifies the construct being shape-aware about surfaces rather than purely
narrative.

## The idea in one paragraph

A `journey` names an actor, declares an outcome the actor achieves, and lists
the ordered steps that realise it. Each step references an existing surface
action or rule and declares what it `captures` or `establishes` in typed
field identifiers. Steps that consume information declare what they `need`.
The checker walks the DAG of steps, accumulates the set of captured and
established fields at each point, and verifies that every `needs` entry is a
subset of what has been gathered by the step's predecessors. Referential
integrity (every reference resolves) and acyclicity (no cycles in the DAG)
are checked as by-products. Journeys add no runtime semantics: they do not
constrain when rules fire. They are static assertions about the information
structure of a path to an outcome.

## Syntax

```
------------------------------------------------------------
-- Journeys
------------------------------------------------------------

journey CandidateApplies {
    actor: Candidate
    achieves: OfferExtended | ApplicationRejected

    step signup {
        via: PublicPortal.register
        captures: email, name
    }

    step apply {
        via: CandidatePortal.submit_application
        captures: role_id, resume
        after: signup
    }

    step technical_screen {
        actor: TechnicalReviewer
        via: rule TechnicalScreenConducted
        establishes: technical_score
        after: apply
    }

    step interview {
        actor: HiringManager
        via: rule InterviewConducted
        establishes: interview_score, interviewer_feedback
        after: technical_screen
    }

    step decide {
        actor: HiringManager
        via: rule OfferExtended | rule ApplicationRejected
        needs: email, role_id, resume, technical_score, interview_score
        after: interview
    }

    diverges:
        candidate_withdraws: rule CandidateWithdraws at any step after signup
        scheduling_timeout: rule InterviewTimeout after apply

    @guidance
        A journey asserts that the captures and establishments along its path
        are sufficient for its declared outcome. The validator enforces this
        at spec-check time; it does not constrain rule execution.
}
```

### DAG joins

A step with a single predecessor uses a bare reference:

```
step interview { after: apply, ... }
```

A step with multiple predecessors must declare whether the predecessors are
combined disjunctively (the step is reached via any one of them) or
conjunctively (the step is reached after all of them). The two forms are
n-ary functions `or(...)` and `and(...)`:

```
step decide { after: or(phone_screen, in_person_screen) }       -- any-of
step decide { after: and(interview, background_check) }         -- all-of
```

Under `or(...)` the captured-set at the step is the intersection of its
operands' captured-sets — only fields established on every incoming path
are relied upon downstream. This is the conservative default for
alternative paths.

Under `and(...)` the captured-set is the union, because every operand is
asserted to have occurred and every operand's establishments are
available.

The two functions compose freely and nest to arbitrary depth:

```
step decide {
    after: and(apply, or(phone_screen, in_person_screen))
}
```

This reads as: `apply` must have completed AND one of `phone_screen` or
`in_person_screen` must have completed. The resulting captured-set is
`captured-set(apply) ∪ (captured-set(phone_screen) ∩
captured-set(in_person_screen))`. The validator computes this by recursive
descent over the `after:` expression.

The names `and` and `or` are reserved as journey-context keywords in
`after:` clauses. They do not clash with the boolean operators of the
same name used in expression contexts, because the parser disambiguates
by position: `after:` takes a step-reference expression, not a boolean.

### Nesting and composition

Journeys compose by inclusion. Steps in a nested journey are referenced from
the parent via dotted paths:

```
journey SeniorHire {
    actor: Candidate
    achieves: OfferExtended | ApplicationRejected

    includes: CandidateApplies

    step reference_check {
        actor: Recruiter
        via: rule ReferenceCheckCompleted
        establishes: reference_verdict
        after: CandidateApplies.interview
    }

    step senior_decide {
        actor: HiringCommittee
        via: rule OfferExtended | rule ApplicationRejected
        needs: email, role_id, technical_score, interview_score, reference_verdict
        after: reference_check
    }
}
```

The nested journey's captured-set flows into the parent at every step that
references a nested-journey step as a predecessor. When a step is extracted
into a sub-journey, its original name remains available as a qualified path
(`CandidateApplies.interview`) so existing references survive restructuring.

### Multi-actor journeys

The journey-level `actor:` names the primary actor. Per-step `actor:`
overrides narrow specific steps to a different declared actor. The
validator verifies that the named actor is authorised to invoke the step's
surface or is permitted to participate in the step's rule.

The actor may be any declared actor type, including integration actors
whose identity derives from a cron trigger or external system rather
than a user.

## Denotation

### Captured-set recursion

For a step S with `after:` expression E, the captured-set is:

```
captured-set(S) = eval(E) ∪ captures(S) ∪ establishes(S) ∪ given
```

where `eval(E)` returns the captured-set implied by the expression, defined
by recursive descent:

- `eval(P)` = `captured-set(P)` for a bare step reference P
- `eval(or(E₁, …, Eₙ))` = ⋂ᵢ `eval(Eᵢ)`
- `eval(and(E₁, …, Eₙ))` = ⋃ᵢ `eval(Eᵢ)`

Entry steps (no `after:` clause) have `captured-set(S) = captures(S) ∪
establishes(S) ∪ given`. Fields on entities declared in the module's
`given` block are available in the initial captured-set of every step.

### `establishes` cross-validation

A name N in a step's `establishes` clause must satisfy one of the
following, where R is the rule referenced by the step's `via`:

- N appears as the left-hand side of a field assignment in R's `ensures`
  clause (e.g., `entity.N = value`), or
- N appears as a field on an entity that R creates via `Entity.created(...)`
  within its `ensures` clause.

Trigger emissions in `ensures` do not establish fields. Names that appear
in neither form produce a validation error.

For steps whose `via` references a surface action, `captures` must
correspond to declared input parameters of that action — a strictly
narrower check, equivalent to exact match of declared parameter names.

### Validity

A journey is valid when:

- sufficiency holds at every step: `needs(S) ⊆ captured-set(S)`,
- every element of `achieves` is named by some step's `via`,
- the step graph is acyclic,
- every referenced step, rule, surface and actor resolves to a declared
  entity in scope, and
- every `establishes` entry satisfies the cross-validation above.

### What the check does not guarantee

Sufficiency is an information-presence check, not an information-correctness
check. It asserts that the fields required by a step have been captured or
established somewhere earlier in the journey; it does not assert that those
fields carry correct, valid, or current values. Correctness remains the
responsibility of rule `requires` clauses and invariants.

## On representation

Entities and rules admit essentially one valid shape per domain fact: if a
field belongs to a particular entity, that choice is forced. Journeys do not
have this property. The same domain process can legitimately be represented
as:

- a linear sequence of steps,
- a DAG with alternative or parallel paths,
- a parent journey that includes a nested sub-journey,
- two sibling journeys that share nothing syntactic,

and the choice depends on judgement the language cannot make (is this process
reused? does it have a distinct actor? is it always required?). The proposal
takes the position that this shape-malleability is intentional and not a
defect. The language validates well-formedness — referential integrity,
sufficiency, acyclicity, identifier hygiene — but does not enforce a
canonical shape.

Authoring guidance on when to nest, when to branch, when to extract, and how
to name steps coherently lives in `references/patterns.md`, not in the
language. Skills that restructure journeys (principally `tend`) consult the
patterns file.

## Identifier hygiene

Every name appearing in `captures`, `establishes`, or `needs` must resolve to
a declared field on an entity or value-type in the spec. Undeclared names are
a validation error. This is a small authoring discipline for a substantial
soundness gain: the sufficiency check becomes a subset test over typed
identifiers rather than string matching over author-chosen labels. Two
different entities may carry fields with the same name (each lives in its
own namespace), so step references that could be ambiguous must qualify:
`Candidate.email` rather than bare `email`.

## Error messages

The validator produces four classes of error specific to journeys. Each
names the journey, the step and the offending reference(s) so the error
is locally actionable.

### Sufficiency failure

When a step's `needs` set is not a subset of its accumulated captured-set:

```
error: step `decide` in journey `CandidateApplies` needs
       `interview_score` but no prior step captures or establishes it.
       Checked predecessors:
         - signup (captures: email, name)
         - apply (captures: role_id, resume)
```

### Missing predecessor link

When an `after:` expression references a step that is not declared in the
journey or any included journey:

```
error: step `decide` in journey `CandidateApplies` lists
       `technical_screen` as a predecessor, but no such step is declared.
       Did you mean `apply`?
```

### Acyclicity violation

When the step graph contains a cycle:

```
error: cycle detected in journey `CandidateApplies`:
       decide -> review -> decide. A journey's step graph must be acyclic.
```

### Identifier-hygiene failure

When a name in `captures`, `establishes` or `needs` does not resolve to a
declared field on an entity or value type:

```
error: step `decide` in journey `CandidateApplies` needs
       `verified_email`, but no entity or value type in scope declares
       this field. Similar declared fields: `email` on Candidate.
```

## What the construct adds, and what it deliberately does not

### Adds

- Actor-scoped, outcome-bound DAGs of references to existing surface actions
  and rules.
- A sufficiency check: accumulated captures at each step cover that step's
  declared needs.
- Referential integrity, identifier hygiene and acyclicity checks as
  by-products.
- A composition mechanism (`includes`) with dotted step references.
- A reporting artefact: tools can render a journey as a linear narrative for
  stakeholder review.

### Does not add

- Any constraint on when rules fire. Sufficiency is a static property of the
  declared sequence, not a runtime ordering.
- A state machine. Entity states remain the source of truth for reachability
  between states of a single entity.
- Implementation details (screens, routes, clicks).
- A canonical shape for the same domain process.

## Skill integration

### elicit

Journey-first elicitation inverts the current middle phase. Rather than
tracing state machines per entity, the skill walks outcomes per actor and
derives entities, rules and surfaces as a by-product.

The skill infers journey-frame suitability from the user's opening prompt
where possible. A prompt describing actor-driven outcomes ("users apply for
jobs", "candidates book interview slots") routes to the journey-first flow.
A prompt describing infrastructural behaviour ("the service breaks circuits
on failure", "a scheduled job archives stale records") routes to the
current entity-first flow. The skill asks only when the prompt does not
disambiguate.

Journey-first elicitation preserves the existing six-phase structure of
elicit. Phase 2 becomes a journey walkthrough producing the journey block
directly. Phase 4 gains a sufficiency sweep that verifies each step's
needs against the accumulated captures, surfacing gaps as open questions.
The onion-peeling phase structure is retained: sufficiency questions are
asked in Phase 4, not folded into Phase 2.

### tend

Tend gains a restructuring mode, entered when a change touches an existing
journey. Simple additions (new rule, new field, new invariant) remain in
the current additive mode. Changes that insert steps, reorder a path, or
affect a step referenced by more than one journey trigger structural
consideration.

When a change has more than one plausible structural shape, tend presents
the options with their domain tradeoffs and asks the user to choose,
rather than picking silently. Signals in the user's phrasing ("also",
"alternative", "shared with") let tend propose a specific option first
with visible reasoning; genuinely ambiguous phrasing triggers a small-N
options dialogue.

Authoring guidance for structural choices lives in `patterns.md`; tend
reads the patterns file rather than duplicating the guidance in its own
skill description. When tend extracts a step into a sub-journey, the
original step's name is preserved as a qualified path so external
references remain stable.

Tend is obliged to re-run sufficiency after any restructuring, update all
cross-references coherently, and confirm explicitly before any change that
affects more than one journey or that removes or renames an existing step.

### distill

Distill suggests journey candidates from codebase analysis. Tracing from
HTTP or CLI entry-points through to terminal writes produces candidate
paths; each is presented as a suggested journey for the author to accept,
reject, or reshape. Distill does not commit journey structure silently.

### weed

Weed gains a concrete question to ask of implementation: does the codebase
expose a surface action for every `via: Surface.action` reference in each
journey, and does each surface action collect the fields declared as
`captures`? This is the check that catches the UI-mockup failure mode
against real code.

### propagate

Propagate uses journeys as anchors for change scope. When a shared rule is
modified, the set of journeys that reference it identifies which specs
must be revisited.

## Alternatives considered

- **Do nothing.** The sufficiency gap is handled ad-hoc by re-reading rules
  after each change. The gap compounds as specs grow and is the failure
  mode the proposal exists to address.
- **`invariant … sufficient_for` modifier.** An invariant carrying the path,
  captures and needs would save a top-level construct but loses step
  structure (errors become global pass/fail), loses composition, loses
  nesting, and loses actor binding beyond a single field. Readability and
  locality-of-errors are both much worse.
- **Annotations on rules** (`part_of: CandidateApplies`). Scatters the
  journey across the spec; cannot support sufficiency because the checker
  cannot assemble a path from tags without a declared order.
- **Patterns entry plus rendering skill, no language change.** Delivers
  referential-integrity checks via skill-level tooling but cannot deliver
  the sufficiency check, because the checker does not see the whole path
  and its outcome together.

## Backward compatibility

The proposal adds a new top-level section and new keywords. It does not
change the meaning of any existing valid specification. Specs that do not
use journeys are unaffected and require no migration.

The one surface where backward compatibility matters is `establishes`. A
journey references existing rules, and those rules do not carry an
`establishes` annotation of their own today. The proposal does not require
this: `establishes` is declared on the journey step, cross-validated
against the rule's `ensures` clause. No rule-side change is needed. Rules
that are not referenced from any journey are not affected.

## Open questions (deferred)

These are not blockers for the initial construct and are flagged for a
subsequent iteration:

1. **Conditional captures.** A surface action may capture a field only
   when a certain option is selected. Sufficiency under conditional
   capture is subtler than under DAG alternatives and deserves dedicated
   treatment.
2. **Cross-module journeys.** A journey that spans modules via `use`
   declarations must resolve step references across module boundaries.
   The mechanism exists for other constructs and can be extended; the
   interaction with nested journeys needs care. In particular,
   `includes:` referencing a journey declared in another module is not
   supported in the initial construct; within-module composition only.
3. **Validator warnings for speculative structure.** The patterns file
   will encode rules-of-thumb for when to nest vs. when to branch vs. when
   to extract. Whether any of these rules-of-thumb rise to the level of
   validator warnings is a judgement call worth revisiting after initial
   adoption shows which shapes authors actually produce.

## Status and next step

Cycle 1 of the PROPOSE.md protocol returned a verdict of **Refine** with
three concrete refinement items and three minor additions. v4 applies all
six:

- `all_of(...)` replaced with n-ary `and(...)` / `or(...)` wrappers that
  nest to arbitrary depth; bare multi-predecessor lists deprecated in
  favour of explicit composition.
- `establishes` ↔ `ensures` cross-validation algorithm stated in the
  denotation.
- Journey-level `for Actor` renamed to `actor: Actor`, eliminating the
  contextual overloading of `for`.
- Short error-message catalogue added.
- Cross-module `includes` named as deferred.
- One-sentence note on non-human actors added.

**Next step:** cycle 2 of the PROPOSE.md protocol against v4.
