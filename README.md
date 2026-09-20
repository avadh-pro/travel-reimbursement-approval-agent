# Travel Reimbursement Approval Agent

**HCLTech — GenAI Engineer case study**

**Avadh Dobariya** &middot; avadhdobariya@gmail.com &middot; [github.com/avadh-pro](https://github.com/avadh-pro)

An AI agent that evaluates employee travel reimbursement claims against policy,
receipts, per-diem limits and approval thresholds, and returns a structured
recommendation: `APPROVE`, `PARTIAL_APPROVE`, `REJECT`, or `MANUAL_REVIEW`.

### The deliverable

**[`avadhdobariya.ipynb`](avadhdobariya.ipynb)** — a single, self-contained notebook.
It runs top-to-bottom with no manual steps and no API key. The full README, setup
steps, sample outputs and the "Design Notes & Reasoning" section all live inside it,
as the assignment specifies.

```bash
pip install jupyter
jupyter notebook avadhdobariya.ipynb     # Run All
```

Setting `ANTHROPIC_API_KEY` switches the agent from its deterministic fallback
planner to a live Claude tool-use loop. Both paths produce the same decisions.

### Results at a glance

| Claim | Decision | Approved | Deducted | Driven by |
|---|---|---:|---:|---|
| CLM-001 | `APPROVE` | $1,110.00 | $0.00 | fully compliant, manager tier (POL-APR-02) |
| CLM-002 | `REJECT` | $0.00 | $380.00 | spa + minibar are ineligible (POL-CAT-02) |
| CLM-003 | `PARTIAL_APPROVE` | $840.00 | $100.00 | lodging over the $200/night cap (POL-PD-02) |
| CLM-004 | `MANUAL_REVIEW` | $0.00 | $0.00 | business class + missing receipt + over $2,000 |
| CLM-005 | `MANUAL_REVIEW` | $0.00 | $0.00 | $220 meal, no receipt (POL-RCT-01 → POL-RCT-02) |

![Dashboard](UI%20SS_1.png)

### Repository contents

| Path | What it is |
|---|---|
| `avadhdobariya.ipynb` | **The deliverable.** Agent, tools, dashboard, UI, tests, design notes. |
| `UI SS_1.png` | Results dashboard, written by the notebook when the dashboard cell runs. |
| `UI SS_2.png` | The claim-reviewer interface. |
| `docs/specs/` | The design spec written before implementation. |
