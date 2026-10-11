Below is the full pipeline, built on three decisions: the thread is the unit, `reviews.jsonl` is not exported at all, and conservativeness matters more than completeness.

## What the "hundreds of comments" scale changes

This matters more than any other input. After grouping into threads and filtering out bots/noise, you will be left with roughly 100–300 threads. It follows that:

- **No embedding clustering** (UMAP/HDBSCAN) — at this volume it produces artifacts. Rule merging is done by the LLM in a single pass; everything fits in the context.
- **Expect 5–12 rules as output, not 25.** If three rules with solid grounding survive, that is a normal result, not a failure.
- **Reproducibility instead of statistics.** Rule induction is run three times with shuffled batches; a rule survives only if it appeared in at least two runs.
- **The holdout will be thin** (30–50 threads). Validation in step 7 is a sanity check in absolute numbers, not a percentage metric.

Conservativeness thresholds applied between steps (read from `config.yaml` → `thresholds`; the values below are the defaults): a rule requires $\geq 3$ distinct threads, $\geq 2$ distinct PRs, `confidence` $\geq 0.6$, and reproduction in $\geq 2$ of 3 runs. Rules with a single source reviewer are not discarded, but they are flagged and go to manual curation first.

## Project structure

```
review-rules/             # = repository root
├── config.yaml           # human: the ONLY human-supplied input (repos, bots, dates, model, thresholds, …)
├── README.md             # written by the step-0 coding agent: run commands, PAT scopes, request estimate
├── PIPELINE.md           # human: decision log, run log, recorded limitations
├── schemas/              # JSON Schema for each LLM response: p1, p1b, p2…p7 *.schema.json
├── prompts/              # system.txt, p1_normalize.txt p1b_taxonomy.txt p2_induce.txt p3_merge.txt
│                         # p4_coverage.txt p5_score.txt p6_rubric.txt p7_review.txt
├── scripts/
│   ├── extract.py        # step 0
│   ├── prepare.py        # step 0b: threads, features, holdout, LLM views
│   ├── run_llm.py        # single LLM runner: batching, retry, JSON validation (steps 1–7)
│   ├── normalize.py      # step 1: p1 + claim filter (also reused by step 7 for ground truth)
│   ├── infer_taxonomy.py # step 1b: p1b → draft taxonomy for human approval
│   ├── filter_rules.py   # mechanical thresholds after step 2; p3 + cluster stats in step 3
│   ├── coverage.py       # step 4: counts per axis + p4
│   ├── prioritize.py     # step 5: stats, p5, priority formula
│   ├── build_rubric.py   # step 6: assembles the p6 input, calls p6, writes the draft
│   └── evaluate.py       # step 7: diffs, ground truth, p7, matching, report
└── data/
    ├── split.json
    ├── raw/comments.jsonl state.json errors.jsonl
    ├── work/threads.jsonl threads_llm.jsonl claims_raw.jsonl claims.jsonl
    ├── work/taxonomy_draft.json taxonomy.yaml
    ├── work/rules_run{1,2,3}.jsonl singletons_run{1,2,3}.jsonl
    ├── work/rules_filtered.jsonl clusters_raw.jsonl canonical.jsonl
    ├── work/canonical_curated.jsonl coverage_input.json coverage.json
    ├── work/stats.jsonl scores.jsonl priority.jsonl
    ├── work/rubric_input.json rubric_draft.md rubric.md errors.jsonl
    └── eval/threads.jsonl threads_llm.jsonl claims_raw.jsonl gt.jsonl diffs.jsonl
        eval/findings.jsonl labels.jsonl report.json errors.jsonl
```

Conventions:

- **Human input.** Everything a human supplies to the pipeline as a parameter is in `config.yaml`. That covers repositories, bot logins, date range, model/endpoint, embedder, batch sizes, holdout, and thresholds. The only exceptions are secrets: `GITHUB_TOKEN` and the LLM API key are read from environment variables, and `config.yaml` names the variable that holds the API key. Curation results (`data/work/taxonomy.yaml`, `canonical_curated.jsonl`, `rubric.md`, `data/eval/labels.jsonl`) are pipeline artifacts edited by a human, not configuration. Every script takes `--config` (default `./config.yaml`).
- **Errors.** Any script that calls the network or `run_llm.py` writes failures to `errors.jsonl` in the directory of its own output (`data/raw/`, `data/work/`, `data/eval/`). The contract tables below do not repeat this.
- **LLM output lines.** In `run_llm.py` output, each element of the response's top-level array becomes one JSONL line. The line gets `run_id`, `batch_id` and `schema_version`.
- **Language.** Review comments are in Russian. Every free-text field the model writes is in Russian (`llm.output_language` in `config.yaml`). Prompts stay in English. Ids, enum values, code and verbatim quotes are never translated.
- **Scope of data.** All repositories belong to one team with the same reviewers and similar domains. Rules are therefore not grouped per repository; `repo` is kept only to build links.
- **Data is not committed.** Everything under `data/` is in `.gitignore` because it contains reviewers' comments and code; only the `.gitkeep` files that keep the directories are tracked.

Common system prefix for all LLM steps 1–7 (`prompts/system.txt`, prepended by `run_llm.py`):

```text
You are a component of an automated pipeline. Respond only with valid JSON matching the specified schema, without markdown or explanations. Do not invent data that is not present in the input. When uncertain, use the values provided by the schema (null, needs_context, insufficient_data) rather than guesses. If the input contains a field not described in the task, ignore it. Write every free-text value in {{OUTPUT_LANGUAGE}}; never translate ids, enum values, code, or verbatim quotes.
```

`{{OUTPUT_LANGUAGE}}` ← `config.yaml` → `llm.output_language` (default `Russian`); `run_llm.py` substitutes it in every step.

---

## Step 0 — export (prompt for the coding agent)

| Contract | |
|---|---|
| Input | `config.yaml` (`github.repos`, `github.since`, `github.until`), env `GITHUB_TOKEN`, GitHub GraphQL API |
| Output | `data/raw/comments.jsonl`, `data/raw/state.json`, `data/raw/errors.jsonl`; `README.md` (written by the coding agent together with the script) |
| Performer | `scripts/extract.py` |
| Template vars | — |

```text
Write extract.py in Python 3.11+ with httpx: export inline code review comments from GitHub via GraphQL.

INPUT: GITHUB_TOKEN from the environment; --config (default config.yaml): github.repos (list of "owner/name"), github.since / github.until (filter by mergedAt), github.concurrency (default 4); CLI flags --out, --state, --max-prs, --verbose. CLI flags override config values.

SOURCE — GraphQL only, https://api.github.com/graphql. Do not use REST: it does not return isResolved/isOutdated.

Queries:
1. search(query:"repo:O/N is:pr is:merged merged:SINCE..UNTIL", type:ISSUE, first:50, after:$cursor) — enumerate PRs, paginate by cursor.
2. For each PR: number, mergedAt, additions, deletions, changedFiles, author{login}, reviewDecision,
   reviewThreads(first:100, after:$cursor){ nodes{ id isResolved isOutdated path line originalLine startLine diffSide
     comments(first:100){ nodes{ id databaseId body createdAt author{login} authorAssociation diffHunk originalCommit{oid} commit{oid} replyTo{id} } } } }
   reviews(first:100){ nodes{ state submittedAt author{login} } }  — needed ONLY to compute blocked_merge; do not store the review body.
   commits(first:100){ nodes{ commit{ committedDate oid } } }
   Paginate all nested collections if hasNextPage.

OUTPUT: a single file data/raw/comments.jsonl (default of --out; --state defaults to data/raw/state.json), one line = one inline comment. repo = "owner/name", pr_id = PR number (integer), id = GraphQL node id, database_id = databaseId (needed for the discussion_r link in curation):
{"id","database_id","repo","pr_id","pr_merged_at","pr_author","thread_id","path","line_range":{"start","end","side"},
 "diff_hunk","comment_body","author","author_association","created_at","is_reply",
 "thread_resolved","thread_outdated","comment_commit_oid","original_commit_oid",
 "pr_size":{"additions","deletions","changed_files"},
 "pr_changes_requested_count","blocked_merge","blocked_merge_reason","schema_version":"1","run_id"}

blocked_merge = true if, after a review with state=CHANGES_REQUESTED, a commit appeared with committedDate > submittedAt. This is a heuristic; write the text of the condition into blocked_merge_reason. If there is no CHANGES_REQUESTED review — false; if commit data was not retrieved — null.
Any unknown value is null, not false and not "". Bots and noise are NOT filtered at this step.

RELIABILITY: retries with exponential backoff on 403/429/5xx, respect Retry-After, pause ≥60 s when the response text mentions a secondary rate limit; read X-RateLimit-Remaining, pause when remaining <100; state.json with processed pr_id values, a re-run appends without duplicates; an error on one PR → record in errors.jsonl and continue. The token is never logged anywhere.

STRUCTURE: build_pr_list_query() and build_pr_query() as separate functions returning a GraphQL string; dry_run() with --max-prs 3 prints two records without writing to the file. No tests needed. README: run commands, required PAT scopes (classic: repo; fine-grained: Pull requests Read + Contents Read + Metadata Read with an explicit list of repositories), estimate of the number of requests.
```

<details>
<summary>Checking the trial run (--max-prs 5)</summary>

1. `diff_hunk` is non-empty and contains code, not just the `@@` line.
2. `thread_resolved` appears as both `true` and `false`; if it is `null` everywhere, the threads query is built incorrectly.
3. `is_reply=true` exactly for comments that have `replyTo`.
4. `pr_merged_at` is filled in for all records.
5. The file contains records from `*[bot]` — the filter has not leaked into step 0.

</details>

## Step 0b — preparation (prompt for the coding agent)

| Contract | |
|---|---|
| Input | `data/raw/comments.jsonl`, `config.yaml` (`github.bots`, `holdout`) |
| Output | `data/work/threads.jsonl`, `data/work/threads_llm.jsonl`, `data/eval/threads.jsonl`, `data/eval/threads_llm.jsonl`, `data/split.json`; summary on stdout |
| Performer | `scripts/prepare.py` |
| Template vars | — |

```text
Write prepare.py: from data/raw/comments.jsonl build threads, features, and the holdout. No network access is used.

1. build_threads(): group by thread_id, comments ordered by created_at.
   Thread fields: thread_id, repo, pr_id, pr_merged_at, path, line_range, diff_hunk (from the first comment), resolved (= thread_resolved), outdated (= thread_outdated),
   comments:[{id, database_id, author, author_role, is_pr_author, body (= comment_body), created_at, comment_commit_oid, original_commit_oid}],
   where author_role: "bot" if the login ends with "[bot]" or is listed in config.yaml github.bots; "member" if author_association ∈ {MEMBER, OWNER, COLLABORATOR}; otherwise "external".
   is_pr_author = author == pr_author.
   Remove threads where ALL comments are from bots OR the first comment is from a bot. Other bots in the middle of a thread remain with author_role="bot".
   Remove threads whose first comment is shorter than 4 characters after trim.

2. derive_features() at the thread level, all null-safe:
   n_comments, n_participants (excluding bots), has_author_reply, first_reviewer_hash (sha1 of the login, 8 characters — for counting reviewer diversity without revealing names),
   resolved, outdated, code_changed_after = outdated OR any(comment_commit_oid != original_commit_oid) (null if there is nothing to compare),
   resolved_after_fix = resolved AND code_changed_after, resolved_without_fix = resolved AND code_changed_after == false,
   thread_turnaround_hours, days_before_merge, blocked_merge (from the PR), pr_changes_requested_count, pr_size.

3. holdout_split(): sort PRs by pr_merged_at; the last holdout.pr_fraction of PRs (default 0.15; split by PRs, not threads!) → data/eval/threads.jsonl, the rest → data/work/threads.jsonl. If the holdout then has fewer than holdout.min_threads threads (default 30), move the boundary to earlier PRs until it reaches that number or the work set falls to holdout.min_work_fraction of all threads (default 0.7). Write the date boundary, PR and thread counts to data/split.json.

4. make_llm_view(): data/work/threads_llm.jsonl — a copy of work/threads.jsonl from which the following are REMOVED: resolved, outdated, code_changed_after, resolved_after_fix, resolved_without_fix, blocked_merge, pr_changes_requested_count, n_comments, thread_turnaround_hours, days_before_merge, first_reviewer_hash, pr_size, pr_merged_at. What remains: thread_id, repo, pr_id, path, line_range, diff_hunk, comments:[{author, body}], where author is a pseudonym: "bot" for author_role="bot"; otherwise replace author logins with "reviewer_1", "pr_author", "reviewer_2" in order of appearance. Do the same for eval: data/eval/threads_llm.jsonl from eval/threads.jsonl.

5. Print a summary: threads total / after filtering / in work / in eval; null share for each computed field; distribution of n_comments; number of unique first_reviewer_hash values. If there are ≤2 unique reviewers, print the warning "reviewer diversity is low; the rules will reflect the style of 1–2 people".
```

---

## Configuration — `config.yaml`

The single human-supplied file. Every script reads it. Missing keys fall back to the defaults shown; `TODO` values must be filled in before the first run.

| Key | Default | Used by |
|---|---|---|
| `github.repos` | TODO | extract.py |
| `github.since`, `github.until` | TODO (ISO dates) | extract.py |
| `github.concurrency` | 4 | extract.py |
| `github.bots` | `[]` (logins ending in `[bot]` are always bots) | prepare.py |
| `holdout.pr_fraction` / `min_threads` / `min_work_fraction` | 0.15 / 30 / 0.7 | prepare.py |
| `llm.base_url`, `llm.model` | TODO | run_llm.py |
| `llm.api_key_env` | `LLM_API_KEY` (name of the env var, not the key) | run_llm.py |
| `llm.temperature` / `max_retries` / `output_language` | 0 / 3 / `Russian` | run_llm.py |
| `batches.p1_threads` / `p2_claims` / `p7_files` | 25 / 50 / 1 | normalize.py, run_llm.py, evaluate.py |
| `runs.induce_seeds` | `[1, 2, 3]` | run_llm.py (step 2) |
| `thresholds.min_threads` / `min_prs` / `min_runs` / `min_confidence` | 3 / 2 / 2 / 0.6 | filter_rules.py |
| `thresholds.repetitive_ratio` | 0.4 | coverage.py |
| `thresholds.insufficient_null_rate` | 0.5 | prioritize.py |
| `rubric.max_rules` | 15 | build_rubric.py |
| `eval.embedder` | `BAAI/bge-m3` | evaluate.py |
| `eval.line_tolerance` / `similarity_threshold` / `low_confidence` | 3 / 0.75 (tuned on labels) / 0.5 | evaluate.py |

The values of the thresholds are the pipeline's decisions (see intro). They are in the config only so that they are logged with each run in `PIPELINE.md`, not so that they can be loosened after a bad run.

---

## Step 1 — normalizing threads → claims

| Contract | |
|---|---|
| Input | `data/work/threads_llm.jsonl`, `data/work/threads.jsonl`, `config.yaml` (`batches.p1_threads`) |
| Output | `data/work/claims_raw.jsonl` (LLM output, one line per thread), `data/work/claims.jsonl` (filtered, one line per claim) |
| Performer | `scripts/normalize.py` (calls `run_llm.py` with `prompts/p1_normalize.txt`, then filters). The same script with `--split eval` runs on `data/eval/` for step 7. |
| Template vars | `{{BATCH_JSON}}` ← threads_llm.jsonl batches of `batches.p1_threads` threads (default 25); `{{OUTPUT_LANGUAGE}}` |

Batch of 20–30 threads. Input — `threads_llm.jsonl`.

```text
You are given a batch of code review threads. Each thread: thread_id, path, line_range, diff_hunk, comments (ordered; the first is the original remark, the rest are discussion; roles: reviewer_N, pr_author, bot).

TASK: for each thread, extract statements about the code (claims) and classify them.

RULES:
1. Rely only on diff_hunk and the comment text. If it is impossible to understand which code is being discussed — needs_context=true, normalized=null.
2. Usually a thread has one claim. Several — only if the reviewer explicitly lists different problems. pr_author replies are context for understanding, not a source of claims.
3. Ignore bot replies completely.
4. If during the discussion the reviewer withdrew the remark themselves ("ah, got it, that's correct") — mark withdrawn=true, but still extract the claim.
5. Exactly one output record per thread; preserve the input order. If there are no claims — an empty list and a reason.

claim CATEGORIES:
- noise: thanks, "+1", LGTM, emoji, reactions without content
- style_only: formatting, naming, imports, line length — anything a linter catches
- question: a clarifying question without a claim of a defect
- defect: bug, edge case, vulnerability, race condition, leak, broken contract, incorrect logic
- design: structure, API, abstraction, testability, duplication
- docs: documentation, code comments, user-facing error messages
- process: not about the code (split the PR, no tests in CI, another reviewer is needed)

For defect/design/docs fill in:
- normalized: one neutral sentence — a verifiable requirement on the code, phrased as an invariant ("when X is absent, field Y must…"). No addressing the reader and no imperative mood.
- evidence: a verbatim quote from the reviewer's comment (≤200 characters)
- code_anchor: an identifier or line from diff_hunk that the claim refers to, or null
- severity_hint: low | medium | high — based only on the text and the hunk
- language: the language of the code in diff_hunk, if determinable, otherwise null

OUTPUT:
{"threads":[{"thread_id":"...","claims":[{"claim_id":"<thread_id>#1","category":"...","normalized":"...","evidence":"...","code_anchor":"...","severity_hint":"...","language":"...","needs_context":false,"withdrawn":false}],"reason":null}]}

Batch:
{{BATCH_JSON}}
```

After the LLM responds, `normalize.py` flattens the threads into claims. It keeps only claims where `category ∈ {defect, design, docs}`, `needs_context=false`, `withdrawn=false`, and writes them to `data/work/claims.jsonl`, one claim per line. For each surviving claim, the code appends `thread_id`, `pr_id`, `repo` and `first_reviewer_hash` from `data/work/threads.jsonl`. The model in step 2 does **not** see `pr_id`, `repo` or `first_reviewer_hash`.

## Step 1b — inferring the taxonomy (then human approval)

| Contract | |
|---|---|
| Input | `data/work/claims.jsonl` (fields `claim_id, category, normalized, language` only) |
| Output | `data/work/taxonomy_draft.json` (LLM output); `data/work/taxonomy.yaml` (approved by a human) |
| Performer | `scripts/infer_taxonomy.py` (calls `run_llm.py` with `prompts/p1b_taxonomy.txt`, writes the draft and an editable `taxonomy.yaml` from it), then human |
| Template vars | `{{CLAIMS_JSON}}` ← claims.jsonl, all claims in one call; `{{OUTPUT_LANGUAGE}}` |

The axes are not fixed in advance; they are derived from the work claims once a run has produced them. One call covers everything (with hundreds of threads this is 100–300 short claims). Holdout claims are never shown.

```text
You are given normalized code review claims (claim_id, category, normalized, language). You will define the axes of a taxonomy that a later step uses to label code review rules.

TASK: propose review axes that partition these claims.

RULES:
1. Propose 8–15 axes. An axis is a kind of problem in the code (e.g. a class of defect or of design issue), not a technology, not a file, and not a severity.
2. Axes must not overlap: a claim must fit one axis better than any other. If two candidate axes share most of their claims, merge them.
3. Each axis that is supported by data lists 2–5 example claim_ids that fit it, and the count of claims that fit it best.
4. Claims that fit no axis go to unassigned_claim_ids. Do not create an axis for a single claim.
5. Additionally, list up to 5 axes that are commonly checked in code review but have NO supporting claims here (n_claims = 0, examples empty, evidenced = false). They let the coverage step show blind spots. Do not invent supporting claims for them.
6. id: snake_case English; title and description: in the output language.

OUTPUT:
{"axes":[{"id":"...","title":"...","description":"one sentence: what falls on this axis and what does not","evidenced":true,"n_claims":0,"example_claim_ids":["..."]}],"unassigned_claim_ids":["..."]}

Claims:
{{CLAIMS_JSON}}
```

`infer_taxonomy.py` checks that all `example_claim_ids` exist in `claims.jsonl`, writes the response to `taxonomy_draft.json`, and writes `data/work/taxonomy.yaml` as a list of `{id, title, description, evidenced}`. **Human approval (mandatory):** rename, merge, split or delete axes in `data/work/taxonomy.yaml`, then record the final list in `PIPELINE.md`. Steps 2 and 4 read only the approved `taxonomy.yaml`. Once approved, keep the taxonomy fixed for later runs, so that coverage can be compared over time. Re-infer it only when the repository set changes.

## Step 2 — rule induction (3 runs)

| Contract | |
|---|---|
| Input | `data/work/claims.jsonl`, `data/work/taxonomy.yaml` (approved), `config.yaml` (`batches.p2_claims`, `runs.induce_seeds`) |
| Output | `data/work/rules_run{1,2,3}.jsonl`, `data/work/singletons_run{1,2,3}.jsonl` |
| Performer | `run_llm.py` with `prompts/p2_induce.txt` (called 3× with different shuffle seeds) |
| Template vars | `{{BATCH_JSON}}` ← claims.jsonl batches of `batches.p2_claims` claims (default 50); `{{TAXONOMY_AXES}}` ← axis ids from data/work/taxonomy.yaml; `{{OUTPUT_LANGUAGE}}` |

Batch of 40–60 claims. Three runs, one per seed in `runs.induce_seeds`; each run shuffles the claims before batching and writes its own `rules_run{k}.jsonl` with a `run_id`. With hundreds of threads, one run is 2–5 batches. `run_llm.py` passes only the fields listed in the prompt (`claim_id, thread_id, normalized, evidence, code_anchor, category, language`); it projects them with its `--fields` option.

```text
You are given a batch of claims — normalized code review remarks with quotes. Fields: claim_id, thread_id, normalized, evidence, code_anchor, category, language.

TASK: derive atomic code review rules confirmed by several independent remarks.

HARD RULES:
1. One piece of evidence = one thread_id. Several claims from the same thread count as one piece of evidence. A rule requires at least 3 DISTINCT thread_ids. Otherwise — put it in singletons.
2. One rule = one check. Combining different requirements with "and"/"or" is forbidden.
3. A rule must be falsifiable: there must exist code that violates it. "Code must be readable" is forbidden.
4. Do not generalize beyond the evidence. If all the remarks are about a missing None check after an ORM query, the rule is about that, not "validate all inputs".
5. If the wording boils down to a team preference ("we don't do it that way", "that's our convention") — scope="local_taste".
6. If the evidence comes from different languages/stacks and the rule holds for all of them — scope="universal". If it holds only for one stack — specify it in applies_to.
7. It is better not to output a rule than to output a doubtful one. If you are unsure whether the evidence supports the same thing — do not merge.

FOR EACH RULE:
- id: kebab-case, meaningful
- statement: one sentence, verifiable on a diff
- axis: exactly one of {{TAXONOMY_AXES}}
- scope: universal | local_taste
- applies_to: language/framework or null
- evidence_thread_ids: list of thread_ids (≥3)
- best_evidence_claim_id: the one claim that best illustrates the rule
- counterexample: a short code fragment that the rule rejects, derived from the evidence
- confidence: 0..1 — confidence that the rule generalizes beyond the batch. Lower it if the evidence looks like the same problem in the same place.

OUTPUT:
{"rules":[{"id":"...","statement":"...","axis":"...","scope":"...","applies_to":null,"evidence_thread_ids":["..."],"best_evidence_claim_id":"...","counterexample":"...","confidence":0.0}],
 "singletons":[{"thread_ids":["..."],"note":"briefly, what it is about"}]}

Claims:
{{BATCH_JSON}}
```

The LLM outputs `{"rules":[...], "singletons":[...]}`. `run_llm.py` writes each element of the `rules` array to `data/work/rules_run{k}.jsonl` (one rule per line), and each element of the `singletons` array to `data/work/singletons_run{k}.jsonl`.

<details>
<summary>Mechanical filter between steps 2 and 3 (filter_rules.py)</summary>

| Contract | |
|---|---|
| Input | `data/work/rules_run{1,2,3}.jsonl`, `data/work/threads.jsonl`, `data/work/claims.jsonl`, `config.yaml` (`thresholds`) |
| Output | `data/work/rules_filtered.jsonl` |
| Performer | `scripts/filter_rules.py` |
| Template vars | — |

The code computes from `evidence_thread_ids` via `threads.jsonl`: `n_threads`, `n_prs`, `n_reviewers` (distinct `first_reviewer_hash`), `n_repos`. It keeps a rule if:

- `n_threads` $\geq 3$ and `n_prs` $\geq 2$;
- `confidence` $\geq 0.6$;
- all `evidence_thread_ids` actually exist in `claims.jsonl` (models tend to make up ids).

The same thresholds apply to rules with `scope=local_taste`. They pass the filter and steps 3–5 like any other rule and keep their `scope`; step 6 puts them in a separate section. Rules with `n_reviewers = 1` get the flag `single_reviewer=true`.

Output: a combined `rules_filtered.jsonl` from all three runs, with each rule's `run_id` preserved.

</details>

## Step 3 — merging runs and deduplication

| Contract | |
|---|---|
| Input | `data/work/rules_filtered.jsonl`, `data/work/threads.jsonl`, `config.yaml` (`thresholds.min_runs`) |
| Output | `data/work/clusters_raw.jsonl` (LLM output), `data/work/canonical.jsonl` (after filter) |
| Performer | `scripts/filter_rules.py --stage merge` (calls `run_llm.py` with `prompts/p3_merge.txt`, then filters) |
| Template vars | `{{RULES_JSON}}` ← rules_filtered.jsonl (all rules in one batch); `{{OUTPUT_LANGUAGE}}` |

One call for everything (with hundreds of threads this is 20–60 rules). Input — `id, run_id, statement, axis, scope, applies_to, evidence_thread_ids, counterexample`.

```text
You are given code review rules derived by three independent runs (run_id) from the same data. The same requirement may be phrased differently in different runs.

TASK: group the rules into canonical clusters.

RULES:
1. Merge rules only if THE SAME code change violates both. A shared topic ("error handling") is not grounds for merging. Timeouts and retries are different rules.
2. Do not merge rules about different libraries or stacks if applies_to differs.
3. Overlap of evidence_thread_ids is a strong signal in favor of merging; an empty overlap is a signal against, but not a prohibition.
4. A rule without a pair is a single-element cluster. Do not force it into another cluster.
5. The canonical statement must cover all members of the cluster and must not be broader than them. If that is impossible — do not merge.
6. If members of a cluster contradict each other — mark conflict=true and describe the contradiction.

OUTPUT:
{"clusters":[{"cluster_id":"...","statement":"...","axis":"...","scope":"...","applies_to":null,"member_rule_ids":["..."],"counterexample":"...","conflict":false,"conflict_note":null,"merge_doubt":null}]}

merge_doubt — one line if the merge is questionable; otherwise null.

Rules:
{{RULES_JSON}}
```

After step 3, `filter_rules.py` appends to each cluster: `runs_covered` (how many of the 3 runs produced a member), the combined `evidence_thread_ids` from all members, `n_threads`, `n_prs`, `n_reviewers`, `single_reviewer` (computed from the combined thread list via `threads.jsonl`), and `best_evidence_claim_id` (taken from the member rule with the highest `confidence`). **Clusters with `runs_covered` $< 2$ are discarded.** What remains is written to `data/work/canonical.jsonl`.

<details>
<summary>Manual curation after step 3 (mandatory)</summary>

With hundreds of threads this takes 15–30 minutes. For each cluster:

1. Open 2–3 threads from `evidence_thread_ids` via the link `https://github.com/{repo}/pull/{pr_id}#discussion_r{database_id}` (`repo`, `pr_id` and the first comment's `database_id` come from `threads.jsonl`) — can the rule really be derived from these comments?
2. Can the rule be turned into a linter/test/CI check? → fill in `automate_with` (the tool or check; otherwise null) and exclude it from the rubric.
3. `single_reviewer=true` → is this a team norm or one person's taste? In the latter case — `scope=local_taste`.
4. `conflict=true` or `merge_doubt` is filled in → split or delete.
5. The rule describes a stack that is no longer in the repository → delete.

The result is `data/work/canonical_curated.jsonl`: the same records with `automate_with` and `curation_note` fields added; deleted clusters are removed and split ones get new `cluster_id`s. A rule you hesitate over for more than a minute is deleted — this is "conservativeness" in action.

</details>

## Step 4 — coverage by axis

| Contract | |
|---|---|
| Input | `data/work/canonical_curated.jsonl`, `data/work/threads.jsonl`, `data/work/taxonomy.yaml` (approved), `config.yaml` (`thresholds.repetitive_ratio`) |
| Output | `data/work/coverage_input.json` (computed stats), `data/work/coverage.json` (LLM output) |
| Performer | `scripts/coverage.py` (computes stats, then calls `run_llm.py` with `prompts/p4_coverage.txt`) |
| Template vars | `{{COVERAGE_JSON}}` ← coverage_input.json (it already contains the axes list with `evidenced` from taxonomy.yaml); `{{OUTPUT_LANGUAGE}}` |

Using `canonical_curated.jsonl`, the code counts the number of clusters and threads for each axis from `data/work/taxonomy.yaml`, as well as `repetitive = n_prs / n_threads < 0.4 OR n_reviewers == 1`. The result is written to `coverage_input.json` as `{"axes":[{axis, evidenced, n_clusters, n_threads}], "clusters":[{cluster_id, statement, axis, counterexample, n_threads, n_prs, n_reviewers, repetitive}]}` and passed to the LLM for interpretation.

```text
You are given canonical code review rules with precomputed statistics and a list of taxonomy axes with the number of rules for each.

TASK:
1. For each rule, check the axis field. If the axis is clearly wrong — fill in axis_suggested; do not change the original.
2. For axes with n_clusters = 0, write a blind_spot_note (one line): what can and cannot be concluded from the absence of data. Do not claim that the team "forgot" or "doesn't know how to do" something.
3. For rules with repetitive=true, write a repetitive_note: judging by the statement and counterexample, does it look like one unresolved problem in one place or a systemic error?

Do not recompute or dispute the statistics.

OUTPUT:
{"axes":[{"axis":"...","blind_spot_note":null}],"clusters":[{"cluster_id":"...","axis_suggested":null,"repetitive_note":null}]}

Data:
{{COVERAGE_JSON}}
```

The output of this step is for humans (`data/work/coverage.json`); it does not go into the rubric. Its value with small data is to honestly show that the rules cover only a few of the approved axes, and that this is a limitation, not a property of the team's code. Because the axes were inferred from the same claims (step 1b), empty axes mostly come from the `evidenced=false` ones and from claims that did not survive steps 2–3. Read blind spots with that in mind.

## Step 5 — prioritization

| Contract | |
|---|---|
| Input | `data/work/canonical_curated.jsonl`, `data/work/threads.jsonl`, `config.yaml` (`thresholds.insufficient_null_rate`) |
| Output | `data/work/stats.jsonl` (computed stats), `data/work/scores.jsonl` (LLM output), `data/work/priority.jsonl` (final) |
| Performer | `scripts/prioritize.py` (aggregates stats, calls `run_llm.py` with `prompts/p5_score.txt`, then computes priority formula) |
| Template vars | `{{CLUSTERS_WITH_STATS_JSON}}` ← stats.jsonl; `{{OUTPUT_LANGUAGE}}` |

The code aggregates from `evidence_thread_ids` in `threads.jsonl`: `resolved_after_fix_rate`, `resolved_without_fix_rate`, `code_changed_rate`, `author_reply_rate`, `mean_n_participants`, `blocked_merge_rate`, `null_rate` (the share of null values across these fields). The result is written to `stats.jsonl`, one line per cluster: `cluster_id, statement, n_threads, n_prs, n_reviewers` plus the rates. The model provides $s$:

```text
You are given code review rules with outcome statistics for their evidence. stats fields (shares in 0..1, null — no data): n_threads, n_prs, n_reviewers, resolved_after_fix_rate, resolved_without_fix_rate, code_changed_rate, author_reply_rate, mean_n_participants, blocked_merge_rate, null_rate.

TASK: estimate s ∈ [0,1] — the cost of missing the problem the rule catches.

SCALE:
- 0.0–0.2: the author may ignore it without harm
- 0.3–0.5: quality without functional risk
- 0.6–0.8: a prevented bug that would not have reached production
- 0.9–1.0: incident, vulnerability, data loss

INTERPRETING THE STATISTICS (follow this, not general considerations about the importance of the topic):
- resolved_after_fix_rate — the main acceptance signal: the author agreed and rewrote the code.
- high resolved_without_fix_rate — remarks were closed without a fix: rejected or insignificant. Lowers s.
- high author_reply_rate and mean_n_participants > 2 — a contested remark, not necessarily an important one.
- blocked_merge_rate — a heuristic; weight no more than half of resolved_after_fix_rate.
- null_rate > 0.5 → insufficient_data=true, s=null.
- Take the severity implied by the statement wording into account, but outcome statistics matter more.

OUTPUT:
{"scores":[{"cluster_id":"...","s":0.0,"insufficient_data":false,"reasoning":"one line"}]}

Data:
{{CLUSTERS_WITH_STATS_JSON}}
```

The code writes the LLM response to `data/work/scores.jsonl`, then computes the priority. Logarithms over threads are needed so that one cluster with 40 identical remarks does not overwhelm the rest; the number of PRs enters linearly as a measure of diversity:

$$priority = \log(1 + n_{threads}) \cdot n_{prs} \cdot a \cdot s, \qquad a = \begin{cases} resolved\_after\_fix\_rate & \text{if not null} \\ 0.5 & \text{otherwise} \end{cases}$$

`insufficient_data=true` → `priority = null`; the rule is ranked last but not deleted. The final output is written to `data/work/priority.jsonl`, one line per cluster: `cluster_id, priority, s, insufficient_data, a`.

## Step 6 — rubric

| Contract | |
|---|---|
| Input | `data/work/priority.jsonl`, `data/work/canonical_curated.jsonl`, `data/work/claims.jsonl`, `data/work/threads.jsonl` (to retrieve best_evidence), `config.yaml` (`rubric.max_rules`) |
| Output | `data/work/rubric_input.json` (assembled input), `data/work/rubric_draft.md` (LLM output), `data/work/rubric.md` (after manual curation) |
| Performer | `scripts/build_rubric.py` (assembles the input, calls `run_llm.py` with `prompts/p6_rubric.txt` in raw-text mode), then human |
| Template vars | `{{PRIORITY_JSON}}` ← rubric_input.json; `{{MAX_RULES}}` ← `rubric.max_rules` (default 15); `{{OUTPUT_LANGUAGE}}` |

`build_rubric.py` joins `priority.jsonl` with `canonical_curated.jsonl` by `cluster_id`. From each curated record it takes `statement, axis, scope, applies_to, counterexample, automate_with` and `best_evidence_claim_id`. It looks up that claim's `evidence` (the quote) in `claims.jsonl` and, via the claim's `thread_id`, the `diff_hunk` in `threads.jsonl`. It writes `rubric_input.json`: a list of `{cluster_id, statement, axis, scope, applies_to, priority, counterexample, best_evidence:{quote, diff_hunk}, automate_with}` sorted by `priority` descending, with nulls last. Rules with `scope=local_taste` (from the model in step 2 or set during curation) are in this same list; the prompt places them in their own section.

Step 6 is the only step whose answer is Markdown rather than JSON. `run_llm.py --raw` skips schema validation and the JSON part of the system prefix, and writes the text as is.

```text
Build a rubric for an LLM reviewer from the canonical rules with priorities. Input: cluster_id, statement, axis, scope, applies_to, priority, counterexample, best_evidence (quote + diff_hunk), automate_with.

CONSTRAINTS:
1. No more than {{MAX_RULES}} rules. Do not include rules with automate_with — move them to the "Automate" section.
2. For each rule: wording, axis, applies_to (if any), a violation example (based on counterexample/best_evidence; do not invent a new stack), a NON-violation example — similar code that the rule must NOT flag. If a convincing non-violation example cannot be constructed, the rule is too vague; put it in "Deferred" with that reason.
3. "When to stay silent" section: generated code, lock files, migrations, pure renames, changes only to test fixtures, and anything that clearly follows from the rules' applicability (e.g., an ORM rule does not apply to files without ORM).
4. Rules with scope=local_taste — a separate section "Conventions of this repository", with a note that they do not apply to external code.
5. Do not add rules that are not in the input. Do not add "well-known" checks.

FORMAT — Markdown:
## Role
You are a pull request reviewer. You leave remarks only according to the rules below, referencing the rule number. If in doubt — stay silent.
## Rules
### R1. <title>
- Check: ...
- Applies to: ...
- Violation: ```<language> ...```
- Not a violation: ```<language> ...```
## When to stay silent
## Conventions of this repository
## Automate (not for the reviewer)
## Deferred

Data:
{{PRIORITY_JSON}}
```

The LLM output goes to `data/work/rubric_draft.md`. After step 6 — a second, short manual curation: read the rubric as instructions for a new employee. Any item that is unclear without the pipeline's context is rewritten or deleted. Save the result as `data/work/rubric.md`.

## Step 7 — validation on the holdout

| Contract | |
|---|---|
| Input | `data/eval/threads.jsonl`, `data/eval/threads_llm.jsonl`, `data/work/rubric.md`, `data/eval/labels.jsonl` (optional, human), `config.yaml` (`eval`, `batches.p7_files`), env `GITHUB_TOKEN`, GitHub REST API (diffs) |
| Output | `data/eval/claims_raw.jsonl` (GT from p1), `data/eval/gt.jsonl` (filtered GT), `data/eval/diffs.jsonl` (fetched diffs), `data/eval/findings.jsonl` (p7 output, both variants), `data/eval/report.json` |
| Performer | `scripts/evaluate.py` (fetches diffs, runs `normalize.py --split eval` for ground truth, calls `run_llm.py` with `prompts/p7_review.txt` twice, then matches) |
| Template vars | `{{RUBRIC_MD}}` ← rubric.md or the baseline string; `{{DIFF}}` ← one file's patch from diffs.jsonl; `{{OUTPUT_LANGUAGE}}` |

Reviewer prompt:

```text
You are an LLM reviewer. You are given a diff (one file or part of one) and a rubric. You do not know and do not try to guess what humans wrote.

TASK: produce remarks strictly according to the rubric.

RULES:
1. Each remark references a rule number (rule_id = null when the rubric has no numbered rules) and specific lines of the diff (line numbers from the RIGHT side).
2. Priority — minimum false positives. If in doubt — do not write.
3. Problems outside the rubric go into outside_rubric; they do not count as answers.
4. An empty findings list is a normal result.

OUTPUT:
{"findings":[{"rule_id":"R1","path":"...","line_start":0,"line_end":0,"comment":"...","confidence":0.0}],"outside_rubric":[{"path":"...","line_start":0,"note":"..."}]}

Rubric:
{{RUBRIC_MD}}

Diff:
{{DIFF}}
```

Baseline — the same prompt, but instead of the rubric a single line: "Find defects, edge cases, and design problems." Without the pair of numbers "with rubric / without", the result cannot be interpreted. `evaluate.py` writes both runs to `findings.jsonl`, adding `variant` (`rubric` | `baseline`), `repo` and `pr_id` to every finding.

<details>
<summary>evaluate.py — ground truth and matching</summary>

| Contract | |
|---|---|
| Performer | `scripts/evaluate.py` (all bullets below) |

- **Input diff**: for each PR from `eval/threads.jsonl`, the diff is fetched via `GET /repos/{o}/{r}/pulls/{n}/files` (the `patch` field) and written to `diffs.jsonl` as one line per file: `repo, pr_id, path, patch`. All files of the PR are submitted, not only the files with threads — otherwise precision is inflated. Lock files and generated files listed in the rubric's "When to stay silent" section are submitted too; the reviewer is expected to stay silent on them.
- **Ground truth**: holdout threads in `eval/threads_llm.jsonl` are run through prompt 1 by `normalize.py --split eval` (it doesn't learn anything) with the same filter as step 1 (`category ∈ {defect, design, docs}`, `needs_context=false`, `withdrawn=false`); raw output → `eval/claims_raw.jsonl`. `evaluate.py` then adds `repo, pr_id, path, line_range` from `eval/threads.jsonl` → `gt.jsonl`.
- **Match**: same `repo`, `pr_id` and `path`, line ranges overlap with a tolerance of `eval.line_tolerance` ($\pm 3$), and the cosine similarity of `normalized` ↔ `finding.comment` is above `eval.similarity_threshold`. The embedder is `eval.embedder` (default `BAAI/bge-m3`: local, multilingual, strong on Russian; GT and findings are both in Russian). Tune the threshold on 20–30 manually labeled pairs saved in `labels.jsonl` (`gt_claim_id, finding_id, variant, match: true|false`); labels, when present, override the embedder. With small data you can even label the matches entirely by hand — there will be a few dozen of them.
- **Report in absolute numbers** (written to `report.json`): total GT / found with rubric / found by baseline / false positives with rubric / false positives baseline / findings with `confidence < 0.5`. Breakdown by `rule_id`: a rule with zero matches and non-zero false positives is a candidate for removal.
- **Success criterion**: the rubric finds more GT than the baseline with no more false positives. If recall grows only due to `confidence < 0.5`, that is noise.

</details>

---

## Run order and tooling

`run_llm.py` — a single runner for steps 1–7: reads the prompt from `prompts/`, prepends `prompts/system.txt`, substitutes template variables ({{VAR_NAME}}), calls the model at `llm.base_url` / `llm.model` with `temperature=llm.temperature` (0) and the key from the env var named in `llm.api_key_env`, validates the response against `schemas/<prompt>.schema.json`, on a parse error retries the request with the error text (up to `llm.max_retries`, 3), writes `run_id`, `batch_id`, and `schema_version` into every JSONL line. Batches do not overlap; if a batch fails, the others continue, and failed batch records go to `errors.jsonl`. `--raw` (step 6 only) disables JSON handling.

Run order:

```
extract.py → prepare.py → normalize.py: [p1] + filter claims
→ infer_taxonomy.py: [p1b] → APPROVE TAXONOMY
→ run_llm.py: [p2 ×3, shuffle] → filter_rules.py: thresholds
→ filter_rules.py --stage merge: [p3] + drop runs_covered<2 → CURATION
→ coverage.py: count + [p4]
→ prioritize.py: stats + [p5] + formula
→ build_rubric.py: assemble + [p6] → CURATION
→ evaluate.py: diffs, normalize.py --split eval [p1 on holdout], [p7 ×2 (rubric / baseline)], match + report
```

The approved `data/work/taxonomy.yaml` is substituted into prompt 2 via `{{TAXONOMY_AXES}}` and into prompt 4 as part of `{{COVERAGE_JSON}}`. The user maintains `PIPELINE.md` manually as a decision log and run log, recording: the thread as the unit; `reviews.jsonl` is not exported; conservativeness thresholds ($\geq 3$ threads, $\geq 2$ PRs, $\geq 2$ runs, confidence $\geq 0.6$); the approved taxonomy; a copy of `config.yaml` (without secrets) for each run; date ranges; observed limitations (reviewer diversity, axes coverage).

<details>
<summary>What to expect at this volume and when to stop</summary>

- If ≤ 2 rules remain after step 3 — do not loosen the thresholds. Either widen the date window / list of repositories, or honestly record that reviews in the team are heterogeneous and rules cannot be derived from them. That is also a result.
- If `prepare.py` showed ≤ 2 unique reviewers — whatever comes out describes the style of those people. The rubric is usable, but this must be recorded as a limitation in `PIPELINE.md`.
- A holdout of 30–50 threads will yield 10–20 GT remarks. A difference of 2–3 findings between the rubric and the baseline is not proof. Proof is repeating step 7 a quarter later on new PRs.
- Prompt 1 is the only step where it is worth spot-checking the labeling by hand (20 threads). An error there propagates downstream, and the check costs ten minutes.

</details>

Remaining assumption worth verifying: the team has more than two active reviewers (otherwise see the warning in `prepare.py`).
