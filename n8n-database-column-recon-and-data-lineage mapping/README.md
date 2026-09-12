# AI-Driven Column Reconciliation & Data Lineage (n8n + OpenRouter)

An n8n workflow that reconciles column-level schemas between two databases using an LLM, producing a matched-column mapping (data lineage) and a flagged review queue for ambiguous or missing matches.

## Problem

When two systems evolve independently (a source CRM and a downstream analytics warehouse, for example), their schemas drift: columns get renamed, merged, split, reunited under synonyms, or dropped entirely. Manually reconciling column-level lineage across 100+ columns is slow and error-prone. This project automates the first pass with an LLM and produces a structured output a human can review in minutes instead of hours.

## What this does

1. Loads two synthetic schemas, **System A** (`crm_core`, 100 columns) and **System B** (`analytics_warehouse`, 90 columns), each with column name, data type, description, sample values, and source table.
2. Sends both schemas to an LLM (via OpenRouter, model configurable) with instructions to match columns by semantic meaning, not just name similarity: same concept can differ in name, unit, granularity, or be split/merged across columns.
3. Parses the model's structured JSON output and splits it into two review files:
   - `matched_columns.csv` — confidence >= 0.6
   - `needs_review_columns.csv` — confidence < 0.6, explicitly unmatched, or B-only columns with no A counterpart

## Why this isn't a trivial name-matching exercise

The synthetic data is built with deliberate structural drift, not just clean 1:1 renames:

| Pattern | Count | What it tests |
|---|---|---|
| Identical | 10 | Baseline / sanity check |
| Renamed (naming convention drift) | 20 | Abbreviation and convention handling (`billing_postal_code` -> `bill_zip`) |
| Synonym (lexically distant, same meaning) | 15 | Semantic matching beyond string similarity (`loyalty_points_balance` -> `reward_credits`) |
| Unit change | 10 | Matching despite unit conversion (`weight_kg` -> `weight_lb`, `unit_price_usd` -> `unit_price_cents`) |
| Split (1 -> 2 columns) | 4 | One-to-many mapping detection |
| Merge (2 -> 1 column) | 3 | Many-to-one mapping detection |
| **Decoy pairs** | 6 | Two near-identical B columns compete for one A column; only one is the true match |
| Dropped (A-only) | 29 | Correctly identifying "no match exists" instead of forcing one |
| Net-new (B-only) | 12 | Columns that exist downstream with no upstream source |

The six decoy pairs are the hardest part of the dataset. For example, `related_order_id` in System A has two plausible-looking counterparts in System B: `order_ref_id` (the correct match) and `order_id` (a decoy that looks like the right answer but is actually the primary order identifier, not the support-ticket cross-reference). These aren't flagged anywhere in the prompt. The workflow's own confidence scoring is what should expose the ambiguity, and the `needs_review` queue is where a human catches what the model got wrong or wasn't sure about. Ground truth for all 114 mappings, including which decoy is correct, is in `ground_truth.json` for scoring the AI's output after a run; it is never given to the model.

## Workflow structure

```
Manual Trigger
  -> Load Both Schemas (embeds system_a.json / system_b.json)
  -> Build Reconciliation Prompt (Code node)
  -> AI Reconciliation Agent (LangChain Agent node)
       <- OpenRouter Chat Model (model configurable, default claude-sonnet-4.5)
  -> Parse & Validate AI Output (Code node: JSON validation + confidence threshold split)
  -> Split: Matched Rows       -> Convert to CSV (matched_columns.csv)
  -> Split: Needs Review Rows  -> Convert to CSV (needs_review_columns.csv)
```

Import `column_reconciliation_workflow.json` directly into n8n (Workflows -> Import from File).

## Setup

1. n8n instance (cloud or self-hosted) with the LangChain nodes available (bundled by default in recent n8n versions).
2. An OpenRouter API key, added as an n8n credential for the OpenRouter Chat Model node.
3. Import the workflow JSON, select your OpenRouter credential on the `OpenRouter Chat Model` node, execute manually.

To point this at real databases instead of the embedded demo JSON, replace the `Load Both Schemas` node with two source nodes (Postgres/MySQL `information_schema.columns` query, or an HTTP Request against a schema registry) feeding a `Merge` node (`combinationMode: mergeByPosition`) upstream of `Build Reconciliation Prompt`.

## Files

- `system_a.json` / `system_b.json` — the two synthetic schemas
- `ground_truth.json` — full mapping key including decoy answers, for scoring only
- `column_reconciliation_workflow.json` — importable n8n workflow
- `generate_data.py` / `generate_system_b.py` — regenerate or modify the synthetic datasets
- `build_workflow.py` — regenerate the workflow JSON if the prompt or node structure changes

## Known limitations

- Confidence threshold (0.6) is a fixed heuristic, not calibrated against a labeled validation set. A production version would tune this against measured precision/recall on held-out ground truth.
- Single LLM call for the full 100x90 column set works at this scale but will hit context and reliability limits well before 1,000+ columns; a production version would batch by table or entity and reconcile incrementally.
- No retry/backoff on the OpenRouter call and no schema-validation retry loop if the model's JSON output is malformed; the Code node throws on invalid JSON rather than requesting a corrected response.
