# Travel Reimbursement Approval Agent — Design Spec

**Date:** 2026-09-20
**Context:** HCLTech GenAI Engineer case study submission (due 2026-09-22 EOD)
**Deliverable:** a single Jupyter notebook `avadhdobariya.ipynb` in a public GitHub repo

---

## 1. Problem Statement

Build a working prototype agent that evaluates employee travel reimbursement
claims against a fixed policy (Appendix A, rules `POL-*`) and returns a
structured recommendation for each of the five sample claims (Appendix B).

The final code cell must emit a JSON array with one object per claim containing
**exactly** these fields:

```
claim_id, decision, approved_amount, deducted_amount,
missing_docs, policy_refs, confidence, explanation, tools_used
```

`decision` ∈ `{APPROVE, PARTIAL_APPROVE, REJECT, MANUAL_REVIEW}`.

## 2. Constraints That Drive the Design

| Constraint | Source | Consequence |
|---|---|---|
| Single notebook, runs top-to-bottom, no manual steps | Deliverables | No external files to load; policy + claims are embedded as literals. No `input()` calls. |
| Reviewer may have **no API key** | Implied by "runs without manual steps" | The notebook must degrade gracefully to a deterministic path, never crash. |
| Graded on *agentic* AI: tool calling, workflow control, prompting | Evaluation criteria | A pure rule engine scores badly. A real LLM tool-use loop is required when a key exists. |
| Graded on *business correctness* | Evaluation criteria | Money arithmetic must be exact and reproducible — not LLM-generated. |
| "avoidable over-engineering removed" | Evaluation criteria | No vector DB, no LangChain/CrewAI framework, no server. Stdlib + 4 common packages. |
| 2–3 day timebox | Submission guidance | Depth over breadth. MCP is explicitly out of scope. |

## 3. Core Architectural Decision

**The LLM orchestrates; deterministic tools decide the money.**

This is the single most important trade-off in the design. An LLM asked to
compute "2 nights at \$250 against a \$200/night cap" will usually get it right
and occasionally not. Since *business correctness* is a named grading axis,
every dollar figure in the output is produced by a pure Python function with a
unit-testable contract. The LLM's job is genuinely agentic but bounded:

- decide **which** tools to call and in what order,
- decide **when it has enough** information to stop,
- **reconcile conflicting signals** (e.g. a claim that is simultaneously
  over-cap, missing a receipt, and over the director threshold),
- author the human-readable `explanation` and assign `confidence`.

A `MANUAL_REVIEW` trigger from any tool is a hard override the LLM cannot talk
its way out of — enforced by the validator in §7, not by prompting alone.

## 4. Architecture

```
                     ┌─────────────────────────────┐
  claim (JSON) ─────▶│      ReimbursementAgent     │
                     │  (orchestration loop)       │
                     └──────────┬──────────────────┘
                                │  selects tools
                 ┌──────────────┴───────────────┐
                 ▼                              ▼
   ┌──────────────────────────┐   ┌──────────────────────────┐
   │  Planner: LLM            │   │  Planner: Deterministic  │
   │  Claude tool-use loop    │   │  fixed tool sequence     │
   │  (ANTHROPIC_API_KEY set) │   │  (fallback, no key)      │
   └──────────────┬───────────┘   └───────────┬──────────────┘
                  └───────────┬───────────────┘
                              ▼
              ┌────────────────────────────────┐
              │        TOOL REGISTRY           │   ← all pure, deterministic
              │  policy_lookup                 │
              │  receipt_completeness_check    │
              │  per_diem_limit_checker        │
              │  timeliness_check              │
              │  duplicate_detector            │
              │  approval_threshold_check      │
              │  output_validator              │
              └───────────────┬────────────────┘
                              ▼
              ┌────────────────────────────────┐
              │  Validator + self-heal         │
              │  Pydantic schema               │
              │  arithmetic reconciliation     │
              │  MANUAL_REVIEW override        │
              └───────────────┬────────────────┘
                              ▼
                 ClaimDecision  +  AuditTrail
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
        Dashboard (matplotlib)          Final JSON array
        Interactive UI (ipywidgets)     (last code cell)
```

### 4.1 Notebook cell layout

| # | Section | Content |
|---|---|---|
| 1 | Title + README (md) | Setup, env vars, how to run, design choices |
| 2 | Install / imports | `pip install` guarded by try/except; graceful if offline |
| 3 | Policy corpus | Appendix A as structured `POL-*` records + limit table |
| 4 | Claims corpus | Appendix B as 5 claim dicts with line items |
| 5 | Schemas | Pydantic models: `LineItem`, `Claim`, `ClaimDecision`, `ToolCall` |
| 6 | Tool layer | The 7 tools + JSON-schema tool specs for the LLM |
| 7 | Retrieval | Keyword/TF-IDF `policy_lookup` over the POL-* corpus |
| 8 | Planners | `LLMPlanner` (Claude tool-use loop) and `DeterministicPlanner` |
| 9 | Agent | `ReimbursementAgent.evaluate(claim) -> (ClaimDecision, AuditTrail)` |
| 10 | Validator | Schema + arithmetic reconciliation + override + retry |
| 11 | Run all 5 claims | With per-claim audit trail printed |
| 12 | Sample outputs (md+code) | 3 claims walked through in detail |
| 13 | `## Dashboard` | matplotlib charts + ipywidgets interactive claim explorer; saves `UI SS_1.png` |
| 14 | Eval harness | Asserts expected decision + amounts for all 5 claims |
| 15 | Design Notes & Reasoning (md) | Assumptions, trade-offs, Manual Review rationale, next steps |
| 16 | **Final code cell** | `print(json.dumps(results, indent=2))` — the graded artifact |

## 5. Tool Layer (7 tools — brief requires ≥2)

Every tool takes JSON-serialisable input, returns a JSON-serialisable dict, has
no side effects, and cites the `POL-*` ids it applied.

| Tool | Input | Returns | Policy refs |
|---|---|---|---|
| `policy_lookup(query, k=3)` | free-text query | top-k policy rules with id + text + score | — (retrieval) |
| `receipt_completeness_check(claim)` | claim | `missing_docs[]`, `requires_manual_review` | POL-RCT-01, POL-RCT-02 |
| `per_diem_limit_checker(claim)` | claim | per-item `allowed`/`deducted`, totals | POL-PD-01/02/03, POL-CAT-01/02, POL-AIR-01 |
| `timeliness_check(claim)` | claim | `days_late`, `within_window` | POL-TIME-01 |
| `duplicate_detector(claim, history)` | claim + prior claims | `duplicates[]` | — (control) |
| `approval_threshold_check(amount)` | reimbursable total | `tier`, `auto_approvable` | POL-APR-01/02/03 |
| `output_validator(decision_obj)` | candidate decision | `valid`, `errors[]` | schema + arithmetic |

### 5.1 Category eligibility table

Eligible (POL-CAT-01): `airfare`, `lodging`, `meals`, `ground_transport`,
`conference_fees`.
Ineligible (POL-CAT-02): `alcohol`, `minibar`, `spa`, `gym`, `entertainment`,
`in_room_movies`, `shopping`, `gifts`, `fines`, `personal`.

### 5.2 Per-diem computation rules

- **Meals** — cap `$75 × trip_days` (POL-PD-01). Excess deducted.
- **Lodging** — cap `$200 × nights` where `nights = checkout − checkin`
  (POL-PD-02). Excess deducted.
- **Ground transport** — cap `$50 × trip_days` (POL-PD-03). Excess deducted.
- **Airfare** — no cap; economy reimbursed in full. Business/first class is a
  **policy exception → MANUAL_REVIEW, not auto-deducted** (POL-AIR-01).
- **Conference fees** — no cap.
- **Ineligible** — deducted in full (POL-CAT-02).

Caps are applied per **category aggregate across the trip**, not per line item.
This is an explicit assumption documented in the notebook (the policy states
daily/nightly caps; the sample claims present one aggregated line per category,
so aggregating the cap over trip days is the only consistent reading).

## 6. Decision Precedence (deterministic, LLM cannot override)

Evaluated in this order; first match wins:

1. **MANUAL_REVIEW** if *any* of:
   - business/first-class airfare present (POL-AIR-01)
   - a required receipt is missing (POL-RCT-01 → POL-RCT-02)
   - submitted > 30 days after the expense date (POL-TIME-01)
   - reimbursable total > \$2,000 (POL-APR-03)
   - a duplicate claim is detected
   - the validator failed twice
2. **REJECT** if nothing is reimbursable (all items POL-CAT-02).
3. **PARTIAL_APPROVE** if `deducted_amount > 0` and `approved_amount > 0`.
4. **APPROVE** otherwise.

For a `MANUAL_REVIEW`, `approved_amount = 0.0` and `deducted_amount = 0.0` —
no money is committed either way until a human decides. The *provisional*
computation is preserved in the audit trail and surfaced in `explanation`.

## 7. Validator & Self-Heal

1. **Schema validation** — Pydantic; exact field set, enum on `decision`,
   `0.0 ≤ confidence ≤ 1.0`, `policy_refs` matching `^POL-[A-Z]+-\d+$`.
2. **Arithmetic reconciliation** — for non-`MANUAL_REVIEW` decisions,
   `approved_amount + deducted_amount == total_claimed` (±\$0.01).
3. **Precedence override** — recompute §6 from raw tool results; if the LLM's
   `decision` disagrees, the deterministic verdict wins and the disagreement is
   logged to the audit trail.
4. **Retry once** on schema failure with the validator errors fed back to the
   LLM; on a second failure, fall back to the deterministic planner's result
   with `confidence` capped at 0.5.

This layer is what makes the "reliability / fallbacks" criterion demonstrable
rather than claimed.

## 8. Confidence Scoring

Starts at 1.0, reduced by reason codes:

| Reason code | Δ |
|---|---|
| `MISSING_RECEIPT` | −0.30 |
| `POLICY_EXCEPTION` (business class) | −0.25 |
| `OVER_THRESHOLD` (> \$2,000) | −0.15 |
| `LATE_SUBMISSION` | −0.20 |
| `AMBIGUOUS_CATEGORY` | −0.20 |
| `VALIDATOR_RETRY` | −0.25 |
| `LLM_UNAVAILABLE` (deterministic path) | −0.05 |

Floor 0.05. Reason codes are carried in the audit trail and shown in the UI.

## 9. Expected Results (the eval harness asserts these)

| Claim | Decision | Approved | Deducted | Key policy refs |
|---|---|---|---|---|
| CLM-001 | `APPROVE` | 1110.00 | 0.00 | POL-CAT-01, POL-PD-01/02, POL-RCT-01, POL-APR-02 |
| CLM-002 | `REJECT` | 0.00 | 380.00 | POL-CAT-02 |
| CLM-003 | `PARTIAL_APPROVE` | 840.00 | 100.00 | POL-PD-02, POL-APR-02 |
| CLM-004 | `MANUAL_REVIEW` | 0.00 | 0.00 | POL-AIR-01, POL-RCT-02, POL-APR-03 |
| CLM-005 | `MANUAL_REVIEW` | 0.00 | 0.00 | POL-RCT-01, POL-RCT-02, POL-PD-01 |

Derivations:

- **CLM-001** — 2 nights × \$200 = \$400 cap vs \$360 lodging ✓; 3 days × \$75 =
  \$225 cap vs \$180 meals ✓; economy airfare ✓; all receipts ✓; submitted
  8 days after trip end ✓; \$1110 falls in the manager tier (POL-APR-02).
- **CLM-002** — spa and minibar are both POL-CAT-02. Nothing reimbursable →
  REJECT with the full \$380 deducted.
- **CLM-003** — lodging \$500 over a 2-night × \$200 = \$400 cap → \$100
  deducted (POL-PD-02). Meals \$140 under the 2 × \$75 = \$150 cap ✓. Net
  \$840, manager tier → PARTIAL_APPROVE.
- **CLM-004** — three independent MANUAL_REVIEW triggers: business-class
  airfare (POL-AIR-01), lodging receipt missing (POL-RCT-02), and \$3000 over
  the \$2,000 director threshold (POL-APR-03).
- **CLM-005** — **the trap.** A \$220 meal line item exceeds \$25 and has no
  receipt, so POL-RCT-01 → POL-RCT-02 routes to MANUAL_REVIEW. A careless agent
  applies the \$75 single-day meal cap and returns PARTIAL_APPROVE(75, 145);
  the policy is explicit that a missing receipt is "not silently rejected." The
  receipt gate is evaluated **before** per-diem scoring. The provisional
  \$75/\$145 split is recorded in the audit trail for the reviewer.

## 10. Dashboard & UI

Rendered inline under a `## Dashboard` heading and saved to `UI SS_1.png`
next to the notebook:

- **Panel 1** — decision breakdown (bar chart, counts by decision)
- **Panel 2** — claimed vs approved vs deducted per claim (grouped bars)
- **Panel 3** — confidence per claim with the manual-review threshold marked
- **Panel 4** — policy rules fired, ranked by frequency

Plus an **ipywidgets** claim explorer: a dropdown of the 5 claims and a "Run
agent" button that re-runs a single claim and prints its full audit trail —
this is the "means by which the user interacts with notebook code" the brief
asks for. It degrades to a static per-claim printout if `ipywidgets` is
unavailable, so the notebook never breaks.

## 11. Dependencies

`pydantic`, `matplotlib`, `pandas`, `ipywidgets`, `anthropic` (optional).
Install cell is wrapped in try/except: if `pip` fails (offline reviewer), the
notebook falls back to stdlib-only rendering and still produces the final JSON.
No vector database, no agent framework, no network at import time.

## 12. Assumptions

1. Per-diem caps aggregate across the trip (see §5.2) since claims are given as
   one line per category.
2. Lodging nights = `trip_end − trip_start`; meals/ground-transport days =
   `trip_end − trip_start + 1` (inclusive).
3. The 30-day submission window runs from the **trip end date** (the latest
   expense date available in the sample data).
4. `MANUAL_REVIEW` commits no money (both amounts 0.00); the provisional split
   lives in the audit trail.
5. `duplicate_detector` runs against the 5-claim set itself — there is no claim
   history in the provided data, so it demonstrates the capability without
   inventing data (the brief forbids inventing claims).
6. Currency is USD throughout; no FX handling.

## 13. Known Gaps / What I'd Improve Next

- No OCR or real receipt parsing — receipt presence is a boolean in the input.
- Retrieval is lexical (TF-IDF); embeddings would handle paraphrased policy
  queries better but add a dependency and a network call.
- No persistence, no reviewer feedback loop, no learning from overturned
  decisions.
- `duplicate_detector` is intra-batch only; production needs a claims database
  keyed on employee + date + amount + vendor.
- Single-currency, single-policy-version. A real system needs effective-dated
  policy versions so historical claims are judged under the policy in force.
- MCP tool exposure was scoped out to protect notebook runnability inside the
  timebox.

## 14. Out of Scope

Production auth, multi-tenant policy management, an ERP/payroll integration, a
web service, MCP transport, and any persistence layer.
