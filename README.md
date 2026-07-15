# cloud-itonami-isic-1399: Manufacture of other textiles n.e.c.

Open Business Blueprint for **ISIC Rev.5 1399**: manufacture of other textiles n.e.c. — a residual textile category (nonwovens, felt, tire-cord fabric, and other technical/industrial textiles) distinct from siblings 1392 (made-up textile articles), 1393 (carpets and rugs), and 1394 (cordage/rope/twine/netting). This is an autonomous "actor" (LLM advisor behind an independent Governor, langgraph-clj StateGraph, append-only audit ledger) that coordinates back-office plant **operations**.

## Concrete illustration chosen for this build

ISIC 1399 is a residual "n.e.c." class, so this build picks a single concrete illustrative product line to make the domain logic real rather than abstract: **nonwoven/felt fabric manufacturing** -- needle-punched, spunbond, or meltblown forming lines, bonded (thermal, chemical, or mechanical) into nonwoven fabric or felt, including technical/industrial textiles such as tire-cord fabric and geotextiles. Production-batch quality is tracked via a tensile-strength test (in kilonewtons) and a basis-weight test (in grams per square meter, gsm) -- the two standard nonwoven/felt quality-control measurements, alongside a defect-rate reading and a closed quality-grade set.

## What this actor does

Proposes **plant operations coordination**, not machine operation:
- `:log-production-batch` — forming/bonding batch, output-quality (tensile-strength/basis-weight test) data logging (administrative, not an operational decision)
- `:schedule-maintenance` — forming-line/bonding-line-equipment maintenance scheduling proposal
- `:flag-safety-concern` — surface an equipment-safety/quality-defect concern (always escalates)
- `:coordinate-shipment` — outbound product shipment coordination proposal

## What this actor does NOT do

**CRITICAL SCOPE BOUNDARY** (forming lines, bonding lines; materials-handling and equipment-safety hazards):

- Does NOT control forming-line or bonding-line equipment directly
- Does NOT make plant-safety, labor-safety, or materials-safety decisions (that's the plant supervisor's exclusive human authority)
- Does NOT directly operate forming/bonding-line equipment under any proposal (permanently blocked, see Architecture)
- ONLY proposes/coordinates operations back-office; all actuation requires explicit human approval
- Safety-concern flagging ALWAYS escalates — never auto-decided, no confidence threshold or phase below escalation

## Architecture

Classic governed-actor pattern (`nonwovenops.operation/build`, a langgraph-clj StateGraph):
1. **`nonwovenops.advisor`** (sealed intelligence node, `NonwovenAdvisor`): proposes decisions only, never commits
2. **`nonwovenops.governor`** (independent, `Nonwoven & Technical Textiles Plant Operations Governor`): validates against domain rules, re-derived from `nonwovenops.registry`'s pure functions and `nonwovenops.store`'s SSoT -- never trusts the advisor's own self-report
   - HARD invariants (always `:hold`, no override):
     - Plant/batch record must be independently verified/registered (`:verified?` AND `:registered?`) before any action is taken against it (equipment before maintenance scheduling, batch before shipment coordination)
     - The request's own `:effect` must be `:propose` (never a direct-write bypass)
     - `:op` must be in the closed four-op allowlist
     - The proposal's own `:effect` must be one of the four propose-shaped effects (no direct forming/bonding-line-equipment control)
     - Directly operating forming/bonding-line equipment (`:direct-operate? true`) is a PERMANENT, unconditional block
     - A shipment may not push a batch's own recorded shipped area past its own logged production area (independently recomputed)
     - No double-scheduling the same maintenance record
     - No fabricated `:quality-grade` value on a production-batch patch
     - No physically implausible `:tensile-strength-kn` value on a production-batch patch
     - No physically implausible `:basis-weight-gsm` value on a production-batch patch
     - No physically implausible `:defect-rate-percent` value on a production-batch patch
   - ESCALATE (always human sign-off, overridable by a human):
     - `:flag-safety-concern` always escalates, regardless of confidence
     - Low-confidence proposals
3. **`nonwovenops.phase`** (Phase 0->3 rollout): `:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` are NEVER in any phase's `:auto` set (permanent, matching the governor's own posture); only `:log-production-batch` may auto-commit at phase 3 when clean
4. **`nonwovenops.store`** (append-only audit ledger + SSoT): a single `MemStore` backend behind a `Store` protocol (see ns docstring for why a second Datomic-backed backend is out of scope for this build)

## Development

```bash
# Run tests (top-level deps.edn already pins langgraph+langchain local/root)
clojure -M:test

# Run tests via the workspace :dev override alias (equivalent, kept for sibling-repo parity)
clojure -M:dev:test

# Run the demo
clojure -M:dev:run

# Lint
clojure -M:lint
```

## Status

`:implemented` — `governor.cljc`/`store.cljc`/`advisor.cljc`/`registry.cljc` + `deps.edn` complete the module set; tests green, demo runnable, langgraph-clj integration verified.

## License

AGPL-3.0-or-later
