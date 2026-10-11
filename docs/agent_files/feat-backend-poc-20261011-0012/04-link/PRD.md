---
status: draft
summary: Stage 2 Link - per-request BM25+rapidfuzz turn shortlist with question/answer/correction/other-speaker neighbors (or whole transcript under a benchmarked size), one mid-tier call per checkable claim returning verbatim excerpts with role, link type and inference premises, validated against the shortlist and handed to the assembler.
date: 2026-10-11
---
# PRD: 04 Link

Repo/branch: iaudit / `feat/backend-poc` · Depends on: 01-scaffold-api, 02-assembler, 03-prep-atomize (imports 02's `ExcerptSegment`/`EvidenceDraft`/`locate`) · Owns: `backend/src/iaudit_backend/link/` (`tokenize.py`, `retrieve.py`, `shortlist.py`, `prompt.py`, `validate.py`, `linker.py`, `models.py`), `prompts/link_system.md`, `prompts/link_fewshot.json`, tests

## TL;DR
For each in-scope atomic claim, find the transcript evidence. Cheap code picks a shortlist (or the whole transcript when small); one mid-tier call per claim, parallel, returns verbatim excerpts per turn with evidence role (supporting/contradicting/qualifying), link type (quote/paraphrase/inference) and, for inference, premise excerpts plus the inferential step. Code then rejects anything not in the shortlist and pre-checks each excerpt with 02's `locate()`. A claim whose link call fails is reported `stalled` for 06. Linking, not judging: SP04 never decides a verdict, attribution, or question-as-answer concern (05 does); it supplies the facts those checks need.

## Problem
Evidence linking is the hard part (AnchorSIPS evidence-link F1 <= 0.290; TROVE retrieval 51.18 vs 9.61 F1 without). The ADHD set shows what a naive retriever misses: variant 3 needs informant turn 21 (shares only "appointments"), variant 4 needs a question plus a "Hm." that shares no words with the claim, variant 6 needs turns 19 and 23 from two speakers. Whole-transcript prompts per claim get expensive on long interviews. Roles are optional, so nothing here may rely on `role`. [verified: brainstorm variants table]

## Goals / Non-Goals
**Goals**
- High-recall shortlist without embeddings/spaCy; recall measurable offline without LLM calls.
- Verbatim, turn-bounded excerpts only; cited turns must come from what the linker was shown.
- Surface adverse evidence too (contradicting, qualifying, other speakers, later corrections, non-answers).
- Honest per-claim failure reporting; prompt layout friendly to caching.

**Non-Goals**
- Offsets and fuzzy resolution (02), atomizing/acts (03), verdicts/concerns/explanations/attribution/bucket-9 decisions (05), timeouts and wiring (06), eval harness (07).
- Searching outside the shortlist or relocating a quote to another turn (02 D-3).
- Counterevidence search for `out_of_scope` conclusions (deferred in brainstorm). Out-of-scope claims are not linked.
- Embeddings (see O-4), speaker/role inference, diarization repair.

## Scope tiers
**Core** (must ship): R1 shortlist as plain `BM25Okapi` top-k (tokenizer, index docs with paired-question tokens, query) plus neighbors: the preceding question for a hit that responds, the following reply/replies for a hit that is a question, and the clarification window; R2 whole-transcript mode; R3 linker prompt/output (one mid call per in-scope claim, inference shape); R4 validation (drop out-of-shortlist cites, `locate()` dry run, question snapping, inference checks, `answer_status` computed from acts with `supporting` demoted to `qualifying` for question-only/non-answer, dedup, caps, `ambiguous_turns`); R5 retry and `stalled` failure; R6 with a single `cache_prefix` (system + transcript in whole mode; system + stage block in shortlist mode); R7 `LinkResult` fields; R8 logging/config; `build_link_context`, `link_one`, `link_claims`, `to_evidence_drafts`. Decisions D2, D4-D8, D10, D11 and D1 (BM25 only). Acceptance criteria 1-6, 7 (single-prefix form), 8 (without `shortlist_only`), 10.

**Optional (gated: after all core work in every SP)**:
- [OPTIONAL] Per-speaker floor (R1.4, `LINK_PER_SPEAKER_FLOOR`, `LINK_MAX_SPEAKER_ADDS`, D3). Core fallback: BM25 top-k only (k may be raised via `LINK_TOP_K`); other-speaker turns appear if they rank.
- [OPTIONAL] Later-correction pull (R1.5 bucket 10). Core fallback: the clarification window (R1.5) only.
- [OPTIONAL] Rank fusion with rapidfuzz `token_set_ratio` by RRF and its tuning (R1.3, D1 fusion part, O-3 tuning). Core fallback: BM25 alone. The R1.7 rapidfuzz-only fallback becomes: whole transcript if under `LINK_FALLBACK_MAX_TOKENS`, else `items: []` with status `ok`, counted as `empty_shortlist`.
- [OPTIONAL] `shortlist_only()` recall hook and `Retriever` protocol (R7). Core fallback: 07 reads `LinkResult.shortlist` from a stub-linker run (no LLM cost) for retrieval recall.
- [OPTIONAL] Shared cache layout tuning (R6 two-breakpoint `cache_prefixes`, 06 `layout.py` alignment, D9, O-5, O-10). Core fallback: one `cache_prefix` per call.
- [OPTIONAL] Few-shot leak CI test (R3 leak rule, acceptance 9). Core fallback: reviewed by hand.

## Requirements

### R1 Shortlist (`shortlist.py`, `retrieve.py`, pure code)
Built once per request: a `TurnIndex` over all turns; per-claim `shortlist(claim) -> Shortlist(indices, mode, scores)`.
1. **Tokenize** (`tokenize.py`): NFKC, lowercase, `[a-z0-9']+`, built-in English stopword list (negators and numbers kept in text sent to the LLM, dropped from the query only if stopwords), in-house light suffix stripper (`ies/es/s/ing/ed/ly`, min stem 3) plus a prefix-5 token as a second channel so "forgets"/"forgetful" and "weekly"/"week" meet. English only.
2. **Index doc** per turn = its own tokens; for turns with `responds_to` and <= 12 tokens, plus the paired question's tokens (so "Yes." / "Hm." inherit the question topic).
3. **Query** = `proposition` + `segments` + `subject_text` + `quantities[].raw` (dedup). Core ranker: `rank_bm25.BM25Okapi`, top `LINK_TOP_K` (8) with positive score. [OPTIONAL: second ranker, `rapidfuzz` `token_set_ratio` of the proposition against each turn, fused with BM25 by reciprocal rank (k=60).]
4. [OPTIONAL] **Speaker diversity**: for each distinct `speaker` string that has a positive-score turn but fewer than `LINK_PER_SPEAKER_FLOOR` (2) hits in the top-k, add its best turn(s), at most `LINK_MAX_SPEAKER_ADDS` (4) in total. Speaker strings, not roles. This is what surfaces informant turns (variants 3, 6; bucket 11).
5. **Neighbors** for each hit h (using 03's `acts`, `responds_to`, `QAPair`):
   - h responds to a question: add the question (and the other responses in that pair).
   - h is a question: add its `response_indices` (cap 3) - catches "Hm." (variant 4).
   - Clarification: within `LINK_CORRECTION_WINDOW` (4) turns after h, add turns with act `correction`, or by the same speaker as h with positive score.
   - [OPTIONAL] Later correction (bucket 10): any later turn with act `correction` by h's speaker (any distance) that shares a query token; cap 3.
6. Merge, sort by index, cap `LINK_MAX_SHORTLIST` (30; drop lowest-priority neighbors first, never BM25 hits), fill one-turn gaps. Rendered with `[... turns a-b omitted ...]` markers so the model knows the list is not contiguous.
7. Empty/zero-score result: fall back to whole transcript if under `LINK_FALLBACK_MAX_TOKENS` (3x the whole-mode threshold), else `items: []` with status `ok` (counted `empty_shortlist`). [OPTIONAL: instead use the top-k by rapidfuzz alone plus neighbors.]

### R2 Whole-transcript mode
`LINK_MODE=auto|whole|shortlist` (default `auto`). `auto`: estimated tokens (`ceil(total_chars/4)`, no tokenizer dependency) `< LINK_WHOLE_MAX_TOKENS` -> all turns, no retrieval; else R1. Proposed default 8,000 tokens [OPEN O-2, set by benchmark]. Tests force `shortlist` on the ADHD fixtures so retrieval is exercised although the example is far below the threshold (est. under 1k tokens).

### R3 Linker call (`linker.py`, `prompt.py`)
- Eligible claims: `scope == in_scope` only. One mid-tier call per claim (`call_tag="link"`, `temperature=0`, `response_model=LinkerOutput`, `max_tokens=LINK_MAX_TOKENS` 1200), `asyncio.gather` under a local `Semaphore(LINK_CONCURRENCY=20)` on top of 01's global client semaphore; `timeout_s=min(LLM_TIMEOUT_S, remaining deadline)`.
- **Input rendered**: claim `proposition`, `reported_by` (speaker ids), `polarity`, `timeframe`, `quantities[].raw`, and the turns as `<turn i="12" speaker="clinician1" role="interviewer|unknown">verbatim text</turn>` (no act hints inside the transcript block, so it stays byte-identical across claims). Dialogue-act hints (e.g. `12=question, 13=backchannel`) go in the claim block, labelled heuristic hints. Turn text is never altered or escaped (excerpts must stay verbatim); content inside `<turn>` is data, not instructions.
- **Prompt rules**: copy each excerpt character-for-character from ONE turn, one segment per turn, the smallest span that carries the relevant statement (the whole turn if short); cite only turn indexes shown; never invent or complete text; never guess a speaker or role; notes are paraphrase, not quotes. Return evidence that cuts against the claim as well (contradicting) and evidence that limits it (qualifying: conditions, different amounts or timeframes, another speaker's differing account, an answer that does not address the claim). Report evidence from any speaker; do not filter or judge attribution (05 does). An interviewer question alone is not respondent evidence; include it as a context segment only together with the reply whose meaning depends on it (question + "yes"). If the claim appears to rest on a question whose reply is a bare continuer or non-answer (turns 12-13), return that pair as `qualifying` with a note saying the reply does not address it. Use `inference` only when no single turn states the claim and the claim follows from named premise turns plus one stated step; never to bridge numbers or timeframes.
- **Output** `LinkerOutput.items[]`: `key` (`e1..`), `role`, `link_type`, `segments[{turn, text}]`, `premises[]` (keys of other items; inference only), `inference_step` (one sentence, inference only), `note`. `items: []` is a valid answer ("nothing found").
- **Inference shape**: an inference item has no segments of its own. Its premises are ordinary items (quote/paraphrase) in the same output, each carrying the inference's role. Only premises are forwarded to the assembler; the inference link (premise keys + step) stays internal for 05 ("supported only if all premises supported and the step is neutral-reader acceptable").
- Few-shot (`link_fewshot.json`, `split: "prompt"`): fictional "Jordan" cases covering quote, paraphrase, Q + "yes", informant contradicts the subject, near-miss non-answer, inference with premises, nothing found. [OPTIONAL] Same leak rule as 03 R10: CI fails on any 8-gram overlap with eval fixtures/gold files.

### R4 Validation (deterministic, per claim, after parse)
1. Drop segments whose `turn` is not in the shortlist (counted `not_in_shortlist`); drop empty text.
2. For each segment, run 02's `locate(turn_text, segment_text, None, cfg)` in dry mode (offsets discarded; the assembler recomputes). Unmatched: drop (`unlocatable`). Item with no segments left: drop (and its inference parents).
3. Sort segments by turn index. **Question snapping**: a segment from a `question`-act turn that is a context segment of a Q+reply item is replaced by the full turn text, and a reply segment in a turn of <= 80 chars by the full turn text, so 02's cross-turn merge rule (first segment runs to end of its turn, last starts at turn start) holds. Snapped text is still a verbatim substring of that turn.
4. Inference: premises must exist, be non-inference items, no cycles, `inference_step` non-empty; else the inference item is dropped and its premises kept as `qualifying`.
5. `answer_status` computed from 03 acts, not by the LLM: `endorsed` (a non-backchannel `answer` turn from a different speaker than the question is among the segments), `question_only` (only question turns), `non_answer` (question + backchannel), else `n/a`. Items that are `question_only`/`non_answer` and labelled `supporting` are coerced to `qualifying` (counted `coerced`) so a bare question can never produce a `supported` verdict through rule 2.
6. Dedup identical (turn set, role); cap `LINK_MAX_ITEMS` (8) per claim, adverse items kept first.
7. `ambiguous_turns` = cited turns with `role is None`, passed on for 02's `ClaimDraft.ambiguous_turns` and the `ambiguous_speaker` warning.

### R5 Retry and failure
- The client's one transport/invalid-JSON retry (01 R6) is separate. SP04 adds **one** semantic retry only when the first output claimed items and R4 dropped all of them; the retry resends with a terse error list (kinds and counts, no text), and is skipped if remaining budget < one call timeout + margin.
- Optional `LINK_WIDEN_ON_EMPTY` (default false): if shortlist mode returns `items: []`, retry once with the whole transcript when under the fallback cap. Off until eval shows gain (O-3).
- `LinkResult.status = "stalled"` with `reason` in `timeout | unavailable | bad_output | all_dropped` when the call fails or the retry still yields only invalid items. `[]` items with status `ok` means "searched, found nothing". 06 maps `stalled` to `ClaimDraft.outcome="stalled"` (claim -> `insufficient_evidence` + `other` warning naming the claim, locked). `link_claims` never raises for per-claim failures; `LLMUnavailable` on every claim is surfaced to 06, which decides on 503.

### R6 Prompt caching (with 01 `cache_prefix`, 06)
Core: one `cache_prefix` per call (whole mode: common system + transcript rendered without act hints; shortlist mode: common system + link stage block); the claim block goes last. Everything below about `cache_prefixes`/shared layout is [OPTIONAL] tuning.
- **Shared layout [RESOLVED: per 06 R6]**: prompts are built with 06's `pipeline/layout.py` (system -> transcript -> stage -> claim), shared with 05; 04 no longer renders its own prefix. Via `LLMRequest.cache_prefixes`: whole mode = [common system + transcript (rendered without act hints)], [link stage block: instructions + taxonomy + few-shot]; the claim block (act-hint lines, claim, etc.) goes last. Shortlist mode: no transcript prefix; prefix = common system + link stage block; the claim block carries the shortlist. The earlier per-stage layout (system + few-shot + transcript) is superseded.
- Rendering is a single deterministic `render_turns(turns, with_acts)`; no timestamps or per-call data before the claim.
- Built-in warm-up is dropped [RESOLVED: per 06 R6, warm-up runs at t=0 in 06 `warmup.py` while atomize runs; `LINK_WARMUP` superseded, default off].
- Provider minimum cacheable length and `cache_read` surfacing through LiteLLM are 01 O-2; SP04 only sets the hint and reads `usage` from `MetricsCollector`.

### R7 Eval hooks (for 07 Linked mode)
- [OPTIONAL] `shortlist_only(claim, index, cfg) -> Shortlist` is a pure function: 07 measures **retrieval recall** (gold evidence turns in shortlist, by k and bucket) with no LLM cost.
- `LinkResult` exposes `mode`, `shortlist`, `evidence` (turn indexes per item) so 07 computes **evidence-span recall** against gold turns/spans, per bucket, never one overall number.
- [OPTIONAL] `Retriever` protocol (`rank(claim) -> [(turn_index, score)]`) lets an embedding retriever be benchmarked later without touching the linker. Embeddings enter only if linked-mode recall improves F1 by >= 0.05 at acceptable cost.

### R8 Logging and config
Counts only per claim: mode, shortlist size, items returned/kept, drops by reason, coerced, retries, stall reason, tokens in/out/cache_read. Never claim text, turn text, excerpts, notes (01 R9 allowlist). Config additive to 01 `Settings`: `LINK_MODE`, `LINK_WHOLE_MAX_TOKENS`, `LINK_FALLBACK_MAX_TOKENS`, `LINK_TOP_K`, `LINK_PER_SPEAKER_FLOOR`, `LINK_MAX_SPEAKER_ADDS`, `LINK_CORRECTION_WINDOW`, `LINK_MAX_SHORTLIST`, `LINK_MAX_ITEMS`, `LINK_MAX_TOKENS`, `LINK_CONCURRENCY`, `LINK_RETRY`, `LINK_WIDEN_ON_EMPTY`, `LINK_WARMUP`, `LINK_WARMUP_MIN_CLAIMS`.

## Architecture
```
turns + PrepResult(acts, qa_pairs) -> TurnIndex (once)
claim (in_scope) -> shortlist (BM25+rapidfuzz RRF, speaker floor, neighbors) | whole
   -> render + mid call (parallel, sem 20) -> parse -> validate (shortlist, locate dry-run, snap, inference, answer_status)
   -> LinkResult -> [05 check]  and  to_evidence_drafts() -> [02 assemble via 06]
```
Public surface:
```python
def build_link_context(turns: list[Turn], prep: PrepResult, ctx) -> LinkContext   # TurnIndex + mode (whole/shortlist), built once per request (per 06 R8)
async def link_one(claim: AtomicClaim, lctx: LinkContext, ctx) -> LinkResult   # single unit, called per claim by 06; the stage semaphore (LINK_CONCURRENCY) lives on lctx
async def link_claims(claims: list[AtomicClaim], prep: PrepResult, turns: list[Turn], ctx) -> list[LinkResult]   # gather wrapper over build_link_context + link_one
# [OPTIONAL] def shortlist_only(claim, index, cfg) -> Shortlist
# LinkResult: claim_id, status ok|stalled, reason, mode, shortlist: list[int],
#   evidence: list[LinkedEvidence], ambiguous_turns: list[int]
# LinkedEvidence: draft: EvidenceDraft (02: key, role, segments[ExcerptSegment(turn_index, text)], note),
#   link_type, premise_keys, inference_step (internal, never emitted), answer_status
def to_evidence_drafts(r: LinkResult) -> list[EvidenceDraft]   # drops inference items, keeps premises
```
Uses 01 `LLMClient.complete_json`, `RequestContext`, `Settings`, tier `mid`, Stub/FixtureClient; 03 `AtomicClaim`, `IndexedTurn`, `PrepResult`, `QAPair`, `Act`; 02 `EvidenceDraft`, `ExcerptSegment`, `locate`. Evidence keys are `"{claim.id}:e{n}"`; 02 assigns final `eN` ids.

## Decisions
| # | Decision | Choice | Alternatives considered | Why |
|---|---|---|---|---|
| D1 | Retrieval | BM25 (core) [OPTIONAL: + rapidfuzz fused by RRF], no embeddings | BM25 only; embeddings now | Locked: no torch; fuzzy covers morphology; embeddings gated on >=0.05 F1 |
| D2 | Short answers | Index "Yes."/"Hm." with paired question tokens, plus pair neighbors | Neighbors only | Answers share no words with the claim |
| D3 | [OPTIONAL] Other-speaker recall | Per-speaker floor in top-k | Rely on rank | Variant 3/6 informant turns rank low; buckets 3, 11 need them |
| D4 | Whole vs shortlist | Auto by estimated tokens; forceable | Always shortlist; always whole | Short transcripts lose nothing from whole send; long ones cost 30 x T tokens |
| D5 | Inference representation | Premises are normal items; link (keys+step) internal | Inference item with own span | Contract has only spans; each premise must be a visible, checkable excerpt |
| D6 | Answer status / question-only | Computed in code from 03 acts; `supporting` coerced to `qualifying` | LLM labels it; leave to 05 | Deterministic guard so rule 2 cannot be satisfied by a question |
| D7 | Excerpt validation | Shortlist membership + 02 `locate` dry run, drop failures | Trust LLM; resolve offsets here | Blocks invented quotes early and enables one retry; offsets stay 02's |
| D8 | Failure | `stalled` per claim, no raise | Fail request | Locked decision; 06 maps to insufficient_evidence |
| D9 | [OPTIONAL] Cache layout | Shared prefix per 06 (system -> transcript -> stage -> claim); no built-in warm-up in 04, warm-up owned by 06 | Per-stage prefix; 04 warm-up | Parallel first calls all miss the cache; 06 warms at t=0 |
| D10 | Stemmer | In-house suffix stripper + prefix-5 | nltk/snowball dependency | No new deps; English-only POC |
| D11 | Role semantics | Role is relative to the claim as written | Relative to the interview | 05's rules 2/3/5 reason from claim support |

## Manual steps (human-only)
- Threshold benchmark (O-2) needs a provider API key (owner step in 01) and, for the real long-transcript timing, the CDR file (test-only; never invented).

## Risks / Open Questions
- O-1 Model mismatches: (a) [RESOLVED: 02 `ClaimDraft` uses 03 `segments` + `span_hint`; 02 `locate()` is the single matcher] (was `claim_text`+`char_hint`); (b) [RESOLVED: `LinkedEvidence` wrapper adds link_type/premises/inference_step, internal to 04/05, not sent to 02; 02 unchanged]; (c) [RESOLVED: 02 path is `backend/src/iaudit_backend/assembler/`]; (d) [RESOLVED: 04 now declares dependency on 02]; (e) 03 says 02 finishes `unresolved` segments, 02 R3 re-locates everything (consistent, confirm); (f) [OPEN] `ambiguous_turns` is filled by 04 for evidence turns only; 06/05 merge other sources.
- [OPEN] O-2 `LINK_WHOLE_MAX_TOKENS` 8,000 is a proposal. Benchmark whole vs shortlist at 4k/8k/16k on dev + 3x synthetic transcript: evidence-span recall, p50/p95 latency of the Link stage (budget 8-12 s), tokens. Cost shape (estimate): 30 claims x 8k tokens = 240k input tokens, about $0.48 uncached at $2/M vs about 60k tokens in shortlist mode.
- [OPEN] [OPTIONAL tuning] O-3 Shortlist defaults (k=8, floor 2, window 4, cap 30) and the stemmer are guesses; tune on retrieval recall@shortlist; `LINK_WIDEN_ON_EMPTY` on/off by eval.
- [OPEN] O-4 Embedding retriever: only if linked-mode recall gain >= 0.05 F1 [DEFERRED until then].
- [RESOLVED: per 06] O-5 Warm-up and shared prefix: 06 warms the cache at t=0 (not serialized in Link) and 04/05 share a transcript-first layout in whole mode (06 R6).
- [RESOLVED: per 06] O-10 D9, AC 7 and R3 now follow 06's layout (system -> transcript -> stage -> claim); no warm-up in 04; act hints in the claim block.
- [RESOLVED: per 05, `qualifying` is the role for non-answer pairs; rule 5 only forbids `contradicting` under `insufficient_evidence`] O-6 Role of near-miss/non-answer pairs: `qualifying` is a convention for 05 (rule 5 forbids `contradicting` on `insufficient_evidence`); confirm with 05 and with how 02's repair treats it.
- [OPEN] O-7 BM25Okapi idf can be zero/negative on tiny corpora (epsilon floor); BM25Plus or `snowballstemmer` are swaps if tests show noise.
- [OPEN] O-8 Question snapping edits what the highlight covers (full question turn); check reviewer value vs over-highlighting.
- [OPEN] O-9 Variant 6 (one claim, two `reported_by`): linker labels turn 23 `contradicting` or `qualifying`? Test both against 05's expected `partially_supported` + `conflicting_accounts_flattened`.
- [RESOLVED: 03 R4 pairing by speaker string] Missing roles do not block pairing/neighbors; prompt shows `role=unknown`.
- [DEFERRED] Distant denial search for conclusions, multilingual tokenization, cross-interview retrieval, per-claim batching of shared shortlists.

## Acceptance Criteria
1. With `LINK_MODE=shortlist` and the ADHD faithful claims plus 8 variant claims, the shortlist (k=8 defaults, no LLM) contains every gold evidence turn from the brainstorm table: v1 {3}, v2 {5}, v4 {12,13}, v5 {17}, v6 {19}, v7 {11}, v8 {25}; [OPTIONAL: v3 {21} and v6 {23}, which rely on the speaker floor / later-correction pull]. Test uses the 03 prep output with `role` removed from every turn.
2. [OPTIONAL] Variant 3: turn 21 (informant) is selected via the speaker floor and the stub linker's contradicting item on it survives validation; variant 6: items cover turns 19 and 23 (with question 22 as context). Core: variant 4: 12 and 13 are both shortlisted and the surviving item has `answer_status="non_answer"` and role `qualifying`.
3. `auto` mode sends all turns below the threshold and the shortlist above it; mode reported on `LinkResult`.
4. Validation unit tests: segment with turn outside shortlist dropped; segment text not in the turn dropped; fuzzy-unmatched dropped; inference with missing premise dropped (premises kept as `qualifying`); `question_only` + `supporting` coerced; Q+"Yes." item snaps so 02 merges it into one cross-turn span; the result never contains a segment spanning two turns.
5. Out-of-scope claims make zero LLM calls; N in-scope claims make N calls (plus at most one retry each) with up to 20 in flight.
6. Failure paths (stub/failing client): timeout, bad JSON twice, all-items-dropped after retry each yield `stalled` with the right reason; other claims still link; `items: []` yields `ok` with no evidence.
7. Whole mode: the prefix (system -> transcript; [OPTIONAL: -> stage with shared 06 layout]) bytes are identical across claims (test); act hints appear only in the claim block; 04 has no built-in warm-up (06 owns it).
8. [OPTIONAL] `shortlist_only` runs without a client. Core: `to_evidence_drafts` output feeds 02's `assemble` in a fixture test and every returned excerpt is located.
9. [OPTIONAL] Leak test: no 8-gram overlap between `link_fewshot.json` and eval fixtures.
10. Sentinel test: no turn/claim text in logs, stall reasons or metrics. Suite runs offline (`LLM_MODE=stub|fixture`).
