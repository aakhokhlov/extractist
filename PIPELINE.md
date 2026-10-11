# Decision log and run log

## Decisions
1. The thread is the unit of analysis (not individual comments or reviews).
2. `reviews.jsonl` is not exported; only inline code review comments are extracted.
3. Conservativeness thresholds: ≥3 threads, ≥2 PRs, ≥2 of 3 runs, confidence ≥0.6.
4. All human input is in `config.yaml`; secrets stay in environment variables.
5. The taxonomy is inferred from work claims (step 1b) and approved by hand before step 2.
6. Model output language is Russian; prompts are in English.
7. All repositories share one team of reviewers and similar domains; rules are not grouped per repository.

## Approved taxonomy
TODO: copy the axes from `data/work/taxonomy.yaml` after approval.

## Runs
TODO: for each run, record the date range, repositories, a copy of `config.yaml` (without secrets), and the counts after each step.

## Limitations
TODO: reviewer diversity, axes coverage, data quality issues.
