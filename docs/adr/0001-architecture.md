# ADR-0001: NonwovenAdvisor ⊣ Nonwoven & Technical Textiles Plant Operations Governor architecture

## Status

Accepted. `cloud-itonami-isic-1399` promoted from `:spec` to
`:implemented` in the `kotoba-lang/industry` registry, following the
verified fresh-scaffold protocol established by prior actors in this
fleet.

## Context

`cloud-itonami-isic-1399` publishes an OSS blueprint for ISIC 1399
(Manufacture of other textiles n.e.c.) **operations coordination**. As
a residual "n.e.c." category, 1399 covers nonwovens, felt, tire-cord
fabric, and other technical/industrial textiles not captured by
siblings 1391 (knitted/crocheted fabrics), 1392 (made-up textile
articles), 1393 (carpets and rugs), or 1394 (cordage/rope/twine/
netting). Like every actor in this fleet, the blueprint alone is not
an implementation: this ADR records the governed-actor architecture
that promotes it to real, tested code, following the same langgraph
StateGraph + independent Governor + Phase 0->3 rollout pattern
established across the cloud-itonami fleet.

This build's own chosen concrete illustration is nonwoven/felt fabric
manufacturing: needle-punched, spunbond, or meltblown forming lines
bonded (thermal, chemical, or mechanical) into nonwoven fabric or
felt, including technical/industrial textiles such as tire-cord fabric
and geotextiles. Production-batch output quality is tracked via a
tensile-strength test (kilonewtons) and a basis-weight test (grams per
square meter, gsm) -- the two standard nonwoven/felt QC measurements
-- alongside a defect-rate reading and a closed quality-grade set.

The closest domain analog is `cloud-itonami-isic-1394` (Manufacture of
cordage, rope, twine and netting): both are back-office plant-
operations-coordination actors with real physical-safety-relevant
equipment (forming-line/bonding-line equipment vs. fibre-twisting/
braiding/winding-line equipment), production-batch tracking with
quality/grade data, and equipment maintenance scheduling with a
permanent block on directly operating that equipment. This build
mirrors 1394's module shape (advisor ⊣ governor ⊣ phase ⊣ store, four
ops, `MemStore`-only backend) closely, substituting nonwoven-specific
ground truth: `area-square-meters` in place of `length-meters`,
`tensile-strength-kn` (from a tensile-strength test) in place of
`breaking-strength-kn`, and ADDS a second, independent quality-control
field this domain's own task spec explicitly calls for --
`basis-weight-gsm` (from a basis-weight test) -- that 1394 has no
analog to. A permanent `:direct-operate?` block mirrors 1394's own
`:direct-operate?` block -- a proposal-level field that, if set true,
attempts to bypass "propose/schedule a DRAFT" and reach actual
equipment operation. `:flag-safety-concern` (equipment-safety/
quality-defect) mirrors 1394's own `:flag-safety-concern` -- an
always-escalating concern-flagging surface specified for this
vertical.

This vertical has NO pre-existing `kotoba-lang/nonwoven`-style
capability library to wrap (verified: no such repo exists). This
build therefore uses self-contained domain logic -- pure functions in
`nonwovenops.registry` (equipment/batch verification, shipment-area
recompute, quality-grade validation, tensile-strength-test
plausibility validation, basis-weight-test plausibility validation,
defect-rate plausibility validation) are re-verified independently by
the governor, the same "ground truth, not self-report" discipline
established across prior actors (most directly
`cloud-itonami-isic-1394`'s `cordageops.registry`).

This blueprint's own `:itonami.blueprint/governor` keyword,
`:nonwoven-technical-textiles-plant-operations-governor`, is
grep-verified UNIQUE fleet-wide (`gh search code
"nonwoven-technical-textiles-plant-operations-governor" --owner
cloud-itonami`, zero hits before this repo was created).

Regulatory context (informational, not enforced in code): nonwoven and
technical-textile manufacturing is subject to product-safety and
labor-standards regimes in multiple jurisdictions -- e.g.
international nonwoven-fabric test/classification standards such as
ISO 9073 (textiles -- test methods for nonwovens) and ISO 10319
(geotextiles -- wide-width tensile test), and Japan's 労働基準法 (Labor
Standards Act) and 労働安全衛生法 (Industrial Safety and Health Act)
covering plant labor conditions. `:flag-safety-concern`'s
`:equipment-safety`/`:quality-defect` concern-types exist to surface
exactly this class of issue to a human for review -- this actor does
not itself adjudicate compliance.

## Decision

### Decision 1: Self-contained domain logic (no external nonwoven capability library to wrap)

Unlike actors that delegate to pre-existing domain libraries, this
nonwoven/felt technical-textiles vertical has NO pre-existing
capability library to wrap. The equipment/batch-verification /
shipment-area / quality-grade / tensile-strength / basis-weight /
defect-rate validation functions live as pure functions in
`nonwovenops.registry` and are re-verified independently by
`nonwovenops.governor` -- the same "ground truth, not self-report"
discipline established across prior actors (most directly
`cloud-itonami-isic-1394`'s `cordageops.registry`).

### Decision 2: Coordination, not control — scope boundary at the back-office

This actor is **strictly back-office coordination** of nonwoven/felt-
plant operations. It does NOT:
- Control forming-line or bonding-line equipment directly
- Make plant-safety, labor-safety, or materials-safety decisions (exclusive to the human plant supervisor)
- Directly operate forming/bonding-line equipment under any proposal

All proposals are `:effect :propose` only. The advisor proposes; the
governor validates; escalation paths funnel to human plant-supervisor
approval. This is not a replacement for the supervisor's authority —
it is a proposal-screening and documentation layer.

**CRITICAL SAFETY BOUNDARY**: nonwoven/felt manufacturing carries real
physical-safety and labor-standards dimensions (entanglement and
pinch-point injury risk on forming lines, thermal/chemical-bonding
hazards on bonding lines, repetitive-strain labor conditions,
materials-defect and equipment-safety risk). Safety-concern flagging
NEVER auto-commits. All safety concerns escalate immediately to human
review.

### Decision 3: Safety-concern escalation — always human sign-off

`:flag-safety-concern` (equipment-safety or quality-defect concern)
ALWAYS escalates, never auto-commits. This is not a "low-stakes
proposal" — it is a circuit-breaker that must reach human authority,
deliberately broad enough to cover both a defective-fabric materials
finding and a machine-condition equipment finding simultaneously.

### Decision 4: Two independent verified/registered gates (equipment AND batch), not one

Unlike a single-entity-gated vertical, this vertical has TWO entity
kinds each gating a different op: `:schedule-maintenance`
independently verifies the referenced **equipment** unit's own
`:verified?`/`:registered?` fields; `:coordinate-shipment`
independently verifies the referenced **batch**'s own
`:verified?`/`:registered?` fields. Both are the same "plant/batch
record must be independently verified/registered before any action"
HARD invariant applied to the two distinct record kinds this domain
actually has. `:coordinate-shipment` additionally independently
recomputes whether a batch's own recorded shipped-to-date area plus
the proposal's own claimed area would exceed the batch's own recorded
production area -- never taken on the advisor's self-report.

### Decision 5: Two independent quality-control fields (tensile-strength AND basis-weight), not one

Unlike `cloud-itonami-isic-1394`'s single breaking-strength-test
plausibility check, this build's own task spec explicitly calls out
BOTH a tensile-strength test and a basis-weight test as the batch
output-quality data this actor logs. Both get their own independent
plausibility-validation function (`tensile-strength-valid?` /
`basis-weight-valid?`) and their own dedicated governor HARD check --
one additional concrete check beyond 1394's own elaboration, for this
domain's own explicit basis-weight-test data-logging requirement.

### Decision 6: HARD invariants (no override)

Four HARD governor invariants (elaborated into twelve concrete checks
in `nonwovenops.governor`, mirroring `cloud-itonami-isic-1394`'s own
elaboration of its HARD invariants into concrete checks -- one
additional check beyond 1394's own eleven, for this domain's own
explicit basis-weight-test data-logging requirement) block proposals
and cannot be overridden by human approval:
1. Plant/batch record (equipment for maintenance, batch for shipment) must be independently verified/registered before any action is taken against it, and a shipment's area must independently recompute within the batch's own logged production area
2. Proposals must be `:effect :propose` only (never direct equipment control)
3. Direct forming/bonding-line-equipment control (`:direct-operate? true`) is permanently blocked
4. The op allowlist is closed — `:log-production-batch`/`:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` only

## Consequences

(+) Nonwoven/felt technical-textiles plant operations back-office now
has a documented, governed, auditable coordination layer that funnels
all decisions through independent validation before human approval.

(+) The "coordination, not control" boundary is explicit in code: all
`:effect :propose`, all real-world actuation requires human plant-
supervisor sign-off.

(+) Scope is bounded and verifiable: four HARD invariants (elaborated
into twelve concrete governor checks) protect against scope creep into
unauthorized equipment operation. Safety concerns are a
circuit-breaker, not a threshold.

(+) Safety-critical discipline is explicit: safety-concern flagging
cannot be rate-limited, suppressed, or auto-decided by phase gate.
Human review is mandatory.

(-) Still a simulation/proposal layer, not a real plant-operations
control system. Equipment actuation remains human-controlled via
external channels.

(-) No integration with real plant-management databases (equipment
telemetry, batch tracking, freight dispatch) — this is a standalone
coordinator blueprint.

## Verification

- `cloud-itonami-isic-1399`: `clojure -M:test` green (all tests pass;
  see the superproject ADR and `kotoba-lang/industry` registry entry
  for the exact `Ran N tests containing M assertions, 0 failures, 0
  errors` output, verified from an independent fresh clone), `clojure
  -M:lint` clean, `clojure -M:dev:run` demo narrative exercises
  proposal submission, escalation, and every HARD-hold scenario
  directly (not-propose-effect, unknown-op, equipment-not-verified,
  batch-not-verified, shipment-area-exceeded, direct-operate-blocked,
  already-scheduled, invalid-grade, invalid-tensile-strength,
  invalid-basis-weight, invalid-defect-rate).
- All source is `.cljc` (portable ClojureScript / JVM / nbb) — no
  JVM-only interop; the actor graph is invoked exclusively via
  `langgraph.graph/run*` (not `.invoke`, which is not cljs-portable).
- Audit ledger is append-only, all decisions are traced; every settled
  request (commit or hold) leaves exactly one ledger fact.
- `deps.edn` pins `io.github.kotoba-lang/langgraph` and
  `io.github.kotoba-lang/langchain` via `:local/root` directly in the
  top-level `:deps` (not only under a `:dev` alias), so a bare
  `clojure -M:test` resolves offline inside the monorepo checkout.
