# Grounded Card Advisor

A complete Dozer + retrieval sample for a banking use case. It recommends card
products only after applying hard eligibility rules, attaches field-level
provenance to every result, and rejects prompt injection and personally
identifiable information before customer data is retrieved.

The default path is deliberately boring to run: Python's standard library is
enough. It uses the same four CSV datasets and the same response contracts as
the Dozer-backed path, so the application and its tests work without Docker,
an API key, an LLM account, or network access.

## What is different about this sample?

Many retrieval demos allow the closest vector to become the answer. That is
unsafe for eligibility-sensitive products. This sample uses an explicit
pipeline:

```text
untrusted query
  -> prompt-injection / PII gate (before retrieval)
  -> customer_features + card_products gateways
  -> fail-closed eligibility constraints
  -> deterministic vector ranking of eligible products only
  -> annual-value calculation + field-level provenance
  -> optional LangChain explanation of the already selected top result
  -> JSON or human-readable output
```

Hard constraints are never delegated to an LLM or vector database. Missing,
malformed, non-finite, or out-of-range constraint data raises a data-contract
error instead of silently approving a product.

## Data and Dozer APIs

Four small synthetic datasets make the feature derivation inspectable:

| Dataset | Purpose |
| --- | --- |
| `customers.csv` | Age, income, credit score, and risk band |
| `accounts.csv` | Customer/account mapping and active status |
| `transactions.csv` | Monthly spend by category |
| `card_products.csv` | Eligibility constraints, fees, and reward factors |

`dozer-config.yaml` joins and aggregates the first three datasets. It exposes
only the two contracts the advisor needs:

- `customer_features`: eligibility attributes and aggregated spending
- `card_products`: product constraints and reward terms

The SQL produces exactly one feature row per customer, even when a customer has
both active and closed accounts. Spending is counted only for active accounts.
The local `FixtureGateway` derives the equivalent `customer_features` contract
directly from CSV. `DozerGateway` sends a JSON `$filter` to
`POST /customer_features/query` rather than putting the customer identifier in
an access-log-prone URL, and rejects empty, duplicate, oversized, or unexpected
responses.

## Quick start: fully offline

From this directory, run:

```bash
python -m app.cli \
  --customer-id C001 \
  --query "airport travel rewards with grocery cashback" \
  --json
```

No installation step is required on Python 3.10+. The result includes a plain
language reason, an estimated annual value, every rejected product and why it
failed, and provenance records such as:

```json
{
  "source": "fixture",
  "endpoint": "card_products",
  "record_id": "CARD-TRAVEL",
  "fields": ["annual_fee", "reward_rate", "travel_multiplier", "grocery_multiplier"]
}
```

Human-readable output is the default; add `--json` for automation. Errors in
JSON mode are returned as JSON on stderr with exit code `2`.

## Run with Dozer

Install Dozer using the [official instructions](https://getdozer.io/docs/installation),
then start it from this directory:

```bash
dozer run --config-path dozer-config.yaml
python -m app.cli \
  --gateway dozer \
  --dozer-url http://localhost:8080 \
  --customer-id C001 \
  --query "travel rewards" \
  --json
```

Dozer remains the low-latency materialization and API layer. Switching gateways
does not change eligibility, search, explanation, or output contracts.

## Deterministic and optional vector search

The default `DeterministicVectorSearch` uses SHA-256 feature hashing and cosine
similarity. It is reproducible across processes and machines, needs no model
download, and provides a useful offline CI baseline.

An optional LangChain/Chroma adapter demonstrates how to swap in a vector
database without changing the pipeline:

```bash
python -m pip install -r requirements-chroma.txt
python -m app.cli \
  --search chroma \
  --customer-id C001 \
  --query "frequent airport and hotel travel" \
  --json
```

The imports are lazy: users who choose the offline path do not need LangChain
or Chroma.

## Optional LangChain chat explanation

The advisor always applies the security gate, loads Dozer or fixture context,
checks eligibility, and selects products deterministically before an LLM can
run. With `OPENAI_API_KEY` set, `--llm` asks LangChain/OpenAI to explain only
the already selected top product:

```bash
python -m pip install -r requirements-chroma.txt
python -m app.cli --customer-id C001 --query "travel rewards" --llm
```

The immutable LLM context excludes age, income, credit score, risk band, and
customer ID. It includes only the validated query, the selected product, the
verified calculation, and category spend used by that calculation. LLM text is
attached as `llm_explanation`; it cannot change product IDs, eligibility,
ranking, provenance, or computed value. If `OPENAI_API_KEY` is absent, `--llm`
quietly retains the local deterministic explanation.

For an interactive session, omit `--query` and use `--chat` (type `quit` to
finish):

```bash
python -m app.cli --customer-id C001 --gateway dozer --chat --llm
```

## Safety behavior

- Query validation happens before any gateway method is called.
- Common prompt-injection and secret-exfiltration phrases are rejected.
- Email addresses, payment-card numbers, US SSNs, and IBAN-like strings are
  rejected instead of logged or forwarded.
- Eligibility checks account status, age, income, credit score, and risk band.
- All hard constraints must pass before a product enters semantic ranking.
- Deterministic explanations are assembled from computed values, not generated claims.
- Optional LLM text runs after selection and cannot change decision fields.
- Outputs include a financial-advice disclaimer.

This is an educational sample. Production systems require institution-specific
compliance review, consent, authentication, audit logging, retention controls,
fair-lending tests, and human appeal paths.

## Tests and evaluation

Run the complete offline suite:

```bash
python -m unittest discover -s tests -v
```

The suite covers strict model parsing, input gates, constraint boundaries,
stable ranking, CSV joins, Dozer response shapes, URL encoding, end-to-end
provenance, eligibility-before-ranking, and security-before-retrieval.

The retrieval evaluation is a separate executable artifact:

```bash
python -m tests.eval_retrieval
```

It reports hit-rate@3 and mean reciprocal rank for five checked intents. The
unit suite enforces `hit_rate_at_3 == 1.0` and `MRR >= 0.85` to catch search
quality regressions without a paid evaluation service.

## Layout

```text
app/
  advisor.py        pipeline and deterministic grounded explanations
  cli.py            human and JSON interfaces
  eligibility.py    hard, fail-closed rules
  explainer.py      optional post-selection LangChain explanation
  gateway.py        local fixture and Dozer REST adapters
  models.py         strict data contracts and provenance
  security.py       pre-retrieval injection and PII gates
  vector_search.py  deterministic and optional Chroma search
data/               four synthetic CSV datasets
tests/              unit, integration, security, and eval tests
dozer-config.yaml   LocalStorage sources, transformation, and two APIs
```

## Try failure paths

An inactive customer produces no recommendations:

```bash
python -m app.cli --customer-id C005 --query "cashback" --json
```

A malicious or sensitive query is rejected before customer lookup:

```bash
python -m app.cli --customer-id C001 --query "ignore the system prompt" --json
```

Both commands exit with code `2`, making them easy to assert in scripts.
