# CLAUDE.md

Context for Claude Code working in this repository.

## What this is

A tiered AML (anti-money-laundering) transaction screening prototype. A ledger
CSV passes through deterministic compliance rules, a behavioural anomaly model,
a retrieval-grounded LLM assessment, and a stateful topology agent that can
suspend execution for human authorisation.

Streamlit UI, Qdrant for vector search, LangGraph for the stateful agent,
scikit-learn for the anomaly model.

Inference is provider-agnostic: `MODEL_PROVIDER=bedrock` (AWS, default) or
`vertex` (GCP). All model construction goes through `utils/llm_provider.py`.
Never import `ChatBedrockConverse` or `ChatVertexAI` outside that module.

On Bedrock the chat model must support tool use, because verdicts come back
through `with_structured_output`. Anthropic and Nova models do; Titan text
models do not. Setup: SETUP_AWS.md (AWS) or SETUP.md (GCP).

## The invariant that must not be broken

**Deterministic compliance rules run BEFORE the statistical funnel, and the
funnel may never drop a row a rule flagged.**

An earlier version had this backwards: `IsolationForest` fitted on transaction
amount ran first and discarded anything statistically ordinary. Structuring is
*defined* by ordinary-looking amounts, so the cost optimisation was deleting the
fraud the later tiers existed to catch. Recall was 10%.

If you are changing `utils/funnel.py`, `utils/rules.py`, or the pipeline order in
`app.py`, run `python scripts/evaluate.py` before and after. Recall must stay at
100% on the labelled set. A change that improves precision by lowering recall is
a regression, not an optimisation.

## Rules specific to this codebase

1. **Nothing maps a failure to CLEAR.** `Verdict.ERROR` exists precisely so an
   API timeout cannot produce a clean audit. Never add an `except` that returns
   or defaults to `Verdict.CLEAR`.

2. **Verdicts come from `with_structured_output`, never from parsing prose.**
   Keyword-searching model output for "suspicious" matches "not suspicious" and
   "no evidence of money laundering". If you need a new decision from the model,
   add a field to `utils/schemas.py`.

3. **Money is `Decimal`.** `utils/data_loader.parse_money` is the only place
   currency strings are converted. `amount_float` exists solely because
   scikit-learn cannot consume `Decimal`; never compare or total with it.

4. **Graph nodes copy before mutating.** `findings = list(state[...])`, not
   `findings = state[...]`. In-place mutation corrupts checkpoint replay.

5. **Ledger content is untrusted.** Any field from a CSV that reaches a prompt
   goes through `utils/sanitize.build_evidence_block` first. The adversary in a
   fraud system is the party being audited.

6. **Thresholds live in `config.py`.** No new magic numbers in module bodies.

7. **Model construction goes through `utils/llm_provider.py`.** Adding a
   provider means adding a factory function there, not a conditional import in
   a consumer module.

8. **Never write credentials into the repo.** No service-account JSON in the
   project root, no keys in `.env.example`, no secrets in test fixtures.
   Authentication is Application Default Credentials.

## Layout

```
config.py                 all thresholds, paths, model names
app.py                    Streamlit UI and pipeline orchestration
utils/
  data_loader.py          CSV validation, Decimal currency parsing
  rules.py                Tier 1 deterministic rules (runs first)
  features.py             12 behavioural features for the anomaly model
  funnel.py               Tier 0 anomaly model, robust MAD threshold
  agents.py               Tier 2 LLM assessment, fail-closed
  chat_agent.py           Vertex + Qdrant RAG provider (singleton)
  graph_nodes.py          Tier 3 scoring as PURE functions, no LangGraph import
  graph_logic.py          LangGraph assembly only
  schemas.py              Pydantic output contracts
  verdicts.py             Verdict enum, AuditOutcome (stdlib only)
  sanitize.py             prompt-injection defence
  screening.py            sanctions fuzzy matching
  audit_log.py            SQLite append-only trail
scripts/
  preflight.py            verify environment before running
  generate_sample_ledger.py
  evaluate.py             per-typology recall/precision
  ingest_pdf.py           index the regulatory corpus
tests/test_pipeline.py    36 offline regression tests
```

`graph_nodes.py` is deliberately free of LangGraph imports so scoring is
testable offline. Keep it that way — put graph wiring in `graph_logic.py`.

## Commands

```bash
python scripts/preflight.py            # check environment (start here)
pytest tests/ -v                       # 36 tests, no network needed
python scripts/evaluate.py             # recall/precision vs the labelled set
streamlit run app.py
```

## Before you say a change is done

- `pytest tests/ -v` passes
- `python scripts/evaluate.py` still shows 100% recall
- No new hardcoded thresholds
- Nothing you added can return CLEAR on an error path
