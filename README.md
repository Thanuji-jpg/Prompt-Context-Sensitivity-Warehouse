# Prompt-Context-Sensitivity-Warehouse
Built a Snowflake/dbt data warehouse auditing LLM clinical-response bias across 750+ runs, with 23 dbt tests catching a real join-logic bug before it reached the analysis.

## The story
I ran the same clinical screening scenario (the AUDIT-C alcohol-use questionnaire) through an LLM many times, changing only one thing per batch: which disease was mentioned in the patient's chart (heart disease, kidney disease, diabetes, depression, anxiety, or none), or a demographic detail (age, sex), or the prompt protocol (standard vs. a stricter WHO-guidance version). Each batch came out as its own 5000 row CSV from a separate script run — 15 files, ~750 responses total, each scored for how closely the model's response matched a reference response (word_structural_similarity, semantic_cosine_similarity).

Scattered CSVs are fine for one file. They stop being fine the moment the question becomes cross-cutting: does mentioning kidney disease change how the model responds, compared to no disease at all? Does that effect hold across both prompt protocols? Does it interact with patient sex or age? Answering that means joining conditions that live in different files with slightly different columns. That's exactly the problem a warehouse + a transformation layer solves: land everything in one place, standardize it once, model it as a clean star schema, then let SQL do the cross-cutting analysis instead of copy-pasting across spreadsheets.

Why Snowflake specifically: it's a real cloud warehouse (separate storage and compute, instant elastic warehouses, zero-copy cloning) that's an industry-standard target for dbt, so this project doubles as a showcase of being able to stand up a warehouse, model data properly on top of it, and reason about a real analytical question, all with tools used in production data teams.

## Star schema
```
                dim_disease
                     |
dim_demographic — fct_llm_audit_response — dim_experiment_variant
```
fct_llm_audit_response (grain: one row per model response run) measures: auditc_total_score, word_structural_similarity, semantic_cosine_similarity, zone_mentioned_in_prompt, analysis_eligible, response_status, run_utc
dim_disease — which disease (if any) was injected into the prompt
dim_demographic — biological sex / patient age / whether age was mentioned in the prompt
dim_experiment_variant — which script + prompt protocol (standard vs. WHO-strict) produced the run
