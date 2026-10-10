# Prompts

This directory contains the prompt files for the LLM pipeline steps:

- `system.txt` — common system prefix prepended to all prompts (steps 1-7)
- `p1_normalize.txt` — step 1: extract claims from review threads
- `p2_induce.txt` — step 2: derive rules from claims (run 3×)
- `p3_merge.txt` — step 3: merge rules across runs and deduplicate
- `p4_coverage.txt` — step 4: analyze coverage by taxonomy axes
- `p5_score.txt` — step 5: score rules by impact
- `p6_rubric.txt` — step 6: generate the reviewer rubric (Markdown output)
- `p7_review.txt` — step 7: apply the rubric to diffs for validation

Template variables in prompts (e.g., `{{BATCH_JSON}}`, `{{TAXONOMY_AXES}}`) are substituted by `run_llm.py`.

Full prompt texts are specified in PLAN.md.
