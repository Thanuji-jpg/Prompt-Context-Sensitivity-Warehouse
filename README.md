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
1) fct_llm_audit_response (grain: one row per model response run) measures: auditc_total_score, word_structural_similarity, semantic_cosine_similarity, zone_mentioned_in_prompt, analysis_eligible, response_status, run_utc
2) dim_disease — which disease (if any) was injected into the prompt
3) dim_demographic — biological sex / patient age / whether age was mentioned in the prompt
4) dim_experiment_variant — which script + prompt protocol (standard vs. WHO-strict) produced the run

Dimensions are wide and denormalized on purpose: a BI tool or analyst can join the fact table to any dimension with a single equality join and no further chasing. That's the entire value proposition of a star schema over a normalized (3NF) schema — it trades some redundancy in the dimensions for simple, fast, predictable queries.

## Pipeline layers
```
seeds/          15 raw CSVs, loaded as-is (dbt seed)
      |
staging/        stg_audit_responses         -> union all 15 seeds (dbt_utils.union_relations)
                 stg_audit_responses_typed   -> cast types, normalize booleans/blanks
      |
marts/          dim_disease, dim_demographic, dim_experiment_variant, fct_llm_audit_response
```
Each seed file has a slightly different column set (some have patient_age, some have biological_sex, some have neither). dbt_utils.union_relations unions them into one wide table, filling NULL for whatever a given seed doesn't have — the union happens once, in one place, instead of being hand-coded per file.

## Example questions this schema answers
```
-- Does mentioning a disease shift how closely the response matches
-- the reference, compared to the no-disease baseline?
select
    d.disease_name,
    avg(f.semantic_cosine_similarity) as avg_similarity,
    count(*) as n_runs
from fct_llm_audit_response f
join dim_disease d on f.disease_key = d.disease_key
group by 1
order by 2 desc;

-- Does the WHO-strict protocol change scores relative to the standard one?
select
    v.protocol,
    avg(f.auditc_total_score) as avg_score,
    avg(f.semantic_cosine_similarity) as avg_similarity
from fct_llm_audit_response f
join dim_experiment_variant v on f.variant_key = v.variant_key
group by 1;
```
## Results
Confirmed by running the query above against the live Snowflake tables (LLM_BIAS_AUDIT.ANALYTICS_MARTS), after dbt seed && dbt run && dbt test all passed:
```
Disease injected in prompt	n	avg semantic similarity to reference
Diabetes: Present	100	0.601
No disease (baseline)	400	0.593
Heart Disease: Present	100	0.592
Kidney Disease: Present	50	0.494
Anxiety Disorder: Present	50	0.489
Depression: Present	50	0.482
```

Reading: mentioning kidney disease, anxiety, or depression pulls the model's response noticeably further from the reference answer than the no-disease baseline — the model isn't just answering the alcohol-screening question, it's injecting disease-specific commentary that changes the substance of the response. Diabetes and heart disease barely move the needle. Splitting by prompt protocol shows the effect is even more visible under standard prompting (avg similarity 0.515) than under the WHO-strict, reference-guided protocol (0.689) — the stricter protocol constrains the model closer to the reference regardless of what's mentioned. Sex showed a small difference (female 0.592 vs. male 0.609) on the no-disease baseline runs.

Caveat, stated honestly: each group is 50–100 runs — enough to show a consistent, repeatable pipeline and a visible pattern, not enough to claim statistical significance. The value of this project is the pipeline and the methodology, not a clinical claim.

## Testing, and a real bug it caught
marts.yml declares 23 dbt tests across the four models: unique + not_null on every surrogate key, relationships tests on every foreign key in fct_llm_audit_response (fact → dimension), and accepted_values on dim_experiment_variant.protocol. Each test compiles to a select that should return zero rows; any row returned is a failure.

The first dbt test run failed not_null on fct_llm_audit_response.demographic_key. Root cause: 3 of the 15 seed files don't have an age_in_prompt column at all, so after dbt_utils.union_relations stacked all 15 seeds together, those rows got NULL there. The original staging logic did (age_in_prompt = 'True') — but in SQL, anything compared to NULL evaluates to NULL, not false. That NULL flowed into the join condition in fct_llm_audit_response, and NULL = NULL is also not true (SQL's three-valued logic), so the join silently failed to match for ~150 rows.

Fix: coalesce(age_in_prompt, 'False') = 'True' in stg_audit_responses_typed.sql, so missing data becomes a real FALSE before the comparison runs. One line, fixed at the staging layer, which propagated automatically through the dimension and fact models on the next dbt run. Re-running dbt test afterward: PASS=23 WARN=0 ERROR=0.

## Setup
1) Install deps (dbt-core + dbt-snowflake are already installed in this environment; this pulls the dbt_utils package):
```
cd "llm_bias_audit"
dbt deps
```
2) Configure your Snowflake connection in ~/.dbt/profiles.yml (kept outside the repo on purpose — never commit credentials). See profiles.yml.example in this folder for the template this project expects, and export the referenced environment variables before running dbt.

3) Load the seeds, build the models, and run the tests:
```
dbt seed
dbt run
dbt test
dbt docs generate && dbt docs serve
```



