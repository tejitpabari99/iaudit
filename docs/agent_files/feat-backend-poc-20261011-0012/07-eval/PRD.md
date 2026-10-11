---
status: draft
summary: ADHD fixtures (faithful + 8 single-fault variants as contract requests) with gold-label JSONL, a plain-Python 3-mode harness (Gold/Linked/End-to-end) with per-bucket P/R/F1, turn-level evidence recall, latency and cost, replay for CI, a leakage rule, and a no-data CDR cleanup spec.
date: 2026-10-11
---
# PRD: 07 Evaluation data + harness

Repo/branch: iaudit / `feat/backend-poc` · Depends on: **07a Fixtures (Core): 01-scaffold-api only** (request validation); the 07a smoke script needs 06 for end-to-end runs (or the API with `IAUDIT_PIPELINE=full`). **07b Harness (Optional): 01, 02 (Gold mode, scoring); 03-05 (Linked); 06 (End-to-end)** · Owns: `backend/eval/` (data, harness, tests, CDR spec); adds only an `eval` path entry to SP01's pytest/ruff config

## TL;DR
Measurable quality per bucket, never one overall score. A deterministic builder turns one spec (approved 26-turn ADHD transcript, 7 faithful sentences, 8 injections) into 9 contract `AuditRequest` files plus gold JSONL. A script runs the checker with gold inputs, with the linker, or the full pipeline, scores every run against gold, and records LLM calls so CI replays them offline. POC acceptance = faithful returns no concerns, each variant returns its expected verdict and concern. Splits, the P0 precision gate and P1 suppression stay deferred.

## Part map
This PRD has two clearly separated parts. Hackathon build: **07a is Core** and ships on the critical path. **07b is Optional (gated: start only after ALL core work in every sub-project is finished).** No core item depends on an optional one.
- **07a Fixtures (Core)**: R1 (data layout only), R2, R3 (variants + expected verdict/concern), R5a (simple expected format), R14 (smoke script).
- **07b Harness (Optional)**: R1 (harness files), R4, R5 (full gold JSONL with slots), R6-R13.

## Problem
The brainstorm fixes the evaluation design (3 modes, per-bucket P/R/F1, evidence-span recall) but nothing exists to run it. Without gold labels and a scorer, "the pipeline works" cannot be told apart from "the verdicts look plausible", and a weak linker hides behind a strong checker (AnchorSIPS: endorsement ~1.0, evidence-link F1 <= 0.29). The mock set is tiny, so the harness must also be honest about sample size.

## Goals / Non-Goals
**Goals**
- Fixtures and gold labels for the approved ADHD example (copied verbatim), reproducible from one spec.
- Harness: 3 modes, per-bucket metrics, evidence-span recall, latency, token cost; live is opt-in; recorded replay for CI; a per-fixture POC gate.
- Rules that keep scored data out of prompts (leakage) and keep the real CDR case test-only.
- CDR cleanup script spec and bucket-1 baseline hook (no heavy deps).

**Non-Goals**
- Generating the other ~39 interviews, the 60/20/20 split, P0 precision >= 0.8 as a gate, P1 suppression (<0.6) logic: DEFERRED until data exists.
- Inventing the CDR turns or its rationale. Running MiniCheck/GPT-4o-mini. Phoenix or any dashboard. Any single aggregate score.

## Scope tiers
**Core (07a)**
- ADHD 26-turn transcript copied verbatim (R2), sha256 pin test.
- Faithful + 8 variant contract requests (R2/R3), built by `build_adhd.py` (committed, schema-valid via 01 `validate_request`).
- Expected verdict/concern per variant in simple JSON (R5a).
- Smoke script `backend/eval/smoke.py` (R14): runs the 9 fixtures through the API or pipeline, prints actual vs expected, pass/fail per fixture, exit 1 on any failure.
- Acceptance 1 (core part), 2, 5 (via smoke).

**Optional (07b, gated: after all core work in every SP)**
- 3 eval modes (gold/linked/e2e) and adapters, per-bucket P/R/F1, evidence-span recall, latency/cost report, `compare.py`.
- Full gold JSONL with `slots`, evidence turns/roles, offsets (R5).
- Replay recording manifest, `--live`/`--record`, `--gate poc` (R9).
- Oracle self-test and scorer tests (R10), prompt leakage test (R11).
- V4-noroles variant, `adhd-multi` slot (R4), CDR cleanup spec (R12), baseline hook (R13).
- Acceptance 3, 4, 6, 7, 8 and the non-smoke parts of 1 and 5.

## Requirements

### 07a Fixtures (Core)

**R1 Location and layout.** `backend/eval/`, outside the shipped package `iaudit_backend`, run from `backend/` as `uv run python -m eval.run`.
```
backend/eval/
  data/adhd/spec.json            # transcript, S1..S7, injections, claim slots, gold per sentence
  data/adhd/requests/{faithful,v1..v8,multi}.json   # generated AuditRequests (multi: see R4)
  data/adhd/expected/<fixture>.json                 # [Core, R5a] simple expected verdict/concern
  data/adhd/gold/<fixture>.jsonl                    # [OPTIONAL, 07b R5] generated
  build_adhd.py                  # [Core] deterministic spec -> requests + expected (+ gold in 07b); test asserts committed outputs == regenerated
  smoke.py                       # [Core, R14] 9 fixtures -> actual vs expected, pass/fail
  run.py score.py match.py compare.py adapters.py baselines.py prices.json   # [OPTIONAL, 07b]
  recorded/                      # [OPTIONAL] replay fixtures (synthetic data only)
  private/                       # [OPTIONAL] gitignored: CDR inputs, gold, recordings, runs
  tests/                         # default pytest, no network (core: data + pin tests; rest OPTIONAL)
```
The 07b harness files above, `gold/`, `recorded/` and `private/` are Optional; core ships only `data/adhd/{spec.json,requests,expected}`, `build_adhd.py`, `smoke.py` and their tests. Core is run as `uv run python -m eval.smoke`.
Core `spec.json` holds transcript, S1..S7, injections and the expected block per fixture (07b adds claim slots and evidence). Why not `docs/examples/`: that folder is for public-API usage examples (AGENTS.md); eval data is test infrastructure imported by code. SP08 may link or copy the faithful request there [DEFERRED].

**R2 ADHD fixtures.** 26 turns copied verbatim from the brainstorm table (speakers `clinician1`/interviewer, `alex1`/respondent, `sam1`/informant). Turn ids `t0..t25`; array index equals the table `#`. A test pins turn count, role/speaker per index and a sha256 of the canonical transcript (computed once at authoring from the brainstorm table; any later change must be a deliberate, reviewed edit). Each request: `schema_version` 1.0.0, `request_id` = fixture id (`adhd-faithful`, `adhd-v1`...), one section `{id: "s1", label: "Inattention (DSM-5 A1)", score: "5/9", text}`. Must validate with SP01 `validate_request`. Section text = sentences joined by one space.

Faithful sentences (verbatim from the brainstorm paragraph):

| S | Sentence |
|---|---|
| S1 | Alex reports losing attention in most meetings, about three to four times a week. |
| S2 | He reports misplacing his phone daily but has a system for keys. |
| S3 | He describes some difficulty finishing household projects, although the current unfinished project is partly because they ran out of paint. |
| S4 | His partner reports he forgets agreed errands about twice a month; Alex estimates once a month. |
| S5 | Childhood report cards repeatedly noted daydreaming and unfinished work. |
| S6 | He received a review comment about seeming disengaged in meetings. |
| S7 | He denies careless mistakes at work. |

**R3 Variants** (one edit each to the faithful text; every other sentence unchanged and keeps its faithful gold). Injected wording is the brainstorm column, completed to a sentence; V1 only adds a subject.

| V | Op | Target (faithful sentence) | Injected sentence | Bucket | Expected verdict + concern | Gold evidence turns (role) |
|---|---|---|---|---|---|---|
| 1 | replace | S1 | Alex loses attention in every meeting, several times a day. | 7 P0 | partially_supported + frequency_severity_inflation; explanation contains "inflates" | 3 (supporting) |
| 2 | replace | S7 | He makes frequent careless mistakes in reports. | 2/5 P0 | contradicted + negation_reversal | 5 (contradicting) |
| 3 | append at end | none | He frequently forgets appointments. | 3 P0 | contradicted + wrong_attribution | 21 (contradicting) |
| 4 | append at end | none | He is easily distracted by his surroundings. | 9 P0 | insufficient_evidence + question_as_answer | 12 (qualifying), 13 [OPEN O-2] |
| 5 | append at end | none | He currently forgets work assignments weekly. | 4 stretch | insufficient_evidence + timeframe_mismatch | 17 (qualifying) |
| 6 | replace | S4 | Both Alex and his partner agree he forgets errands about twice a month. | 11 P1 | partially_supported + conflicting_accounts_flattened | 19 (supporting), 23 (contradicting or qualifying) |
| 7 | replace | S3 | The unfinished bathroom shows his difficulty completing tasks. | 8 P1 | partially_supported + context_stripping | 11 (qualifying or supporting) |
| 8 | replace | S6 | His distractibility caused a negative performance review. | 12 P1 | partially_supported + unsupported_causal_link | 25 (qualifying or supporting) |

**[OPTIONAL]** V4-noroles (per 05): V4 with `role` stripped from all turns expects `insufficient_evidence` + `unsupported_claim` + warning `ambiguous_speaker` (no `question_as_answer`; 05's missing-role rule skips A9). Extra check, not gated at POC.


V4-noroles (below) is **[OPTIONAL, 07b]**.

### 07b Harness (Optional, gated)
Everything in this part (R4-R13) is Optional.

**R4 [OPTIONAL] Multi-fault slot.** Request id `adhd-multi`, same builder, `injections: [...]` list in `spec.json`. Content is [OPEN] O-5: no file is generated, no gold written, harness skips it with a notice until the user approves the composition. Constraint proposed for approval: reuse only already-approved injected sentences from >= 2 different buckets and non-overlapping targets, so no new wording is invented.

**R5 [OPTIONAL, 07b] Gold labels (JSONL, one claim per line, one file per fixture).** Contains no transcript text: spans are offsets, evidence is turn indexes (excerpt text is sliced from the request at load time).

| Field | Meaning |
|---|---|
| `gold_version` | 1 |
| `fixture`, `claim_id` | `adhd-v3`, `g1`.. |
| `section_index`, `claim_text` | text is an exact substring of the section; `start`/`end` (code points) are derived by the builder and re-verified by tests |
| `role_in_fixture` | `target` (the injected/fault claim), `control` (faithful claim), `cdr` |
| `bucket`, `priority` | 1-12 (or null for control), `P0`/`P1`/`stretch`/`control` |
| `verdict` | one contract verdict |
| `concerns_required` / `concerns_allowed` | required must appear; allowed extras are not counted as false positives |
| `explanation_contains` | optional lowercase substrings, used only for direction (`inflates` / `understates`); direction follows SP03 `step_diff` sign (positive = claim stronger = inflates, negative = understates) |
| `evidence[]` | `{turn, role, roles_ok[], required, char_start?, char_end?}`; default whole turn; `required:false` turns are credit-free, penalty-free |
| `expect_warnings[]` | e.g. `ambiguous_speaker` (CDR) |
| `slots` | hand-labelled, field names and enums exactly SP03 `AtomicClaim` (R6): `scope` (`in_scope`/`out_of_scope`) + `oos_category?`, `subject_text`, `subject_speaker` (roster speaker or null), `reported_by` (list, >=1, of `{source: self\|informant\|interviewer\|assessor\|unspecified, speaker: str\|null}`), `polarity` (`affirmed`\|`negated`), `quantities` (0-2 of `{kind: frequency\|severity, raw, bucket, rate_per_week?}`), `timeframe` (`current\|past\|childhood\|future\|hypothetical\|unspecified`), `timeframe_explicit`, `hedged`, `links` (`{type: "causal", cause_ids, effect_id, connector, source: assessor\|reported}`). `bucket` uses SP03 `scales.py` names: frequency 0-5 `never, rarely, sometimes, often, most, always`; severity 0-4 `none, mild, moderate, severe, inability`. Not stored in gold: `id`, `section_index`, `segments`, `span_hint` (derived from `claim_text`/`start`/`end` by the adapter). Needed to run Linked/Gold because SP05 compares slots [RESOLVED: O-7] |
| `status` | `approved` (brainstorm-backed) or `proposed` |

Faithful gold (all `control`, `supported`, no concerns): S1 -> 1 claim (turns 1,3); S2 -> 2 ("misplacing his phone daily" 7; "has a system for keys" 9); S3 -> 2 ("some difficulty finishing household projects" 11; "partly because they ran out of paint" 11); S4 -> 2 (partner twice a month 19, 23 optional/qualifying; "Alex estimates once a month" 23); S5 -> 1 (15); S6 -> 1 (25); S7 -> 1 (5). Total 10 claims. `out_of_scope` is not exercised by the ADHD set (no conclusions in the rationale). Gold granularity is a proposal; matching (R7) tolerates the atomizer splitting differently. Gold `claim_id` (`g1`..) is a gold key only; the Gold/Linked adapter builds SP03 `AtomicClaim`s with ids `c{section}.{n}`, `segments=[claim_text]`, `span_hint=(start,end)`, and the slots above. Gold `quantities`/`reported_by` follow SP03 R6/R8 (e.g. S4 "Alex estimates once a month": `reported_by=[{self, alex1}]`, `hedged=true`, `sometimes`; V6 "both agree": one claim, two `reported_by` entries).

**R6 [OPTIONAL] Modes and adapters.** All modes end in the same path: internal drafts -> SP02 `assemble()` -> contract response -> scored. Scoring only ever sees the contract response, so assembler behavior is part of every measurement.

| Mode | Input | Runs | A bad score means |
|---|---|---|---|
| `gold` | gold claims (+slots, as SP03 `AtomicClaim`) and gold evidence (`required` and optional, with gold roles) as `ClaimDraft`/`EvidenceDraft` shells | check (SP05) only | checker weak |
| `linked` | gold claims | link (SP04) then check | linker weak, if gold is good |
| `e2e` | request only | full `Pipeline.run` (SP06) | atomizer weak, if linked is good |

`adapters.py` holds three thin functions that import the SP03-06 entry points; a missing module disables that mode with a clear message (so Gold mode works right after 01/02/05). SP02 interfaces reused: `ClaimDraft`, `EvidenceDraft`, `ExcerptSegment`, `InternalAudit`, `assemble`, `locate`. SP01 reused: `LLMClient`, `FixtureClient`/`StubClient`, `MetricsCollector`, `validate_request`/`validate_response`, `Settings`.

**R7 [OPTIONAL] Matching predicted claims to gold.** Per section, on code-point spans of the response. Overlap coefficient = intersection / min(len). Candidate pair if coefficient >= 0.5. Greedy one-to-one by IoU, descending; ties by earlier span. A gold claim split into several predicted claims (all inside the gold span) takes the worst verdict among them (severity order from SP02 R3) and the union of their concerns/evidence, so a finer atomizer is not penalized for finding the fault. Predicted claims matching no gold claim are "extra": each concern on an extra claim is a false positive for its bucket; extras with `supported`/`out_of_scope` cost nothing. Gold claims with no match count as missed (concern FN, verdict wrong). In Gold/Linked modes claim spans are identical, so IoU = 1.

**R8 [OPTIONAL] Metrics (per bucket, per mode; computed by `score.py`).**
- Bucket = mission check number of a concern type (contract order 1-12).
- Detection: unit = (matched gold claim, concern type b). TP: b predicted and in `concerns_required`. FN: required, not predicted. FP: predicted and not in required or allowed (including on extra claims). P, R, F1 with counts and n (gold positives). `n < 5` prints "indicative only"; the 0.8 P0 target and the +0.05 F1 rule are shown as flags only when `n >= 10` (configurable).
- Verdict accuracy per bucket (target claims) plus a verdict confusion matrix over all matched claims.
- Control row: false-alarm rate = faithful claims with any concern or a verdict other than `supported`.
- Evidence-span recall (turn-level): a predicted evidence span covers turn k if k lies in `[start.index, end.index]` and the span has >= 1 code point in k (so a cross-turn span covers its middle turns, and an end pointer at char 0 covers nothing). Recall = covered required gold turns / required gold turns. Also reported: turn precision (informational) and role-ok rate (predicted role in `roles_ok`). Gold-mode recall is a plumbing check (evidence is passed through).
- Bucket 1 = `supported` vs not on non-oos claims; the ADHD set has no isolated bucket-1 faults, so only control recall is meaningful; bucket 2 (generic contradiction) is likewise unmeasured in the ADHD set (05 O-13) [OPEN O-8].
- Latency: wall time per request and per call_tag (from `MetricsCollector`); p50/max; flag any request > `AUDIT_DEADLINE_S`. Cost: tokens in/out/cache from `MetricsCollector` x `prices.json` (Haiku 4.5 $1/$5, Sonnet 5.5 $2/$10, Opus 5.5 $4/$20 per M tokens, brainstorm figures, edited as pricing changes), reported as an estimate per mode and per request.
- Outputs under `--out`: `claims.jsonl` (per matched claim: ids, gold vs predicted verdict/concerns, evidence coverage; no text), `report.json`, `report.md` (bucket table, then a gold|linked|e2e side-by-side per bucket to localize the weak stage). `compare.py a.json b.json` prints per-bucket F1 deltas and flags >= +0.05 (the add-a-stage rule). No overall score anywhere.

**R9 [OPTIONAL] CLI, live opt-in, replay, recording manifest.**
`uv run python -m eval.run --mode gold|linked|e2e --fixtures adhd [--only v1,v3] [--replay | --live [--record]] [--max-cost USD] [--out DIR] [--gate poc] [--allow-test-split]`
- Default is `--replay` (SP01 `LLM_MODE=fixture`, `FIXTURE_DIR=eval/recorded`). `--live` requires provider keys in env and prints a cost estimate first; it never falls back silently, and a `FixtureMiss` in replay is reported as an error, not a score.
- Recording uses SP01's `LLM_RECORD=1`. Fixture keys are explicit and stable: `<request_id>/<call_tag>/<claim key>` (stages pass `fixture_key` from a scope on `RequestContext`; ask to SP03-06, O-6). A manifest stores per-stage prompt hash, model ids, date. Replay compares hashes: mismatch warns, `--strict-fresh` fails. Rationale: replay then tests plumbing, assembler and scoring without breaking on every prompt edit, while staleness stays visible.
- Recorded data is synthetic only. Recordings of private data go to `private/` and are never committed.
- `--gate poc`: per fixture, boolean pass/fail (not a score). Faithful passes iff every claim is `supported` with no concerns. A variant passes iff its target claim has the expected verdict, all `concerns_required`, and any `explanation_contains`. Evidence recall is printed but not gated. Collateral concerns on non-target claims of a variant are reported, not gated [OPEN O-9]. Exit 1 on any failure.

**R10 Self-tests (default pytest, no network).** Core subset (07a): request files validate against the schema, transcript pin, builder output equals committed files, expected files parse and `target_text` is a substring of its section. **[OPTIONAL, 07b]** the rest: Data tests: request files validate against the schema; transcript pin; gold spans round-trip; gold `slots` validate against SP03 `AtomicClaim` and bucket names against `scales.py` (loader fails on unknown field or enum); gold verdict/evidence roles satisfy contract rules 2-6; enums match the schema; builder output equals committed files. Scorer tests: hand-computed tiny cases for P/R/F1, matching (split, merge, extra claims), turn coverage (cross-turn, char-0 end). An oracle adapter that echoes gold must score 1.0 per bucket and pass the POC gate; a mutated oracle (flipped verdict, dropped concern, shifted span) must move exactly the expected cells. This proves the harness before any LLM exists.

**R11 [OPTIONAL] Leakage rule (prompts vs scored data).**
1. Scored data = everything under `eval/data/` and `eval/private/`. Few-shot examples and taxonomy examples used by SP03/04/05 prompts come from separate synthetic material written for the purpose: different domain or interview, different names, no reused sentence from a scored transcript, rationale or injection. SP03's few-shot file is `prompts/atomize_fewshot.json` (fictional "Jordan" examples, carries `split: "prompt"`); eval loaders must exclude any file with `split: "prompt"` from eval sets.
2. Test `test_prompt_leakage`: collects example text from the prompt directories (globs configured in `eval/leakage.toml`, must include `prompts/atomize_fewshot.json`; SP03-05 land) and fails if any 8-word window is shared with a scored transcript, rationale or injection, or if example speakers use the scored ids (`alex1`, `sam1`, `clinician1`). Same 8-word rule as SP03's own CI test, which scans `backend/tests/data` and this fixture dir; the two checks overlap by design and must not disagree on the window size. The brainstorm atomization example ("She is forgetful because...") is not in scored data and SP03 keeps it out of the prompt (test-only worked case).
3. Real CDR data is never in prompts, tuning loops or replay fixtures in the repo.
4. Reporting label: with only the approved ADHD set, results are labelled split `poc-all`. They are a smoke check on data the authors have seen, not a generalization estimate; no precision claim is made from them. When the 60/20/20 split exists: examples from train, tuning on dev, test run via `--allow-test-split`.

**R12 [OPTIONAL] CDR cleanup script SPEC (`eval/private/prepare_cdr.py`, to be implemented only when the files arrive).** Inputs: informant-interview file, patient-interview file, rater rationale file (all absent from the repo today: the script and gold are written against the user's files, never reconstructed).
- Concatenate informant interview then patient interview (344 turns in the prior review; assert the count and print mismatch, do not auto-fix).
- Informant file: `Speaker 1` -> speaker `clinician1`, role `interviewer`; `Speaker 2` -> `son1`, `informant`. Patient file: `Speaker 1` -> `clinician1`, `interviewer`; `Speaker 2` -> `patient1`, `respondent`. Any other label: stop with an error.
- Drop watermark lines using a `--watermark-regex` supplied by the user (pattern unknown; none invented). Report count dropped.
- Keep diarization merges as-is (turns 68 and 166 in the prior review; indexing base unknown, confirm against the file). Proposed: emit `role` omitted on those two turns so SP05's locked missing-role rule produces `ambiguous_speaker`; speaker text and turn text are never edited [OPEN O-10].
- Do not trim, NFC-normalize or re-wrap text (SP02 offsets are computed on strings as received).
- Output: contract `AuditRequest` + a gold skeleton file; everything lands in gitignored `private/`.
- Gold (from the user-accepted mock review, to be transcribed by the user): orientation "rarely" vs the son's answers on date and day (contradiction-ish, verdict [OPEN O-11]); problem solving "some difficulty" vs "Poor" -> `partially_supported` + `frequency_severity_inflation`, `explanation_contains: ["understates"]`; claims relying on turns 68/166 -> `expect_warnings: ["ambiguous_speaker"]`. Test-only: runs need `--allow-test-split`.

**R13 [OPTIONAL] Bucket-1 baseline hook (parked).** `baselines.py` defines `Bucket1Baseline.predict(claim_text, evidence_text) -> supported|not_supported` and a registry with `minicheck-flan-t5-l` and `gpt-4o-mini` entries that raise `BaselineNotInstalled` with an install note. Heavy deps would live in an optional `baselines` extra, never in the default env. `--baseline NAME` runs it on the Gold-mode bucket-1 inputs and emits the same report shape. DEFERRED: no implementation of the models now.

### 07a Fixtures (Core), continued


**R5a Simple expected format (Core, 07a).** One small JSON per fixture, `data/adhd/expected/<fixture>.json`, generated by the builder:
```json
{"fixture": "adhd-v1", "target_text": "<exact substring of the section text>",
 "verdict": "partially_supported", "concerns_required": ["frequency_severity_inflation"],
 "concerns_allowed": [], "explanation_contains": ["inflates"]}
```
Faithful: `{"fixture": "adhd-faithful", "all_claims": {"verdict": "supported", "concerns_required": [], "concerns_allowed": []}}` (no `target_text`). Variants use the R3 table's expected verdict + concern; V2 may list `contradictory_evidence` under `concerns_allowed`. No evidence, slots or offsets in core.
**Forward compatibility with 07b:** field names (`fixture`, `verdict`, `concerns_required`, `concerns_allowed`, `explanation_contains`) are exactly the R5 gold JSONL names; `target_text` becomes R5 `claim_text`. 07b extends by adding `claim_id`, `section_index`, `start`/`end`, `role_in_fixture`, `bucket`, `priority`, `evidence[]`, `expect_warnings`, `slots`, `status` and by emitting JSONL (one claim per line) from the same spec; the builder keeps writing the simple file so the smoke script never changes. Extra keys are ignored by the smoke script.

**R14 Smoke script (`backend/eval/smoke.py`, Core).** `uv run python -m eval.smoke [--base-url URL | --inprocess] [--only v1,v3]`.
- Loads the 9 request files and their `expected/*.json`; for each, sends the request to `POST /v1/audit` (default: in-process FastAPI `TestClient` against the app factory; `--base-url` targets a running server) and reads the response.
- Finds the target claim as the response claim whose resolved span (positions sliced from the request section text) overlaps `target_text`; if the atomizer split it, takes the worst verdict among overlapping claims and the union of their concerns. No fancy matching: this is a smoke check, the 07b matcher (R7) supersedes it.
- Prints one line per fixture: fixture id, expected verdict + concerns, actual verdict + concerns, PASS/FAIL. Pass iff target verdict equals expected, every `concerns_required` is present, every `explanation_contains` substring appears; faithful passes iff every claim is `supported` with no concerns. Collateral concerns on other claims are printed, not gated. Exit 1 on any FAIL.
- Needs 06 (`IAUDIT_PIPELINE=full`) and provider keys for a meaningful live run; with `IAUDIT_LLM_MODE=stub` or `IAUDIT_PIPELINE=stub` it still runs and reports the stub output (all FAIL), which proves the plumbing. Prints no transcript text.

## Architecture
[Core: spec.json -> build_adhd.py -> requests + expected -> smoke.py -> API/pipeline -> pass/fail. The rest is 07b, Optional.]
```
spec.json --build_adhd.py--> requests/*.json + expected/*.json (core) + gold/*.jsonl (07b)   (committed, reproducible)
run.py --mode--> adapters (SP03-06) -> InternalAudit -> assemble (SP02) -> response
        |  LLM via SP01 client: live | fixture replay | record
        v
match.py (gold<->predicted claims) -> score.py (per-bucket, evidence, latency, cost) -> claims.jsonl, report.json/md
compare.py (F1 deltas), --gate poc (per-fixture pass/fail)
```

## Decisions
| # | Decision | Choice | Alternatives considered | Why |
|---|---|---|---|---|
| D1 | Data location | `backend/eval/data/` | `docs/examples/`; `tests/` | docs/examples is for public usage samples; eval is code-consumed test infra; outside the package so it is not shipped |
| D2 | Fixture source | One spec + deterministic builder, outputs committed | Hand-edited 9 JSON files | Variants differ by one sentence; copies drift; test pins equality |
| D3 | Injection op | Replace the named sentence, or append at end when no faithful sentence is displaced | Always append | Replace keeps the rationale length and style; appending is used where the fault is a new assertion (V3-5) |
| D4 | Gold content | Offsets + turn indexes, no transcript text | Store excerpts | No duplicated text to drift; excerpts sliced at load; matches "positions only" contract spirit |
| D5 | Evidence granularity | Turn-level overlap | Char-level | Gold turns are cheap and stable; char refinement possible later via optional fields |
| D6 | Concern gold | required + allowed sets | Single expected concern | Absorbs legitimate co-concerns (O-1) without hiding wrong ones |
| D7 | Scoring surface | Contract response after `assemble()` | Internal drafts | Measures what the UI gets, includes span resolution; one scorer for all modes |
| D8 | Claim matching | Overlap coefficient >= 0.5, greedy IoU, split takes worst verdict | Exact text; IoU only | Atomizer granularity will differ from gold |
| D9 | Replay keys | Explicit stable keys + prompt-hash manifest | Request-hash keys | Hash keys break on every prompt edit; manifest keeps staleness visible |
| D10 | Gate | Per-fixture boolean on verdict + concern | Aggregate pass rate | Locked bar; n is tiny; avoids a single score |
| D11 | Small-n honesty | Show counts, flag n<5, show targets only n>=10 | Print P/R/F1 plainly | 9 fixtures cannot support precision claims |
| D12 | Self-test | Gold-echo oracle + mutations | Wait for live runs | Harness verified before keys exist |

## Manual steps (human-only)
- Provide provider API keys (none on this machine); core: run the smoke script live; **[OPTIONAL]** run the first `--live --record` pass and commit the recordings.
- Approve or amend gold marked `proposed` and the open ambiguities below; approve multi-fault composition (O-5).
- Supply the CDR files, the watermark pattern and the accepted mock-review gold; confirm turn numbering for 68/166.

## Risks / Open Questions
- [OPEN] O-1 V2: expected concern is only `negation_reversal` in the brainstorm. Proposal: also allow (not require) `contradictory_evidence`. Confirm. Per 05: it yields only `negation_reversal` for V2 (generic suppressed by dedup); keeping `contradictory_evidence` in `concerns_allowed` stays allowed as a safety net. Bucket 2 (generic contradiction) is unmeasured in the ADHD set (see O-8).
- [OPEN] O-2 V4: rule 5 forbids a contradicting excerpt under `insufficient_evidence`. Proposal: turn 12 (the question; SP03 act `question`) as `qualifying`, turn 13 ("Hm."; SP03 act `backchannel`, not `answer`) optional, empty evidence also passes; `supporting` would be wrong (a question is not an answer). Confirm role, and whether to require any evidence. Per 05: V4 emits the 12-13 pair as one `qualifying` item (04 snaps, 02 merges), so a single qualifying 12-13 item qualifies; recommend `qualifying` required-optional.
- [OPEN] O-3 V5 (timeframe) is a stretch bucket: turn 17 is childhood and "stopped by high school"; role proposal `qualifying`. Does a V5 miss block POC acceptance? Locked bar says all 8 should pass; stretch makes a miss likely. Confirm treatment (gate vs report).
- [OPEN] O-4 Partial-support roles (V1, V6, V7, V8): which evidence role is expected is judgment; `roles_ok` sets are proposals; role is not gated.
- [OPEN] O-5 Multi-fault content and composition (R4).
- [OPEN] O-6 Stages must pass `fixture_key` from a scope on `RequestContext` (SP01 R6/R8, SP03-06). If not adopted, fall back to request-hash keys and re-record on every prompt change.
- [RESOLVED: aligned to SP03 `AtomicClaim` slot names (`subject_text`, `subject_speaker`, `reported_by` list of `{source, speaker}`, `polarity` affirmed|negated, `quantities` with `scales.py` buckets, `timeframe` + `timeframe_explicit`, `hedged`, `links`)] O-7. Gold loader validates against the model (R10).
- [OPEN] O-8 Bucket 1 has no isolated negative in the ADHD set; need pure-unsupported variants (new wording, user approval) before bucket-1 metrics or baselines mean anything.
- [OPEN] O-9 Whether collateral false alarms on non-target claims in variants gate acceptance (proposal: report only).
- [OPEN] O-10 Role handling for merged CDR turns (omit role proposed) versus mapping the label and flagging separately.
- [OPEN] O-11 CDR orientation expected verdict/concern (`contradicted` + `contradictory_evidence` vs a milder verdict); text of the accepted mock review is not in the repo.
- [OPEN] O-12 Faithful sensitivity: S7 "denies" from "Not really" and S3 "some difficulty" from "Sometimes" may be flagged by a strict verifier; user approved the text, so gold stays `supported`; failures there are checker findings, not gold errors.
- [OPEN] O-13 Variant 1 wording: brainstorm gives only the fragment; the added subject is a proposal.
- [RESOLVED: SP02 `ClaimDraft` adopts SP03 `segments` + `span_hint`; `locate()` (02) is the only offset resolver, called by SP03 per segment; unresolved segments are an SP03 error/retry, then dropped; SP02 re-verifies hints] O-14 (was: SP03 emits `segments` (<=3, possibly non-contiguous) + `span_hint` and flags `unresolved` segments for 02's fuzzy fallback; SP02 `ClaimDraft` has one contiguous `claim_text` + `char_hint` and re-locates. Mapping for non-contiguous claims and who owns unresolved segments is unsettled (SP03 O-1). Gold claims are contiguous, so Gold/Linked mode is unaffected; e2e matching (R7) depends on the outcome. Not chosen here.)
- [OPEN] O-15 SP03 gold-relevant fields not hand-labelled here: `proposition` (self-contained rewrite, needed by SP04/05 in Gold mode). Needs user-approved wording or the adapter derives it from `claim_text`. Also S1 mixes relative and numeric ("most meetings" = 4, "three to four times a week" = 3.5 = `often`, SP03 R8 known limit): gold `bucket` for S1 and the one-step tie band need confirming.
- [DEFERRED] 60/20/20 split, P0 precision >= 0.8 gate, P1 suppression threshold (<0.6), shortlist-recall metric for the linker (needs SP04 to expose its shortlist), char-level evidence gold, baselines (MiniCheck, GPT-4o-mini), a committed baseline report, copy of the request into `docs/examples/`.

## Acceptance Criteria
1. `uv run pytest` (no network, no keys) passes the R10 core data tests (07a); **[OPTIONAL]** also the scorer and oracle tests; `build_adhd.py` regenerates byte-identical committed files.
2. All 9 request files validate against `audit-request.schema.json`; turns match the brainstorm table exactly (pinned hash); variants differ from the faithful text only by the single sentence in R3.
3. **[OPTIONAL]** `--mode gold --replay` with the echo oracle scores 1.0 on every populated bucket and passes `--gate poc`; each mutation in the self-test changes only its expected metric.
4. **[OPTIONAL]** A report contains per-bucket P/R/F1 with counts, verdict accuracy, turn-level evidence recall, latency (p50/max) and estimated cost, and no overall score; `compare.py` flags a >= 0.05 F1 delta.
5. Core (via smoke script): faithful returns every claim `supported` with zero concerns; each variant's target claim has the expected verdict + required concerns (V1 explanation contains "inflates"); script exits 1 on any failure. **[OPTIONAL]** gold-claim counts, per-variant evidence recall, gold/linked/e2e modes. Original text: Faithful: all 10 gold claims `supported`, zero concerns, in gold mode (and in linked/e2e after 04-06). Each variant: target claim has the expected verdict + required concerns (V1 explanation contains "inflates"). Evidence recall printed per variant.
6. **[OPTIONAL]** `--live` refuses to run without keys and prints a cost estimate; replay never touches the network and reports `FixtureMiss` as an error; a stale prompt hash warns (fails with `--strict-fresh`).
7. **[OPTIONAL]** `test_prompt_leakage` fails on a seeded 8-gram copied from a scored fixture and passes on clean examples.
8. **[OPTIONAL]** `prepare_cdr.py` spec is recorded as above; no CDR turn, rationale or gold text exists anywhere in the repo.
