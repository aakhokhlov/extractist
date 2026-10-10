# Pipeline Audit and Implementation Report

This report delivers the three parts requested: (1) artifact matrix with findings, (2) PLAN.md edits applied, (3) file scaffold created.

---

## PART 1: Artifact Producer/Consumer Matrix

### Artifacts and Dependencies

| Artifact | Producer | Consumers | Template Variables Used | Notes |
|----------|----------|-----------|------------------------|-------|
| `repos.txt` | **human** | extract.py (step 0) | — | |
| `bots.txt` | **human** | prepare.py (step 0b) | — | |
| `taxonomy.yaml` | **human** | run_llm.py (steps 2, 4) | `{{TAXONOMY_AXES}}` | |
| `README.md` | **extract.py (step 0)** | human | — | Written by coding agent with script |
| `PIPELINE.md` | **human** | — | — | Decision/run log, manually maintained |
| `data/raw/comments.jsonl` | extract.py (step 0) | prepare.py (step 0b) | — | |
| `data/raw/state.json` | extract.py (step 0) | extract.py (next run) | — | Checkpoint for resuming |
| `data/split.json` | prepare.py (step 0b) | human (reference) | — | Holdout split boundary |
| `data/work/threads.jsonl` | prepare.py (step 0b) | aggregate.py (step 1), filter_rules.py (step 2→3), prioritize.py (step 5), step 6 enrichment | — | Full thread features |
| `data/work/threads_llm.jsonl` | prepare.py (step 0b) | run_llm.py (step 1) | `{{BATCH_JSON}}` in p1 | Redacted view for LLM |
| `data/eval/threads.jsonl` | prepare.py (step 0b) | evaluate.py (step 7) | — | Holdout with features |
| `data/eval/threads_llm.jsonl` | prepare.py (step 0b) | evaluate.py (step 7, p1 call) | `{{BATCH_JSON}}` in p1 | Holdout redacted view |
| `data/work/claims_raw.jsonl` | run_llm.py (step 1, via aggregate.py) | aggregate.py (step 1 filter) | — | Raw LLM output |
| `data/work/claims.jsonl` | aggregate.py (step 1 filter) | run_llm.py (step 2), step 6 enrichment | `{{BATCH_JSON}}` in p2 | Filtered claims |
| `data/work/rules_run{1,2,3}.jsonl` | run_llm.py (step 2) | filter_rules.py (step 2→3) | — | Three independent runs |
| `data/work/singletons_run{1,2,3}.jsonl` | run_llm.py (step 2) | human (optional review) | — | Unmerged claims |
| `data/work/rules_filtered.jsonl` | filter_rules.py (step 2→3) | filter_rules.py (step 3, p3 call) | `{{RULES_JSON}}` in p3 | After thresholds |
| `data/work/rules_local.jsonl` | filter_rules.py (step 2→3) | step 6 enrichment | — | Local taste rules, rejoins at step 6 |
| `data/work/clusters_raw.jsonl` | run_llm.py (step 3, via filter_rules.py) | filter_rules.py (step 3 filter) | — | Raw merge output |
| `data/work/canonical.jsonl` | filter_rules.py (step 3 filter) | **human (curation)** | — | After runs_covered≥2 filter |
| `data/work/canonical_curated.jsonl` | **human (curation)** | aggregate.py (step 4), prioritize.py (step 5), step 6 enrichment | — | After manual review |
| `data/work/coverage_input.json` | aggregate.py (step 4) | run_llm.py (step 4) | `{{COVERAGE_JSON}}` in p4 | Computed stats by axis |
| `data/work/coverage.json` | run_llm.py (step 4, via aggregate.py) | human (reference) | — | Coverage interpretation |
| `data/work/stats.jsonl` | prioritize.py (step 5) | run_llm.py (step 5, via prioritize.py) | `{{CLUSTERS_WITH_STATS_JSON}}` in p5 | Outcome stats |
| `data/work/scores.jsonl` | run_llm.py (step 5, via prioritize.py) | prioritize.py (step 5 formula) | — | LLM impact scores |
| `data/work/priority.jsonl` | prioritize.py (step 5 formula) | step 6 enrichment | — | Final priority values |
| `data/work/rubric_draft.md` | run_llm.py (step 6) | **human (curation)** | — | Generated rubric |
| `data/work/rubric.md` | **human (curation)** | evaluate.py (step 7) | `{{RUBRIC_MD}}` in p7 | Curated rubric |
| `data/eval/claims_raw.jsonl` | evaluate.py (step 7, p1 on holdout) | evaluate.py (step 7 GT filter) | — | Ground truth extraction |
| `data/eval/gt.jsonl` | evaluate.py (step 7 GT filter) | evaluate.py (step 7 matching) | — | Filtered ground truth |
| `data/eval/diffs.jsonl` | evaluate.py (step 7, GitHub API) | run_llm.py (step 7, p7 call) | `{{DIFF}}` in p7 | Fetched PR patches |
| `data/eval/findings.jsonl` | run_llm.py (step 7, via evaluate.py, ×2) | evaluate.py (step 7 matching) | — | Rubric + baseline runs |
| `data/eval/labels.jsonl` | **human (optional)** | evaluate.py (step 7 matching) | — | Manual match labels for tuning |
| `data/eval/report.json` | evaluate.py (step 7 matching) | human | — | Validation metrics |
| `prompts/system.txt` | **human** | run_llm.py (steps 1–7) | — | Prepended to all prompts |
| `prompts/p1_normalize.txt` | **human** | run_llm.py (steps 1, 7) | — | |
| `prompts/p2_induce.txt` | **human** | run_llm.py (step 2) | — | |
| `prompts/p3_merge.txt` | **human** | run_llm.py (step 3) | — | |
| `prompts/p4_coverage.txt` | **human** | run_llm.py (step 4) | — | |
| `prompts/p5_score.txt` | **human** | run_llm.py (step 5) | — | |
| `prompts/p6_rubric.txt` | **human** | run_llm.py (step 6) | — | |
| `prompts/p7_review.txt` | **human** | run_llm.py (step 7) | — | |
| `schemas/p*.schema.json` | **human** | run_llm.py (steps 1–5, 7) | — | Validation schemas |
| `errors.jsonl` | Any script that fails | human (debugging) | — | Multiple locations in tree |

### Findings

#### (a) Artifacts with no producer (human-provided inputs)
- ✓ `taxonomy.yaml` — correctly marked as human input
- ✓ `PIPELINE.md` — correctly marked as human-maintained
- ✓ `bots.txt` — correctly marked as human input
- ✓ `repos.txt` — correctly marked as human input
- ✓ All prompts and schemas — correctly marked as human-authored

#### (b) Files missing from tree or text
- **RESOLVED**: Tree initially omitted `scripts/prioritize.py`; added in tree with comment "stats, LLM scoring, priority formula (5)"; step 5 contract now references it
- **RESOLVED**: `errors.jsonl` appeared in some contract Output fields but not all; added note in conventions paragraph clarifying it's omitted for brevity but assumed present

#### (c) Operations with no script owner
All "code computes/appends" operations now assigned:
- **Step 1 claim filtering + field appending** → `aggregate.py`
- **Step 2→3 threshold filtering + split** → `filter_rules.py`
- **Step 3 p3 call + cluster aggregation + runs_covered<2 drop** → `filter_rules.py`
- **Step 4 axis counting** → `aggregate.py`
- **Step 5 stats aggregation + formula** → `prioritize.py`
- **Step 6 enrichment (joining priority + canonical_curated + claims + threads)** → noted in contract; implementation in `run_llm.py` or helper
- **Step 7 all operations** → `evaluate.py`

#### (d) Dangling references
- **RESOLVED**: "Update PIPELINE.md from the previous answer" replaced with: "The user maintains `PIPELINE.md` manually as a decision log and run log, recording: the thread as the unit; `reviews.jsonl` is not exported; conservativeness thresholds (≥3 threads, ≥2 PRs, ≥2 runs); date ranges; observed limitations (reviewer diversity, axes coverage)."

#### (e) Input/output mismatches
- **RESOLVED**: Step 0b contract now explicitly lists all output fields including `database_id`, `resolved`, `outdated` per the enriched schema
- **RESOLVED**: Step 6 enrichment process now documented: code joins priority.jsonl + canonical_curated.jsonl + claims.jsonl + threads.jsonl to build the full input
- **RESOLVED**: `rules_local.jsonl` flow clarified: split after step 2, skips steps 3–5, rejoins at step 6 with `priority=null` for the "Conventions" section
- **MINOR OPEN**: Step 7 produces `findings.jsonl` with `rule_id/path/line_start/line_end/comment/confidence` but the matching logic in evaluate.py needs to distinguish rubric vs baseline runs (implementation detail, not a spec gap)

---

## PART 2: PLAN.md Edits Applied

All edits were minimal, targeted patches tied to findings:

1. **Tree correction** (finding b): Added `scripts/prioritize.py` to tree; changed aggregate.py comment from "stats + priority (5)" to "claims (1), coverage (4)"; added prioritize.py comment "stats, LLM scoring, priority formula (5)"

2. **Contract tables: errors.jsonl removal** (finding b): Removed `errors.jsonl` from Output fields in contract tables for steps 1–7 (it's written by run_llm.py as needed, not always by every step); added note after conventions paragraph: "Note: `errors.jsonl` appears in multiple contract Output fields but is omitted from later ones for brevity; assume all LLM-calling steps can produce errors."

3. **Step 6 enrichment documentation** (finding e): Added paragraph before step 6 prompt explaining how the code enriches priority.jsonl with best_evidence details from canonical_curated.jsonl, claims.jsonl, threads.jsonl

4. **rules_local.jsonl flow** (finding e): Added sentence to step 6 enrichment paragraph: "Rules from `rules_local.jsonl` (split off after step 2) are appended to this input with `priority=null` and `scope=local_taste`; they skip steps 3–5 but appear in the 'Conventions of this repository' section of the rubric."

5. **PIPELINE.md sentence replacement** (finding d): Already applied in previous session; reconfirmed present

---

## PART 3: File Scaffold Created

### Directories
- `scripts/` — empty, ready for Python scripts
- `prompts/` — contains `README.md` describing the 8 prompt files
- `schemas/` — contains `README.md` + 6 JSON Schema files
- `data/raw/`, `data/work/`, `data/eval/` — empty, ready for pipeline output

### Files Created

#### Human inputs (placeholders)
- `bots.txt` — comment + TODO
- `repos.txt` — comment + TODO
- `PIPELINE.md` — stub with 3 decisions + sections for runs and limitations
- `taxonomy.yaml` — stub with 12 TODO axes

#### Documentation
- `prompts/README.md` — describes the 8 prompt files and template variable substitution
- `schemas/README.md` — describes the 6 schema files (no schema for p6, which outputs Markdown)

#### JSON Schemas (derived mechanically from OUTPUT examples in PLAN.md)
- `schemas/p1_normalize.schema.json` — validates `{"threads":[{"thread_id","claims":[{"claim_id","category","normalized","evidence","code_anchor","severity_hint","language","needs_context","withdrawn"}],"reason"}]}`
- `schemas/p2_induce.schema.json` — validates `{"rules":[{"id","statement","axis","scope","applies_to","evidence_thread_ids","best_evidence_claim_id","counterexample","confidence"}],"singletons":[{"thread_ids","note"}]}`
- `schemas/p3_merge.schema.json` — validates `{"clusters":[{"cluster_id","statement","axis","scope","applies_to","member_rule_ids","counterexample","conflict","conflict_note","merge_doubt"}]}`
- `schemas/p4_coverage.schema.json` — validates `{"axes":[{"axis","blind_spot_note"}],"clusters":[{"cluster_id","axis_suggested","repetitive_note"}]}`
- `schemas/p5_score.schema.json` — validates `{"scores":[{"cluster_id","s","insufficient_data","reasoning"}]}`
- `schemas/p7_review.schema.json` — validates `{"findings":[{"rule_id","path","line_start","line_end","comment","confidence"}],"outside_rubric":[{"path","line_start","note"}]}`

### NOT Created (as instructed)
- Script bodies: `extract.py`, `prepare.py`, `run_llm.py`, `filter_rules.py`, `aggregate.py`, `prioritize.py`, `evaluate.py`
- Prompt files: `system.txt`, `p1_normalize.txt`, `p2_induce.txt`, `p3_merge.txt`, `p4_coverage.txt`, `p5_score.txt`, `p6_rubric.txt`, `p7_review.txt`
- `PLAN.ru.md` — confirmed absent (deleted in commit 92fee47)

---

## Open Questions for the User

1. **Step 1 script ownership**: Should step 1 (p1 call + claim filtering) remain in `aggregate.py`, or should it move to a separate script? Currently aggregate.py owns step 1 + step 4.

2. **Step 6 enrichment implementation**: The spec says "the code (in `run_llm.py` or a helper)" builds the enriched input. Should this be:
   - A function inside run_llm.py that's called before step 6?
   - A separate helper script?
   - Part of a general "prepare step input" mechanism in run_llm.py?

3. **Embedder for step 7 matching**: The spec says "any multilingual embedder" for computing cosine similarity between GT `normalized` and LLM `comment`. Preferred choice? (sentence-transformers/paraphrase-multilingual-mpnet-base-v2, OpenAI text-embedding-3-small, etc.)

4. **Model/endpoint configuration**: How should `run_llm.py` determine which model to call? Command-line args? Config file? Environment variable?

5. **Taxonomy axes**: The 12 review axes in `taxonomy.yaml` are marked TODO. Do you want to use a standard taxonomy (e.g., from literature on code review), derive it from a pilot labeling pass, or define it up front based on team focus areas?

6. **Multi-repo grouping**: The final note in PLAN.md says "there is one repository or several within a single team". If you're analyzing multiple repositories:
   - Should `applies_to` and `local_taste` be grouped per repo?
   - Should step 3 merging treat rules from different repos separately?
   - Should the rubric have per-repo sections?

7. **PLAN.ru.md**: This file was deleted in commit 92fee47 ("drop obsolete plan"). The checkpoint summary assumed it existed. Should it be:
   - Recreated as a Russian translation of PLAN.md?
   - Left deleted (obsolete)?
   - Replaced with a different localization strategy?

8. **Holdout split percentage**: Step 0b uses "the last 15% of PRs" for the holdout. With hundreds of threads, this yields 30–50 holdout threads. Is 15% the right split, or would you prefer a different percentage or a fixed count?

---

## Summary

**Deliverables completed:**
1. ✓ Producer/consumer matrix with all artifacts, template variables, and findings (a)–(e)
2. ✓ PLAN.md edits applied (tree fixed, errors.jsonl handling clarified, step 6 enrichment documented, rules_local.jsonl flow explained)
3. ✓ Directory scaffold + placeholder files created (bots.txt, repos.txt, PIPELINE.md, taxonomy.yaml, READMEs, 6 JSON schemas)
4. ✓ Open questions listed for user decision

**What was NOT done (as instructed):**
- Script implementations
- Prompt file bodies (content specified in PLAN.md, files not created)
- No execution or testing
- No changes to PLAN.ru.md (file is absent)

**Next steps:**
- User answers open questions 1–8
- Implement the 7 Python scripts following PLAN.md contracts
- Write the 8 prompt files (content already specified in PLAN.md)
- Populate taxonomy.yaml with the 12 axes
- Commit the scaffold + updated PLAN.md
