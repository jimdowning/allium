# Proposal: User journeys

**Status:** pre-proposal sketch, gathering initial reactions
**Scope:** new top-level section `journeys`, with a `journey` construct that binds an actor to an ordered narrative composed of references to existing rules and surface actions

---

## The idea in one paragraph

Today an Allium spec captures rules, surfaces and actors, but not the narrative
that ties them together from the perspective of a single actor over time. A
product owner asking "walk me through what a candidate experiences" has to
reconstruct the sequence by reading rules in isolation and inferring ordering
from state transitions. A `journey` construct would let a spec author name that
narrative explicitly, bind each step to an existing rule or surface action, and
declare the terminal outcomes (success, withdrawal, timeout). The construct
adds no new behaviour; it is an overlay that references constructs already in
the spec, validated by the checker for referential integrity and reachability.

## Why this might matter

- **Stakeholder communication.** The README already describes Allium as a
  communication artefact for non-technical stakeholders. Rules read well in
  isolation but poorly as a sequence. A journey collapses that gap.
- **Gap surfacing.** If a journey declares six steps and only five have rules
  to realise them, that is a missing specification the checker can flag.
  Today the gap is invisible because nothing in the spec asserts that the
  sequence is meant to be supported.
- **Distinguishing happy path from edge handling.** Rules are symmetric:
  `LoginSucceeds` and `LoginFails` are both first-class. Journeys let an author
  mark one as the intended path and others as branches, without giving rules
  themselves a priority.
- **Distillation target.** A codebase tour often follows a user's path through
  the system. Journeys give distillation a named destination, not just a flat
  list of rules.

## Sketch of syntax

```
----
-- Journeys
----

journey CandidateApplies for Candidate {
    starts_with: CandidateSignsUp

    step ProfileCompleted {
        realised_by: rule ProfileSubmitted
    }

    step ApplicationReviewed {
        realised_by: rule ApplicationTriaged
        after: ProfileCompleted
    }

    step InterviewScheduled {
        realised_by: surface CandidatePortal.book_slot
        after: ApplicationReviewed
        within: config.scheduling_window
    }

    step DecisionIssued {
        realised_by: rule OfferExtended | rule ApplicationRejected
        after: InterviewScheduled
    }

    ends_with: OfferExtended | ApplicationRejected

    diverges:
        candidate_withdraws: rule CandidateWithdraws at any step
        scheduling_timeout: rule InterviewTimeout after ApplicationReviewed

    @guidance
        A journey names the accepted path an actor takes through the system.
        It does not introduce new behaviour; every step must resolve to an
        existing rule or surface action. Branches that are part of the
        domain's accepted outcomes belong in `diverges`, not as silent gaps.
}
```

## What the construct adds, and what it deliberately does not

**Adds:**

- A named, actor-scoped sequence of references to existing rules/surfaces
- A checker obligation: every `realised_by` reference resolves; every
  `after` reference names a step within the same journey; `ends_with` values
  are reachable
- A reporting artefact: the skill can render a journey as a linear narrative
  for stakeholder review

**Does not add:**

- Any new runtime, ordering or causal semantics. A journey does not constrain
  when rules fire; rules continue to fire based on their own `when`/`requires`
  clauses.
- A state machine. Entity state transitions remain the source of truth for
  what is reachable.
- Implementation concerns like UI screens, clicks or page routes.

## Design questions still open

1. **Linear vs DAG.** Should `after` support multiple predecessors (a DAG) or
   only one (a line)? DAG is more expressive but harder to read aloud.
2. **Should a step name its own rule, or discover it?** The sketch uses
   `realised_by: rule X`, requiring the author to name the rule. An alternative
   is inference from a declared post-condition. Naming is safer; inference is
   more compact.
3. **Divergence as a first-class clause, or just more steps with
   `ends_with`?** The sketch treats divergence specially. It could collapse to
   ordinary steps plus multiple `ends_with` outcomes.
4. **Can journeys be nested or reused?** A "candidate reapplies" journey might
   include the original application journey as a prefix. Composition adds
   power and cost.
5. **Relationship to surfaces.** Surfaces already describe boundaries between
   parties. A journey often crosses surfaces. Does a journey compose surfaces,
   or sit orthogonal to them?

## Alternatives considered

- **Do nothing.** Authors capture journeys in prose documentation outside the
  spec. This is the current state and the default disposition for any
  proposal. The cost is that the narrative drifts from the rules.
- **Annotations on rules.** Each rule could declare `part_of: CandidateApplies`
  tags. This scatters the journey definition across many rules and makes the
  narrative impossible to read as a unit.
- **A patterns entry, not a language construct.** Document the convention of
  writing a comment block that lists rules in order. Cheapest option. Gives up
  checker validation and tooling support.

---

# Initial panel reactions

A short version of the protocol in `TEAM.md`: each panellist gives a two-to-four
sentence initial reaction, no rebuttals. The goal is to see whether the idea has
legs, not to reach a verdict.

### Simplicity advocate

Cautious. The sketch is mostly a view over existing constructs, which means a
lot of the value could be delivered by a linter or a rendering skill rather
than a new section of grammar. I want to see the minimum viable version
before judging: what is the smallest thing that would be unworkable as pure
tooling? If the answer is "checker reachability", fine, but make the case
explicit.

### Machine reasoning advocate

Interested. A journey declared with explicit `realised_by` references is
exactly the kind of structured cross-reference that automated tooling handles
well. My concern is ordering semantics: if `after` does not constrain rule
firing, I need the reference to be unambiguous about what it does mean.
"Narrative ordering for presentation, not execution" is a defensible answer
but must be stated precisely.

### Composability advocate

Mixed. Composition is the obvious next question: journeys within journeys,
journeys across modules, journeys that share steps. The sketch defers all of
this. I would rather see the composition story resolved before adoption than
ship a construct that has to grow composition later under pressure, since
retrofitting composition is where most languages acquire their warts.

### Readability advocate

Enthusiastic. This is the first construct in the language that speaks
directly to my audience. A product owner reading "journey CandidateApplies
for Candidate" immediately knows what they are looking at, and the step
names read as a table of contents. Keep the keywords domain-facing
(`starts_with`, `ends_with`, `diverges`) and this is a net gain for
non-technical readers.

### Rigour advocate

Sceptical of the semantics. "Narrative ordering" sounds like a hedge. If a
journey step says "ApplicationReviewed after ProfileCompleted" but the rules
fire in the opposite order, what does the spec mean? Either the ordering is
enforced (in which case it duplicates entity state machines) or it is
decorative (in which case it can mislead). I want the semantics stated as a
denotation before I can evaluate the cost.

### Domain modelling advocate

Supportive. User journeys are a well-established unit of conversation in
product and service design, and the language currently forces modellers to
fragment that conversation across rules. Binding journeys to an actor matches
how domain experts actually talk. I would push harder on the actor binding:
can a journey involve more than one actor, or does it strictly follow one?

### Developer experience advocate

Positive on the "day one" test. A newcomer reading `journey X for Y` knows
what it is without a tutorial, and the `realised_by` references give them a
way to jump straight to the relevant rule. My worry is error messages. If a
journey step references a rule that was later renamed, the failure mode needs
to be a clear, local error, not a cascade.

### Creative advocate

Wants more ambition. The sketch stays safe by declaring journeys inert. That
is a defensible first step, but the real expressive gain would be letting
journeys participate in invariants: "every Candidate who reaches
ApplicationReviewed either reaches DecisionIssued or appears in a documented
divergence within config.window". That is the construct that turns journeys
from narration into a checkable property.

### Backward compatibility advocate

Neutral. A new top-level section with a new keyword does not change any
existing valid specification. Migration cost is zero for adopters who ignore
it. My one request: if journeys later acquire semantics (as the creative
advocate suggests), those semantics must be opt-in, because the decorative
version will already be in use by then and retrofitting behaviour onto it
will be painful.

---

# Synthesis of initial reactions

The idea has legs. No panellist recommends dropping it; the split is between
enthusiasm (readability, domain modelling, machine reasoning, creative) and
conditional support (simplicity, composability, rigour, DX, backward
compatibility).

The work to turn this into a real proposal is concentrated in three areas:

1. **Denotation.** State precisely what a journey means. The rigour and
   simplicity advocates both need this; machine reasoning agrees. Candidates:
   (a) pure narrative overlay with referential-integrity checks only,
   (b) reachability assertion (every declared terminal is reachable given the
   rules), (c) enforceable ordering (equivalent to a state machine over
   entities). The first is cheapest; the third is most powerful but likely
   duplicates entity state transitions.

2. **Minimum viable form.** Answer the simplicity advocate's question: what
   is the smallest construct that delivers the value? Specifically, is this a
   language construct at all, or a patterns entry plus a rendering skill? The
   bar to clear is PROPOSE.md's default disposition: leave the language
   alone.

3. **Composition.** Resolve the composability advocate's concern before
   adoption, not after. Specifically: can journeys share steps, nest, or
   cross modules? Decide now, because retrofitting composition is how
   languages acquire warts.

Secondary questions that can wait for the formal proposal: linear vs DAG
ordering, single- vs multi-actor journeys, relationship to surfaces, and
whether journeys can participate in invariants (the creative advocate's
expansion).

**Recommended next step:** pick a denotation (my instinct is (a) + a narrow
form of (b) — referential integrity plus terminal reachability — but this
should be argued, not asserted), write up the minimum viable form, and
answer the composition question. At that point the sketch is ready for the
full PROPOSE.md protocol.
