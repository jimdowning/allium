# Proposal: User journeys

**Status:** pre-proposal sketch, gathering initial reactions
**Revision:** v2 — reframed around information sufficiency after author feedback
**Scope:** new top-level section `journeys`, with a `journey` construct that declares a named outcome for an actor and asserts that the surfaces and rules along a path gather enough information to realise that outcome

---

## Motivation

Working with Allium in anger, the author observed that it is easy to produce
specs which are necessary but not sufficient: the rules are each well-formed,
but nothing in the spec asserts that they collectively support a user's
intended outcome. A surface might collect four of the five fields a downstream
rule needs; a rule might require a score that no earlier step establishes; an
escalation policy might depend on a field that is never captured. The checker
cannot currently see these gaps because no construct ties the relevant
surfaces, captures and rules into a single path with a stated goal.

This is the same weakness that afflicts UI mockups: a wireframe can look
complete while silently omitting fields required by downstream logic. The
problem is general to artefacts that describe parts of a system but not the
flow of information across them toward an outcome.

## The idea in one paragraph

A `journey` names an actor, declares an outcome the actor achieves, and lists
the ordered steps that realise it. Each step references an existing surface
action or rule and declares what it `captures` or `establishes`. The terminal
step declares what it `needs`. The checker walks the path, accumulates the set
of captured/established fields, and verifies that every `needs` entry at each
step is a subset of what has been gathered so far. Referential integrity
(every `via` target exists) and reachability (every declared outcome is named
by some step) are checked as a by-product. Journeys add no runtime semantics:
they do not constrain when rules fire.

## Why this might matter

- **Sufficiency is a genuinely new check.** Nothing in the current language
  asserts that a path through surfaces and rules gathers enough information
  to support its stated outcome. This is exactly the class of bug that
  escapes both rules review (each rule reads correctly in isolation) and UI
  review (each screen looks complete in isolation).
- **The check is concrete and local.** At each step, the checker holds a set
  of established fields and a set of required fields; the question is set
  inclusion. No global reachability analysis, no theorem proving.
- **It makes journeys worth writing.** Pure narrative overlays add a
  documentation obligation without a correspondingly new check. Sufficiency
  turns journeys into a useful artefact that catches real defects.
- **Stakeholder communication still follows.** A sufficiency-bound journey
  still reads as a linear narrative for a product owner, with `achieves`,
  `captures` and `needs` as domain-facing keywords.
- **Distillation target.** A distiller examining a codebase can frame its
  output around journeys: "here are the outcomes this system supports, and
  here are the fields each one depends on".

## Sketch of syntax

```
----
-- Journeys
----

journey CandidateApplies for Candidate {
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

    step interview {
        via: rule InterviewConducted
        establishes: interview_score, interviewer_feedback
        after: apply
    }

    step decide {
        via: rule OfferExtended | rule ApplicationRejected
        needs: email, role_id, resume, interview_score
        after: interview
    }

    diverges:
        candidate_withdraws: rule CandidateWithdraws at any step after signup
        scheduling_timeout: rule InterviewTimeout after apply

    @guidance
        A journey asserts that the captures and establishments along its path
        are sufficient to support its declared outcome. The checker enforces
        this at validation time.
}
```

In this example the checker would pass. If `decide` also needed
`background_check_status`, the check would fail with a clear local error:
*step `decide` needs `background_check_status` but no prior step in
`CandidateApplies` captures or establishes it*.

## Denotation (first pass)

- **Captured-set(step)** = the union of `captures` on surface-action steps
  plus `establishes` on rule steps plus the captured-set of every step that
  this step lists in `after`.
- **Sufficiency condition(step)** = `needs` ⊆ captured-set(step).
- **Journey valid** iff sufficiency holds at every step and every element of
  `achieves` is named by some step's `via`.

Open semantic questions (candidates for the full proposal):

1. **What counts as "established" by a rule?** Candidates: (a) fields
   assigned in `ensures`, (b) only newly-created entities' fields,
   (c) author-declared `establishes` that the checker cross-validates against
   `ensures`. (c) is most explicit and matches how `captures` works on
   surface actions.
2. **How are derived values handled?** If a field is computed from other
   captured fields, does it count as established without a declaration?
   Probably yes, since derivation is deterministic — but the check should be
   able to explain the chain.
3. **Conditional captures.** A surface action might only capture a field
   when a certain option is selected. Sufficiency under branching is
   subtler; likely defer to v2 of the proposal.
4. **Pre-existing context.** Fields on entities in `given` are available
   throughout; they should be in the initial captured-set.

## What the construct adds, and what it deliberately does not

**Adds:**

- Actor-scoped, outcome-bound sequences of references to existing rules and
  surface actions.
- A sufficiency check: accumulated captures at each step must cover that
  step's declared needs.
- Referential-integrity and reachability checks as by-products.
- A reporting artefact: the skill can render a journey as a narrative for
  stakeholder review.

**Does not add:**

- Any constraint on when rules fire; sufficiency is a static property of the
  declared sequence, not a runtime ordering.
- A state machine; entity states remain the source of truth for reachability
  between states of a single entity.
- Implementation details (screens, routes, clicks).

## Design questions still open

1. **Linear vs DAG.** Should `after` permit multiple predecessors? DAG is
   strictly more expressive; sufficiency composes fine across a DAG
   (captured-set is still a union), but readability suffers.
2. **Single-actor vs multi-actor.** Many domain journeys include handoffs
   (candidate → recruiter → hiring manager), and each party contributes
   different captures. Do we permit `for Candidate, Recruiter` or force
   separate journeys with explicit interlocks?
3. **Composition.** Can a journey reuse another as a prefix? Under
   sufficiency this is natural: the child journey's captured-set becomes the
   initial captured-set of the parent. But it raises module-boundary
   questions.
4. **Relationship to invariants and surfaces.** If sufficiency is
   checkable, should surfaces and rules be able to declare their own `needs`
   independently, with journeys being one way to group path checks? (The
   creative advocate flagged this in v1 and it remains open.)

## Alternatives considered

- **Do nothing.** The sufficiency gap is handled ad-hoc by authors
  re-reading rules after each change. The gap compounds as specs grow.
- **Invariants rather than journeys.** Instead of a new construct, add a
  `sufficient_for` modifier to invariants. This preserves one less keyword
  but loses actor binding and narrative ordering, both of which the domain
  and readability advocates rate highly.
- **Annotations on rules.** Tag rules with `part_of: CandidateApplies`.
  Scatters the journey definition across the spec; incompatible with
  sufficiency (the checker can't assemble a path from tags without a
  declared order).
- **Patterns entry + rendering skill, no language change.** Cheapest option;
  matches the simplicity advocate's instinct from v1. But it cannot deliver
  the sufficiency check — the whole point of the revised proposal — because
  only the parser/validator has access to all required information for the
  subset check.

---

# Panel reactions (v2, on the sufficiency framing)

Short version of the TEAM.md protocol: each panellist gives a two-to-four
sentence initial reaction, no rebuttals. Goal is to see whether the
sufficiency reframing changes their position.

### Simplicity advocate

Materially shifted. My v1 objection — that this was a view over existing
constructs deliverable as tooling — is answered: the sufficiency check
cannot be delivered without the validator seeing the whole path with its
outcome declared. I still want the minimum viable form to be defended
against an invariant-with-`sufficient_for` approach, because that might get
the same check with one fewer top-level concept. But the proposal now has a
genuine reason to exist.

### Machine reasoning advocate

Positive. A subset relation over accumulated field sets is exactly the
shape of check I find tractable: no ambiguity, no surprisal, a clear local
error when it fails. My one structural concern is that `captures` and
`establishes` must resolve to the same kind of entity — typed fields or
something equivalent — otherwise the subset check becomes informal.
Require both sides to reference declared field identifiers, not free-form
names.

### Composability advocate

Sufficiency makes composition more important, not less, and the sketch
defers it. If a child journey's captured-set can feed a parent's, that is
the first composition story worth writing down; I want it in the proposal
before adoption. Also: the `after: step_name` reference is local to the
journey, which is fine, but once journeys compose, step names become a
namespace concern. Resolve this now or pay later.

### Readability advocate

Still enthusiastic. The reframing adds `captures`, `establishes`, `needs`
and `achieves`, all of which read naturally to a product audience
("what does this step capture? what does the decision need?"). If anything
the sufficiency frame is easier to explain to a non-technical stakeholder
than the v1 narrative frame, because it answers a question they already
ask ("do we have enough to make this decision?").

### Rigour advocate

Major shift. The denotation section gives me something to argue with, which
v1 did not. The subset-relation formulation is clean. The open semantic
questions (what a rule establishes, derived values, conditional captures)
are real, but they are tractable and honest. I would also ask for a
statement of what sufficiency does *not* guarantee — it asserts
information presence, not information correctness — to prevent
false-confidence use.

### Domain modelling advocate

Stronger support than v1. "Does this process gather enough to make the
decision?" is a canonical domain question, and journeys are the right
locus for it. I reiterate the multi-actor concern: many real journeys have
handoffs, and each party contributes different captures. A strictly
single-actor construct will either be under-used or worked around with
scaffolding. At minimum, allow a journey to name the other actors it
depends on without making them first-class.

### Developer experience advocate

Positive, conditional on error quality. Sufficiency failures need to point
at the field, the step that needs it, and the set of prior steps that did
not provide it — all three. A message like "step `decide` needs
`interview_score`; checked against captures from steps: signup, apply
(none establish `interview_score`)" is the bar. Anything less and the
check becomes frustrating rather than useful. I also want to understand
the blast radius when a rule adds a new `requires` field: how many
journeys break, and can the checker group those failures?

### Creative advocate

Encouraged. The reframing is exactly the kind of move I wanted in v1: a
construct that turns a narrative into a checkable property. Now the
question is whether journeys are the only place sufficiency should live.
Consider letting rules and surface-action composites declare their own
`needs`, with journeys being one way to group path checks. That would make
sufficiency a cross-cutting property of the language rather than a
journeys-only feature, which is a bigger idea but potentially a simpler
language.

### Backward compatibility advocate

Neutral on the core: still a new section, still doesn't change existing
specs. One real compatibility risk surfaces under sufficiency: if journeys
later enforce `establishes` declarations on rules, existing rules do not
have them, and every rule referenced from a journey would need annotation.
Either keep `establishes` optional with inference from `ensures`, or ship
a mechanical migration alongside. Do not impose a hand-review burden on
the installed base.

---

# Synthesis (v2)

The reframing answers the hardest v1 objection (simplicity: "why is this a
language construct?") and gives the proposal a denotation the rigour
advocate can argue with rather than dismiss. Every panellist's position has
shifted toward support; the split now is not *whether* to pursue this but
*how far to take it*.

The work to turn this into a real proposal has narrowed to four items:

1. **Defend the construct against `invariant … sufficient_for`.** Simplicity
   accepts that sufficiency needs validator support, but wants to see why a
   new top-level `journey` beats a modifier on existing invariants. The
   answer probably lies in actor binding and narrative ordering, both of
   which invariants lack — but the argument must be written.

2. **Nail the data-flow semantics.** What does a rule `establish`? The
   cleanest answer is author-declared `establishes` cross-validated against
   `ensures`, with inference for straightforward cases. Derived values fall
   out of this. Conditional captures are a v2 concern and can be flagged as
   open.

3. **Decide composition now, not later.** Sufficiency under composition is
   clean (captured-sets union), so there is no semantic obstacle; the
   question is purely how to spell it. Pick a spelling before going to the
   full protocol, so composability has nothing to rebut against.

4. **Error message specification.** Developer experience's bar for error
   quality is specific enough to write down as part of the proposal, not
   left to implementation. A one-page "error catalogue" in the proposal
   would settle this.

Two further questions — multi-actor journeys (domain modelling) and
sufficiency-as-cross-cutting-property (creative) — are real but can be
marked as deferred and revisited after v1 of the construct is live. They do
not block adoption of the narrower form.

**Recommended next step:** write the four items above into a formal proposal
body and submit to the full PROPOSE.md protocol. The sufficiency reframing
has cleared the pre-proposal bar.
