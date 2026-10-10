# JSON Schemas

This directory contains JSON Schema files for validating LLM responses:

- `p1_normalize.schema.json` — step 1 output (threads → claims)
- `p2_induce.schema.json` — step 2 output (claims → rules + singletons)
- `p3_merge.schema.json` — step 3 output (rules → clusters)
- `p4_coverage.schema.json` — step 4 output (coverage analysis)
- `p5_score.schema.json` — step 5 output (impact scores)
- `p7_review.schema.json` — step 7 output (findings + outside_rubric)

Note: p6 (rubric generation) outputs Markdown, so it has no schema.

`run_llm.py` validates each response against the corresponding schema and retries on validation errors.
