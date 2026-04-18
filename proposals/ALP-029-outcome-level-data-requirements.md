# ALP-29: Locally consistent specifications may fail to support stated outcomes

**Status**: proposed
**Related**: ALP-022 (cross-rule data dependencies, adopted)
**Constructs affected**: `rule` (`requires`, `ensures`), `surface` (`exposes`, `provides`), `entity` (fields), validation rules
**Sections affected**: Rules, Surfaces, Entities and Variants, Validation rules

## Problem

Allium validates each rule, surface, entity and invariant independently. A
rule's `requires` and `ensures` clauses are checked against the entity's
field declarations. A surface's `exposes` and `provides` clauses are checked
against the declared actor and context. Each construct is internally
consistent when its references resolve. The validator cannot detect when a
collection of internally-consistent constructs fails to collectively
support a stated purpose.

Consider a purchase spec. A `CheckoutForm` surface provides an action that
takes `customer_email` and `payment_method`. A `DispatchOrder` rule has
`ensures: order.status = dispatched` and reads `order.shipping_address`.
The spec passes validation. No surface, rule or entity is malformed. And
yet no surface in the spec captures a shipping address, so no order can
reach `DispatchOrder`. A reader examining each construct in isolation does
not see this; a reader tracing the whole spec must assemble the gap by
hand.

The same failure mode appears in subtler forms:

- A rule requires a computed field (`interview_score`) that no preceding
  rule produces.
- A decision rule's `requires: user.verified_email = true` is never
  satisfiable because no surface or rule sets `verified_email` to `true`.
- A multi-step process has a missing intermediate surface between data
  collection and data use, so information captured at an early step has no
  path to the rule that needs it downstream.

In each case the spec passes validation, each individual construct reads
correctly, and the gap is visible only by reading every rule and surface
together and mentally tracing which data reaches which decision.

ALP-022 (*Cross-rule data dependencies*, adopted) addresses a narrower
version of this problem: tracing implicit dependencies between rules
within an entity lifecycle, where data written by one rule is read by
another at a later lifecycle state. This ALP concerns the broader pattern
where information flows across rules, surfaces and actor handoffs toward a
stated purpose, and where the stated purpose itself has no first-class
representation in the language.

## Evidence

A spec under development by the author for a hiring process passed the
validator with a `DecisionIssued` rule that required
`background_check_status` on a `Candidacy`, while no surface in the spec
captured or derived this field. The missing capture was discovered only
during a manual walkthrough of the intended end-to-end process. The rule
was correct; the surface was correct; the capture was absent, and the
validator had no basis to flag it. Similar gaps had been introduced and
later corrected by manual review in other specs the author maintains.

The wireframe analogy is the recurring external example of this failure
mode. Design teams routinely produce wireframes where every screen is
individually complete but the set of screens fails to carry the data the
backend needs at decision points. This happens despite the wireframes
being reviewed and approved, because reviewing each screen in isolation
does not test whether the screens compose to support the stated purpose.
Allium currently shares this weakness.

Practitioners of domain-driven design describe this as the difference
between *consistent* and *sound* models. Consistency can be checked
locally; soundness requires a view of the whole. Event-storming and
user-story-mapping workshops exist in part because no formal artefact
exists that expresses soundness across events, surfaces and rules.

ALP-022's adoption acknowledges that the language needed a way to reason
about data flow between rules. That fix is narrower than the problem here:
it assumes the flow is an intra-entity lifecycle, expressible through
transition graphs plus produces/consumes declarations. The author's
experience and the wireframe analogy both point at a broader pattern —
cross-construct, outcome-bound — where the same validator gap persists.

## Design tensions

**Locality vs. whole-view.** Allium's existing checks are local: a rule's
validity is checkable from the rule's own body plus the entity it
references. Detecting the insufficiency pattern requires a view that
crosses construct boundaries. Whatever addresses this must resolve how far
the check should reach without forcing local validation to carry global
state.

**Actor-driven vs. infrastructural specs.** Some specs describe flows an
actor experiences (user-facing processes, approval workflows); others
describe infrastructural behaviour (circuit breakers, batch processors,
scheduled jobs). The insufficiency pattern is pronounced in the first
category and largely absent in the second. A fix scoped only to
actor-driven specs avoids imposing a discipline where no benefit accrues;
a fix that treats all specs uniformly imposes cognitive cost on authors of
infrastructural specs without a corresponding gain.

**Overlap with ALP-022.** ALP-022 already allows rules to declare
produces/consumes within entity lifecycles. Any mechanism here must
subsume that declaration, coexist with it without duplication, or carve a
clean boundary between the two. The worst outcome is two adjacent
vocabularies for the same relationship.

**Absence of a first-class purpose.** Allium has no construct for "the
purpose this spec supports". Rules terminate in state changes and entity
creations; surfaces expose data and provide actions; invariants assert
properties that must hold. None of these name the stated outcome that the
rules and surfaces are meant collectively to realise. A check for
insufficiency needs a target: what is this spec meant to achieve? The
language currently has no place to say.

**Vocabulary register.** The language is deliberately domain-facing:
`exposes`, `provides`, `requires`, `ensures`. Any vocabulary introduced to
address this problem must land in the same register. Programmer's
vocabulary (`depends_on`, `inputs`, `outputs`) would drift from the stated
audience of product owners and domain experts.

**Many valid shapes for the same flow.** Unlike entities (a field belongs
to one entity) and rules (a behaviour is a rule), information flows across
a system admit multiple valid representations of the same domain process.
The same sequence can be expressed as a linear chain, as a graph with
alternatives, as a decomposition into nested processes, or as a pair of
handoffs between actors. A mechanism must either impose a canonical shape
(losing expressiveness) or accept shape-malleability (creating cognitive
load and tooling complexity).

## Scope

This ALP concerns the detection of specifications that are individually
consistent at the construct level while failing to collectively gather or
relay the information required to realise their stated purpose. It covers:

- the gap between data captured at surfaces and data required by rules
  that consume it,
- the analogous gap between data produced by earlier rules and data
  required by later rules across entity boundaries, where ALP-022 does
  not reach,
- the case where a stated purpose is not reachable at all through any
  sequence of captures and rules in the spec.

It does not cover:

- the specific shape of any mechanism that addresses the problem,
- runtime semantics or execution ordering of rules,
- validation of the correctness of captured values (e.g., whether an
  email is well-formed),
- user-interface concerns (screens, routes, clicks, navigation),
- the extent to which any first-class representation of stated purpose
  should be reusable, composable or nested — these are considerations
  for a mechanism, not constraints on the problem.
