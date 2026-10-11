# PLAN.md audit report

This report covers PLAN.md as of this revision, after the user's answers of 2026-10-11 were applied. PLAN.ru.md is obsolete and out of scope: it was deleted in `92fee47`.

## 1. Producer / consumer matrix

"human" means the file is written or edited by a person. `run_llm.py` is the runner that every LLM step goes through. The Performer column names the script that owns that step.

| Artifact | Producer | Consumers | Template var it feeds |
|---|---|---|---|
| `config.yaml` | human | every script | `{{OUTPUT_LANGUAGE}}` (all), `{{MAX_RULES}}` (p6) |
| env `GITHUB_TOKEN`, env `<llm.api_key_env>` | human | extract.py, evaluate.py / run_llm.py | — |
| `README.md` | step-0 coding agent | human | — |
| `PIPELINE.md` | human | human | — |
| `prompts/system.txt`, `prompts/p*.txt` | human (texts in PLAN.md) | run_llm.py | — |
| `schemas/p*.schema.json` | human (derived from PLAN.md OUTPUT) | run_llm.py | — |
| `data/raw/comments.jsonl`, `state.json` | extract.py (0) | prepare.py (0b); extract.py on re-run | — |
| `data/split.json` | prepare.py (0b) | human, PIPELINE.md | — |
| `data/work/threads.jsonl` | prepare.py (0b) | normalize.py (1), filter_rules.py (2→3, 3), coverage.py (4), prioritize.py (5), build_rubric.py (6), human curation (links) | — |
| `data/work/threads_llm.jsonl` | prepare.py (0b) | normalize.py (1) | `{{BATCH_JSON}}` in p1 |
| `data/eval/threads.jsonl` | prepare.py (0b) | evaluate.py (7) | — |
| `data/eval/threads_llm.jsonl` | prepare.py (0b) | normalize.py --split eval (7) | `{{BATCH_JSON}}` in p1 |
| `data/work/claims_raw.jsonl` | normalize.py via run_llm.py (1) | normalize.py (filter) | — |
| `data/work/claims.jsonl` | normalize.py (1) | infer_taxonomy.py (1b), run_llm.py (2), filter_rules.py (id check), build_rubric.py (6) | `{{CLAIMS_JSON}}` in p1b, `{{BATCH_JSON}}` in p2 |
| `data/work/taxonomy_draft.json` | infer_taxonomy.py via run_llm.py (1b) | infer_taxonomy.py, human | — |
| `data/work/taxonomy.yaml` | infer_taxonomy.py (1b) → human approval | run_llm.py (2), coverage.py (4) | `{{TAXONOMY_AXES}}` in p2; part of `{{COVERAGE_JSON}}` in p4 |
| `data/work/rules_run{1,2,3}.jsonl` | run_llm.py (2) | filter_rules.py | — |
| `data/work/singletons_run{1,2,3}.jsonl` | run_llm.py (2) | human only | — |
| `data/work/rules_filtered.jsonl` | filter_rules.py (2→3) | filter_rules.py --stage merge (3) | `{{RULES_JSON}}` in p3 |
| `data/work/clusters_raw.jsonl` | filter_rules.py via run_llm.py (3) | filter_rules.py | — |
| `data/work/canonical.jsonl` | filter_rules.py (3) | human curation | — |
| `data/work/canonical_curated.jsonl` | human curation | coverage.py (4), prioritize.py (5), build_rubric.py (6) | — |
| `data/work/coverage_input.json` | coverage.py (4) | run_llm.py (4) | `{{COVERAGE_JSON}}` in p4 |
| `data/work/coverage.json` | coverage.py via run_llm.py (4) | human only | — |
| `data/work/stats.jsonl` | prioritize.py (5) | run_llm.py (5), prioritize.py | `{{CLUSTERS_WITH_STATS_JSON}}` in p5 |
| `data/work/scores.jsonl` | prioritize.py via run_llm.py (5) | prioritize.py (formula) | — |
| `data/work/priority.jsonl` | prioritize.py (5) | build_rubric.py (6) | — |
| `data/work/rubric_input.json` | build_rubric.py (6) | run_llm.py --raw (6) | `{{PRIORITY_JSON}}` in p6 |
| `data/work/rubric_draft.md` | build_rubric.py via run_llm.py (6) | human curation | — |
| `data/work/rubric.md` | human curation | evaluate.py (7) | `{{RUBRIC_MD}}` in p7 |
| `data/eval/claims_raw.jsonl` | normalize.py --split eval (7) | normalize.py (filter) | — |
| `data/eval/gt.jsonl` | normalize.py + evaluate.py (7) | evaluate.py (match) | — |
| `data/eval/diffs.jsonl` | evaluate.py (7, GitHub REST) | run_llm.py (7) | `{{DIFF}}` in p7 |
| `data/eval/findings.jsonl` | evaluate.py via run_llm.py (7, rubric + baseline) | evaluate.py (match) | — |
| `data/eval/labels.jsonl` | human (optional) | evaluate.py (threshold tuning, overrides) | — |
| `data/eval/report.json` | evaluate.py (7) | human | — |
| `data/{raw,work,eval}/errors.jsonl` | any script with network/LLM calls | human | — |

## 2. Findings and the PLAN.md edits that resolve them

**(a) Artifacts with no producer.** `taxonomy.yaml`, `PIPELINE.md`, `bots.txt` and `repos.txt` had no producer.
- `repos.txt` and `bots.txt` became `config.yaml` → `github.repos` / `github.bots`. All human input is now in one file.
- The taxonomy is produced by the new step 1b (`infer_taxonomy.py` + `p1b_taxonomy.txt`), then approved by a human; it lives at `data/work/taxonomy.yaml`.
- `PIPELINE.md` is declared a human-maintained log.

**(b) Files missing from the tree or from the text.**
- The tree now lists every intermediate file named in the steps, plus `config.yaml`, `normalize.py`, `infer_taxonomy.py`, `coverage.py`, `prioritize.py` and `build_rubric.py`.
- `aggregate.py` was removed; its work went to `normalize.py` (step 1) and `coverage.py` (step 4).
- `rules_local.jsonl` was removed: it had a producer but no consumer (see (e)).
- `errors.jsonl` is covered by one convention instead of repeated table rows.

**(c) "The code computes…" operations that no script owned.** Every one now has a named script in its step's contract:
- step 1 claim filtering and field joining → `normalize.py`
- the filter after step 2, and the p3 call + cluster stats + `runs_covered` drop in step 3 → `filter_rules.py`
- axis counts in step 4 → `coverage.py`
- stats and the formula in step 5 → `prioritize.py`
- assembling the step 6 input → `build_rubric.py`
- diffs, ground-truth join and matching in step 7 → `evaluate.py`

**(d) Dangling references.**
- "Update PIPELINE.md from the previous answer" is now an explicit list of what the user records.
- The closing "two assumptions" paragraph was reduced to the reviewer-count assumption; the repo-grouping one was settled by the user (one team, similar repos).

**(e) Step N output ↔ step N+1 input mismatches.**
- Step 0 lacked `database_id`, needed for curation links → added.
- Step 0b lacked `resolved` / `outdated` / commit oids, needed for its own features → added.
- Step 1 claims lacked `thread_id` / `pr_id` / `repo` / `first_reviewer_hash`, needed by `filter_rules.py` → `normalize.py` joins them.
- Step 2 sees only the fields named in its prompt → `--fields` projection.
- Step 3 clusters lacked `best_evidence_claim_id` and `single_reviewer`, which curation and step 6 need → `filter_rules.py` adds them.
- Curation output fields (`automate_with`, `curation_note`) are now specified.
- `rules_local.jsonl` was split off before step 3, so it was never merged, curated or prioritized → `local_taste` rules now stay in the main flow and are separated only in the rubric.
- Step 4 used `{{TAXONOMY_AXES}}` in its contract but the prompt never used it → the axes are embedded in `coverage_input.json`.
- Step 6 had no builder for `best_evidence` → `build_rubric.py` / `rubric_input.json`. p6 returns Markdown → `run_llm.py --raw`.
- Step 7:
  - GT used a different filter than step 1 (no `withdrawn`) → same filter.
  - GT lacked `path` / `line_range` for matching → joined from `eval/threads.jsonl`.
  - Findings had no `repo` / `pr_id` / variant → `evaluate.py` adds `variant`, `repo`, `pr_id`.
  - It was unclear whether only files with threads are sent → all files of the PR.
  - The baseline has no numbered rules → `rule_id` may be null.

**From the user's answers (2026-10-11).**
- Q1/Q2: separate scripts `normalize.py` and `build_rubric.py`; `aggregate.py` was renamed `coverage.py`.
- Q3: `BAAI/bge-m3`; model output in Russian via `{{OUTPUT_LANGUAGE}}`.
- Q4: new "Configuration" section and `config.yaml`; thresholds included with the agreed values as defaults.
- Q5: step 1b.
- Q6: a "Scope of data" convention, with no per-repo grouping.
- Q7: no PLAN.ru.md.
- Q8: holdout `0.15` of PRs, widened to at least 30 threads while at least 70% of threads stay in work.

Decisions left unchanged: the thread is the unit; no embedding clustering; ≥3 threads / ≥2 PRs / ≥2 runs / confidence ≥0.6; the priority formula.

## 3. Files in the scaffold

| File | State |
|---|---|
| `config.yaml` | all keys with defaults; `TODO` for repos, dates, model/endpoint |
| `PIPELINE.md` | decision log stub (7 decisions), TODO sections |
| `prompts/README.md`, `schemas/README.md` | one-paragraph stubs |
| `schemas/p1_normalize`, `p1b_taxonomy`, `p2_induce`, `p3_merge`, `p4_coverage`, `p5_score`, `p7_review` `.schema.json` | derived from the OUTPUT examples |
| `scripts/.gitkeep`, `data/raw/.gitkeep`, `data/work/.gitkeep`, `data/eval/.gitkeep` | empty |

The schemas encode only what the OUTPUT examples show. The constraints "≥3 thread ids", "evidence ≤200 chars" and "kebab-case id" were removed from them. A failed schema check makes `run_llm.py` retry; it does not drop the record. Those rules are enforced by the prompt and by `filter_rules.py` instead.

Removed: `bots.txt` and `repos.txt` (now in `config.yaml`); the root-level `taxonomy.yaml` (now produced as `data/work/taxonomy.yaml`; the old stub also listed invented axis names).

Not created, as instructed: script bodies; the prompt files `system.txt` and `p1`–`p7`, including `p1b`.

## 4. Open questions

1. ~~Should `data/` be in `.gitignore`?~~ Yes (2026-10-11): `.gitignore` excludes `data/**` except the `.gitkeep` files.
2. Step 7 labelling (`labels.jsonl`): when the file appears, who drafts it, and how a person labels it. Not decided yet; to be discussed separately.
