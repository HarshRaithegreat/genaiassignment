# Travel Reimbursement Approval Agent

## Overview

A single-notebook (`yourname.ipynb` — rename to `<yourname>.ipynb`) GenAI/agentic prototype that reviews travel-reimbursement claims against the Appendix A policy and returns `APPROVE`, `PARTIAL_APPROVE`, `REJECT` or `MANUAL_REVIEW`.

**Principle:** *LLM = interpretation, orchestration, explanation. Python = deterministic financial / business-rule enforcement.*

## Problem Statement

Evaluate the five Appendix B claims against policy (categories, per-diem caps, receipts, approval tiers, 30-day window), use real LLM tool calling, route anything uncertain to Manual Review, validate the output, keep an audit trail, and show a data-driven dashboard.

## Solution Architecture

```
INPUT CLAIM (JSON) -> CLAIM NORMALIZATION (Pydantic Claim / ClaimItem)
        |
        v
OPENAI GPT-4o-mini AGENT (OpenAI SDK -> OpenAI endpoint, function calling, max 8 iterations)
   |-- policy_lookup_tool          retrieve POL-* rules (id / category / concept / keyword)
   |-- category_eligibility_tool   ELIGIBLE / INELIGIBLE / MANUAL_REVIEW
   |-- receipt_requirement_tool    POL-RCT-01
   |-- reimbursement_limit_tool    per-diem caps, Decimal math
   |-- approval_threshold_tool     POL-APR-01/02/03
   |-- submission_window_tool      POL-TIME-01
   `-- conflict_detection_tool     inconsistencies / ambiguity
        |
        v
TOOL RESULTS / EVIDENCE
        |
        v
DETERMINISTIC DECISION ENGINE   (final authority: decision, amounts, confidence)
        |
        v
OPENAI EXPLANATION SYNTHESIS    (writes the final JSON)
        |
        v
PYDANTIC VALIDATION -> CONSISTENCY VALIDATION -> 1 repair call
        | ok                              | still invalid / failure
        v                                 v
   FINAL RESULT                    MANUAL_REVIEW fallback
        |
        v
AUDIT TRAIL -> JUPYTER DASHBOARD (ipywidgets)
```

## Technologies

Python 3.10+, `openai` SDK, GPT-4o-mini, `pydantic` v2, `ipywidgets`, `pandas`, `matplotlib`. No LangChain/LlamaIndex, vector DB, Docker, server or cloud resources.

## Prerequisites

* Python 3.10 or newer, Jupyter (Notebook, Lab or VS Code)
* An OpenAI API key with access to `gpt-4o-mini`
* Internet access to `https://api.openai.com` (only needed when `RUN_LLM_DEMO = True`)

## Installation

```bash
pip install openai pydantic ipywidgets pandas matplotlib jupyter
```

## Environment Configuration

The notebook reads exactly:

```python
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY", "")
```

> **Why is the key still called `OPENAI_API_KEY`?** Because the **OpenAI Python SDK** conventionally expects that environment variable name, and the model is accessed through the standard OpenAI endpoint using `base_url="https://api.openai.com/v1"` and `model="gpt-4o-mini"`.

```bash
# Linux / macOS
export OPENAI_API_KEY="your-openai-key"
```
```powershell
# Windows PowerShell
$env:OPENAI_API_KEY="your-openai-key"
```

Set the variable **before** starting Jupyter. The key is never hard-coded or printed. With `RUN_LLM_DEMO = True` the notebook raises a clear error if the key is missing (it never pretends an LLM call happened).

## Running the Notebook

1. Open `yourname.ipynb`, *Run All* (top to bottom, no manual steps).
2. `RUN_LLM_DEMO = True` (default, used for the final demo): GPT-4o-mini performs real tool calling for each claim.
3. `RUN_LLM_DEMO = False`: deterministic engine + all tests only, no API/network needed.

The final cell prints only the JSON array for the five claims.

## Agent Workflow

1. The normalized claim and the system prompt go to GPT-4o-mini together with the 7 tool schemas.
2. The model decides which tools to call; Python executes them (the claim is bound server-side, so the model never re-types amounts) and returns results as `tool` messages. Calls can be parallel and repeated.
3. If the model skips tools it is nudged once. The loop is capped at `MAX_TOOL_ITERATIONS = 8`; exceeding it routes to `MANUAL_REVIEW`.
4. The deterministic engine computes the authoritative result from the tool evidence.
5. The model receives the engine result and writes the final JSON (explanation included).
6. Pydantic validation + consistency validation. On failure: one repair call; if still invalid, `MANUAL_REVIEW`.

## Tools

| Tool | Purpose |
|------|---------|
| `policy_lookup_tool` | Retrieve policy rules by id, category, concept or keyword (contextual grounding, no vector DB) |
| `category_eligibility_tool` | ELIGIBLE / INELIGIBLE (POL-CAT-02) / MANUAL_REVIEW (unknown category, business/first-class or unverifiable airfare) |
| `receipt_requirement_tool` | Which items need receipts (> $25; airfare and lodging always) and which are missing (POL-RCT-01/02) |
| `reimbursement_limit_tool` | Applicable limit, claimed, allowed, deduction, policy id per item (Decimal) |
| `approval_threshold_tool` | AUTO_APPROVE (≤ $500) / MANAGER (≤ $2,000) / DIRECTOR_MANUAL_REVIEW (> $2,000) on the post-deduction total |
| `submission_window_tool` | Expense date, submission date, elapsed days, compliant/late (POL-TIME-01) |
| `conflict_detection_tool` | Total mismatch, receipt metadata contradictions, invalid dates, category/description conflicts, empty or non-positive items |

## Policy Grounding

Appendix A is an in-memory dict `POLICY_RULES` with all 12 rules (`POL-CAT-01/02`, `POL-PD-01/02/03`, `POL-AIR-01`, `POL-RCT-01/02`, `POL-APR-01/02/03`, `POL-TIME-01`). Each keeps id, title, description, applicability, categories, concepts, behaviour and numeric `parameters`. Limits and thresholds are read from these parameters (single source of truth). The agent retrieves only the rules it needs instead of receiving the whole policy.

## Deterministic Decision Engine

`evaluate_claim_deterministically(claim, tool_results)` is the final authority. All money uses `Decimal` (HALF_UP to cents).

1. Any manual-review reason → `MANUAL_REVIEW`. **All** reasons are collected (e.g. CLM-004 keeps business class + missing receipt + > $2,000).
2. Else nothing reimbursable → `REJECT`.
3. Else any deduction while something remains reimbursable → `PARTIAL_APPROVE`.
4. Else `APPROVE`.

Invariant for automated decisions: `approved + deducted == claimed`.

## Manual Review

Triggers: missing required receipt, business/first-class (or unverifiable) airfare, unknown category, late submission, reimbursable total > $2,000, conflicting or missing information, and any LLM/tool/validation failure.
For manual-review results `approved_amount = 0.00` and `deducted_amount = 0.00` (the agent neither approves nor deducts what a human has yet to decide; the claimed amount is stated in the explanation).

**Confidence** is an application-level score, *not* a calibrated probability: 0.95–1.00 deterministic and unambiguous (APPROVE 0.98, REJECT 0.97, PARTIAL 0.96) · 0.80–0.94 strong evidence with minor uncertainty (manual review with a clear deterministic reason: 0.90) · 0.60–0.79 meaningful ambiguity · < 0.60 manual review strongly preferred (ambiguity/conflict 0.55, agent-failure fallback 0.30).

## Output Schema

Exactly nine fields, `extra="forbid"`:

`claim_id`, `decision` (`APPROVE|PARTIAL_APPROVE|REJECT|MANUAL_REVIEW`), `approved_amount`, `deducted_amount` (Decimal, 2 dp, ≥ 0), `missing_docs`, `policy_refs` (must exist in the repository), `confidence` (0–1), `explanation`, `tools_used` (tools actually executed). The JSON is rendered with money as numbers (e.g. `1110.00`). The audit trail is kept separately.

## Dashboard

The "Dashboard" section shows (all from actual results): claim count, count per decision, total claimed / approved / deducted / pending-manual-review, a decision-breakdown chart and an outcome-split chart per claim. The ipywidgets panel lets you select a sample claim, view/edit its JSON (or paste your own), press **Run agent** and see decision, amounts, missing docs, policy refs, confidence, explanation, tools used and optionally the audit trail. It runs the OpenAI model when `RUN_LLM_DEMO=True`, otherwise the deterministic engine.

## Testing

* **A. Deterministic policy tests (49)** – no LLM/network. The five claims, meal $75 / $75.01, lodging $200 / $200.01, ground $50 / $50.01, totals $500 / $500.01 / $2,000 / $2,000.01, receipt thresholds ($25.00 vs $25.01), missing airfare/lodging receipt, business/first/unknown class, 30 vs 31 days, ineligible, mixed, partial, conflicts (total mismatch, receipt amount mismatch, description conflict, date inversion, empty claim), combined business class + missing receipt + > $2,000, claim-id independence, Decimal exactness, tool outputs, schema strictness, JSON number formatting.
* **B. LLM-layer integration tests (13)** – a scripted fake OpenAI client proves the tool-calling loop, nudging, unknown tool / malformed arguments, iteration cap, retry with backoff, retry exhaustion, auth failure, invalid response, validation → repair → fallback, fabricated policy ids / wrong `tools_used`, and that a manual-review claim cannot be auto-approved.
* The real OpenAI run is the §15 evaluation with `RUN_LLM_DEMO=True`.

Verified in development: with `RUN_LLM_DEMO=False` the notebook ran top-to-bottom with **49/49 and 13/13 tests passing**. The live OpenAI call itself could not be exercised in the build environment, so run it once with your key before the interview and check the audit trail.

## Sample Results

Deterministic engine output (with the OpenAI model enabled, the structured fields are identical because they are validated against the engine; only the wording of `explanation` is written by the model):

| Claim | Decision | Approved | Deducted | Confidence | Policy refs |
|---|---|---|---|---|---|
| CLM-001 | APPROVE | 1110.00 | 0.00 | 0.98 | POL-CAT-01, POL-PD-01, POL-PD-02, POL-AIR-01, POL-RCT-01, POL-APR-02, POL-TIME-01 |
| CLM-002 | REJECT | 0.00 | 380.00 | 0.97 | POL-CAT-02, POL-TIME-01 |
| CLM-003 | PARTIAL_APPROVE | 840.00 | 100.00 | 0.96 | POL-CAT-01, POL-PD-01, POL-PD-02, POL-AIR-01, POL-RCT-01, POL-APR-02, POL-TIME-01 |
| CLM-004 | MANUAL_REVIEW | 0.00 | 0.00 | 0.90 | POL-CAT-01, POL-PD-02, POL-AIR-01, POL-RCT-01, POL-RCT-02, POL-APR-03, POL-TIME-01 |
| CLM-005 | MANUAL_REVIEW | 0.00 | 0.00 | 0.90 | POL-CAT-01, POL-PD-01, POL-RCT-01, POL-RCT-02, POL-APR-01, POL-TIME-01 |

* **CLM-001** all eligible, receipts present, meals $60/day ≤ $75, lodging $180/night ≤ $200, total $1,110 (manager tier) → approve.
* **CLM-002** spa + minibar are POL-CAT-02 → reject, $380 deducted.
* **CLM-003** lodging $250/night × 2 = $500 vs cap $400 → deduct $100; meals $70/day OK; total $840 → partial.
* **CLM-004** business-class airfare (POL-AIR-01) + missing lodging receipt (POL-RCT-02) + $3,000 > $2,000 (POL-APR-03) → manual review, all three reasons preserved.
* **CLM-005** $220 dinner without receipt (> $25) → manual review (POL-RCT-02).

## Design Decisions

* **GPT-4o-mini via the OpenAI SDK**: model choice used for the live agent; standard OpenAI API keeps dependencies minimal.
* **Structured policy, no vector DB**: 12 short rules with exact numbers are better served by keyed lookup – deterministic, explainable, zero infrastructure.
* **Python owns the money**: LLMs make boundary/arithmetic mistakes; `Decimal` + policy-sourced parameters remove that risk.
* **LLM still orchestrates and explains**: it chooses the checks, reads the evidence and writes the justification, constrained by validation.
* **Manual Review over forcing a decision**: matches Appendix A; multiple reasons are preserved.
* **Hallucination control**: bound claim data, tool-only numbers, prompt-injection-aware prompt, field-by-field engine check, unknown policy ids rejected.
* **Fail safe**: backoff for transient errors; everything unrecoverable becomes `MANUAL_REVIEW` with the failure in the audit trail.

## Assumptions

* Days/nights for caps: item `quantity` → description ("2 nights", "3 days") → trip dates. Caps scale (`$75 × days`, `$200 × nights`, `$50 × days`) and apply per line item.
* A stated quantity above the trip span (CLM-004: 3 nights on a 3-day trip) is a non-blocking note; Appendix A has no rule for it.
* Expense date = earliest item `expense_date`, else `trip_start` (most conservative). 30 days is compliant, 31 is late.
* Approval tiers use the post-deduction total; pending (business-class/unknown) items count undeducted so POL-APR-03 is still detected.
* `PARTIAL_APPROVE` also covers ineligible items mixed with reimbursable ones; `REJECT` only if nothing is reimbursable.
* Receipts are not required for ineligible items; "greater than $25" is strict.
* `declared_total` (the printed "Total claimed") was added to the five claims for a totals consistency check; no other claim value was changed.

## Limitations

* Live OpenAI model behaviour varies run to run (wording; occasionally needs the repair call).
* No OCR, duplicate detection, currency conversion or employee history.
* Keyword detection of ineligible items in eligible categories is a heuristic that errs toward manual review.
* In-memory policy and audit (not persisted); per-diem caps are per line, not aggregated per day.

## Future Improvements

Persistent audit storage; policy versioning with effective dates; per-day expense records with receipt metadata and OCR; duplicate-claim detection; reviewer workflow UI; evaluation datasets and drift monitoring; multi-currency; authentication/PII protection; centralized policy management; observability, rate limiting, secret management; distributed execution and database-backed claims.

## Interview Discussion Points

* *Why not let the LLM decide?* Money and thresholds are exact rules; the LLM is used where judgement and language help.
* *Why still tool calling?* The model selects checks, handles ambiguity and explains; Python verifies every number.
* *What if the model disagrees with the engine?* Consistency validation rejects it, one repair attempt, then `MANUAL_REVIEW`.
* *How do you prevent fabricated citations?* `ClaimDecision` rejects unknown ids; the explanation must cite the policy behind every manual-review reason.
* *Why 0.00/0.00 for manual review?* No amount is decided yet; inventing one would be arbitrary.
* *How would you test the LLM layer?* Scripted fake client (done), plus golden datasets and monitoring in production.
* *What would you change for production?* See Future Improvements.
