---
status: draft
summary: Stage 0 prep (indexed, dialogue-act-tagged transcript with question-answer pairing) and Stage 1 atomizer (per-section claims with slots, verbatim segments, ordinal frequency/severity scales, out_of_scope marking) with validation, one retry, and per-section failure reporting.
date: 2026-10-11
---
# PRD: 03 Prep + Atomize

Repo/branch: iaudit / `feat/backend-poc` · Depends on: 01-scaffold-api · Owns: `backend/src/iaudit_backend/{prep/,atomize/,scales.py}`, `backend/src/iaudit_backend/prompts/atomize_*`, their tests

## TL;DR
Two independent stages. **Prep** (no assessment needed) turns the transcript into `IndexedTurn`s with dialogue-act labels and question-to-answer pairs: rules decide most turns, a cheap-tier LLM sees only the unsure ones. **Atomize** (no transcript needed, except the speaker roster) makes one mid-tier call per section and returns atomic claims: verbatim segments, a self-contained proposition, and slots (subject, reported-by, polarity, ordinal quantity, timeframe, causal links, scope). It also owns the ordinal scales 05 uses in both directions. Stages 04, 05 and 02 consume the output models below; 06 runs prep and atomize concurrently.

## Problem
Later stages need turns that know who asked and who answered (checks 3, 9, 10), and claims small enough to verify one by one with comparable values (checks 3, 4, 5, 7, 12). A free-text claim list cannot be compared in code, and a claim that overlaps another breaks contract rule 8. Conclusions and scoring rules must be set aside, not forced into a factual verdict (mission, contract rule 6). `role` is optional, so nothing here may depend on it.

## Goals / Non-Goals
**Goals**
- Deterministic, testable prep; LLM only for undecided turns.
- Claims that are verbatim-anchored, non-overlapping, and carry the slots 04/05 need.
- One shared ordinal-scale module (`scales.py`).
- Honest degradation: bad chunk or section output never crashes the request and never silently invents data.

**Non-Goals**
- Evidence linking, verdicts, concern synthesis (04, 05); contract span rules and fuzzy matching (02); orchestration, budgets (06).
- Diagnosis, scoring, protocol review. `label`/`score` are not sent to the atomizer and never become claims.
- Role inference, speaker repair, diarization fixes, duration as a scale, embeddings, spaCy.

## Requirements

### R1 Prep: turn index
`IndexedTurn`: `index` (array position; this is the contract span index), `id`, `speaker`, `role: Role | None` (copied, never inferred or propagated from other turns of the same speaker), `text`, `length` (`len(text)`, code points), `acts: frozenset[Act]`, `act_source: rule | llm | rule_fallback`, `responds_to: int | None`. Acts: `question, answer, backchannel, hypothetical, correction`; empty set = none. A turn can carry several (e.g. `answer`+`correction`).

### R2 Prep: deterministic rules
Order: normalise (lowercase, strip punctuation for lexicon match only), then per act yes / no / unsure.

| Act | Yes (rule) | Unsure (goes to LLM) |
|---|---|---|
| question | Sentence ends with `?` (excluding tag tails: `you know?`, `right?`, `y'know?`); sentence opens with interrogative or aux (`who/what/when/where/why/how/do/does/did/is/are/can/could/would/have/has`); elicitation openers (`tell me`, `describe`, `walk me through`) | Mid-sentence `?`, aux opener inside a longer declarative |
| backchannel | Whole turn (≤3 tokens) in the pure-continuer lexicon: `hm, mm, mhm, uh-huh, right, okay, i see, got it, sure` | Polar tokens (`yeah, yes, no, nope, yep`) alone, see R3 |
| correction | Sentence-initial marker: `actually, I mean, no wait, sorry, let me correct, what I meant, scratch that, I misspoke, rather` | Marker mid-sentence (turn 5 "my manager actually says…" is emphasis, not correction) |
| hypothetical | Sentence-initial `suppose, imagine, say, let's say, what if, if` + modal in same sentence | `if`/`would` anywhere else (turn 1 "if it goes past twenty minutes I'm gone" is habitual, not hypothetical) |
| answer | Non-question turn whose previous turn is a `question` from a different `speaker` string (position only; roles not used) | none |

Polar single tokens right after a question from a different speaker are `answer` (a bare "yes" can support a claim, mission). The same tokens elsewhere are `backchannel`. Pure continuers after a question stay `backchannel` and are NOT answers (ADHD turn 13 "Hm.").

### R3 Prep: LLM for undecided turns only
- Only turns with an "unsure" flag go to the LLM (cheap tier, `call_tag="prep_acts"`). Turns with no signal and no question before them are `none` by rule; no LLM call.
- Chunking: windows of `PREP_CHUNK_TURNS` (default 30) undecided-or-context turns with `PREP_CONTEXT_TURNS` (3) read-only context either side; chunks run via `asyncio.gather` (global semaphore in 01's client caps concurrency).
- Output schema: `[{index, acts: [...]}]` for requested indices only. Validate: indices subset of requested, acts in enum, no extras. Invalid chunk: rely on the client's one retry, then fall back to the rule best guess (`act_source=rule_fallback`) and set `PrepResult.degraded_chunks += 1`. Prep never fails a request.
- Zero unsure turns means zero LLM calls (typical for short transcripts).

### R4 Prep: question-answer pairing
For each `question` turn Q, the response set is the following turns whose `speaker` string differs from Q's, until Q's speaker speaks again or the next question from another speaker begins. `QAPair(question_index, response_indices)`; each response sets `responds_to=question_index`. Backchannel responses are included (05 sees they carry no endorsement). Pairing uses `speaker` strings only, so it works with missing roles. A turn that is itself a question and also answers a prior one gets both links. Pairing is deterministic; no LLM.

### R5 Atomizer: input and call
Input per section: `text` only plus a roster `[{speaker, role|None}]` (distinct speakers, no transcript text). One mid-tier call per section (`call_tag="atomize"`, `temperature=0`, `response_model=AtomizerOutput`), all sections via `asyncio.gather`. `cache_prefix` = system prompt + taxonomy + few-shot (stable, cacheable); section text and roster follow. `timeout_s = min(LLM_TIMEOUT_S, remaining deadline)`.

### R6 Atomizer: claim model (`AtomicClaim`)
| Field | Meaning |
|---|---|
| `id` | `c{section}.{n}`, document order |
| `section_index` | array index |
| `segments` | 1–3 verbatim substrings of section text, document order |
| `span_hint` | `(start, end)` code points in the section, set by 03 (R9), consumed by 02 |
| `proposition` | self-contained rewrite for link/check ("Alex does not make careless mistakes at work"); never shown, never copied into the response |
| `scope` / `oos_category` | `in_scope` or `out_of_scope`, category in `diagnostic_conclusion, scoring_rule, recommendation, unobservable, external_criteria`; 03 maps category to a fixed reason sentence in code (the LLM never writes the contract's `out_of_scope_reason`); the sentence is exposed on the claim as `oos_reason: str | None` (set iff `out_of_scope`), which 05 copies into `ClaimDraft.out_of_scope_reason` |
| `subject_text`, `subject_speaker` | subject as written; `subject_speaker` only if it exactly equals a roster `speaker`, else `None` |
| `reported_by` | list (≥1) of `{source: self | informant | interviewer | assessor | unspecified, speaker: str|None}`; `speaker` validated against roster, otherwise `None` (a "partner" is not a speaker id) |
| `polarity` | `affirmed | negated` (denial/absence/exception) |
| `quantities` | 0–2 of `{kind: frequency|severity, raw, bucket, rate_per_week?}` (R8) |
| `timeframe`, `timeframe_explicit` | `current | past | childhood | future | hypothetical | unspecified`; present tense without a marker = `current` with `explicit=false` |
| `hedged` | assessment itself hedges ("estimates", "about", "reports", "may") - input to 05's certainty quick pass |
| `links` (section level) | `{type: "causal", cause_ids, effect_id, connector, source: assessor | reported}`; `reported` when the assessment says someone believes it |

`reported_by` captures sources, so "Alex estimates once a month" becomes its own claim: `reported_by=[{self, "alex1"}]`, `hedged=true`, `quantity: sometimes (0.23/wk)`. "His partner reports… twice a month" is a separate claim with `[{informant, "sam1"}]`. "Both Alex and his partner agree…" is one claim with two `reported_by` entries; 05 compares each source. No claim ever merges differing accounts. Out-of-scope claims still get segments (contract: no unexplained gaps).

### R7 Granularity rules (in prompt and tests)
1. One claim = one checkable fact: one subject, predicate, polarity, timeframe, reporter set. Split on different predicates (`forgets medicines and skips dinner`), subjects, reporters, timeframes, or contrast (`phone daily but has a system for keys`).
2. Qualifiers, conditions, time words, and quantities stay inside the claim they modify (check 8 needs them present in the claim).
3. Do not split a list that shares one value only when items cannot be true separately; if unsure, prefer fewer claims. Cap `ATOMIZE_MAX_CLAIMS` (30) per section.
4. Non-overlap: every code point belongs to at most one claim. Shared lead-in words (subject, verb) belong to the first claim in document order; later claims use only their distinctive words and rely on `proposition`.
5. Non-contiguous claim: emit `segments` (≤3). 03 sets `span_hint` to the minimal covering range when it contains no other claim's segment (`span_quality=covering`), else to the longest segment (`anchor_only`). Splitting into two claims is rejected: it changes verdict granularity. `span_quality` is 03-internal; 02 does not consume it.
6. Conclusions: a statement that derives from other claims ("she is forgetful because…"), a diagnosis/criterion met, a score/label restatement, a threshold rule, or a recommendation is `out_of_scope`; its premises remain claims. Assessor observations of the session ("appeared anxious") are `unobservable`.
7. Worked case (test only, kept out of the prompt): "She is forgetful because she forgets medicines and skips dinner" yields 3 claims: `diagnostic_conclusion` out_of_scope (effect), "forgets medicines" and "skips dinner" in scope (causes), and one causal link `[medicines, dinner] -> forgetful, connector "because"`. 05 decides whether the link attaches a check-12 to the in-scope causes; it is stored here, not decided.

### R8 Ordinal scales (`scales.py`, shared with 05, both directions)
Frequency, 0-5: `0 never`, `1 rarely` (below about monthly, <0.2/wk), `2 sometimes` (monthly up to weekly, 0.2 to <1/wk), `3 often` (1 to <4/wk), `4 most` (4 to <7/wk, "most days", "usually"), `5 always` (daily or more, "every time", "constantly").
Severity, 0-4: `0 none`, `1 mild` (slight, a little, "some difficulty"), `2 moderate` (fair, considerable), `3 severe` (poor, marked, significant, "a lot"), `4 inability` (unable, can't, completely).
- Numeric text normalises to `rate_per_week` (count midpoint over period; "three or four times a week" = 3.5, "twice a month" = 0.46) and then to a bucket in code; code wins over the LLM's bucket when a number is present. Relative words ("most meetings", "every meeting") map by lexicon to the same buckets and keep `rate_per_week=None`.
- Exports: `FrequencyBucket`, `SeverityBucket` (IntEnum), `bucket_from_rate()`, `lexicon_bucket(text, kind)`, `step_diff(claim, evidence)` (positive = claim stronger; negative = claim weaker). The materiality threshold and "inflates"/"understates" wording are 05's; suggested default one step, with a one-step band when one side is numeric and the other relative.
- Known limit: numeric and relative values for one answer can disagree (ADHD turn 3 "Most meetings. So three or four times a week" = 4 vs 3). 05 should prefer like-for-like and treat the mixed case as tie within one step.

### R9 Validation, retry, failure (deterministic, after parse)
1. **Resolve segments** to code-point offsets in the section: exact first, then whitespace/quote-normalised with an offset map (NFKC never alters offsets); multiple hits resolve by scanning claims in output order and taking the earliest hit that does not overlap an assigned region. 03 does not implement its own matcher: it calls 02's `locate()` (exact, normalized, guarded fuzzy; `claim_fuzzy_threshold`) for each segment within the section, then computes `span_hint` (covering span per R7.4/R7.5). A segment `locate()` cannot find is flagged `unresolved` (counts as the unresolved-segment error in step 3). 02 re-verifies `span_hint` and re-runs `locate()` if absent or invalid.
2. Check: unique ids; link ids exist; `oos_category` iff `out_of_scope`; roster-valid speakers (invalid becomes `None`, repaired); no overlap between claims (R7.4); claim count within cap; no sentence of ≥5 words left with no claim (coverage gap).
3. **One retry** per section on any non-repairable error (unresolved segment, overlap, bad link, uncovered sentence, empty output for non-trivial text): resend with the previous output and a terse error list (claim ids and error kinds only, no text echo). The client's transport/JSON retry is separate (01 R6). The retry is skipped if the remaining budget is under one call timeout.
4. After retry, still-invalid claims are dropped (counted), except overlap where the later claim is dropped. Remaining coverage gaps are counted, not fabricated.
5. **Section failure** (`SectionAtoms.status="failed"`, reason in `timeout | unavailable | bad_output | no_claims | too_many_dropped`) when: client raises `LLMTimeout/LLMUnavailable/LLMBadOutput`; no valid claims for non-trivial text; or >50% of claims dropped. Result carries `section_index`; 06 maps it to `section_not_processed` + `partial` (locked decision). `atomize_sections` never raises for per-section failures.

### R10 Prompt design
System prompt: task, scope rules, taxonomy (R6/R7/R8 definitions), output JSON schema (Pydantic `AtomizerOutput`, `strict`), rule "segments must be copied character-for-character, never paraphrased", rule "never name a speaker not in the roster; never infer roles". Few-shot: 4 examples in `prompts/atomize_fewshot.json`, each section text plus gold output, covering (a) conjunction split + because link + conclusion, (b) informant vs self report with estimate and shared lead-in words, (c) negation + childhood timeframe + numeric rate, (d) scoring rule + recommendation out_of_scope + non-contiguous claim. All use a fictional person ("Jordan") and invented behaviours different from the approved ADHD example and its 8 variants. The "forgetful because… medicines and skips dinner" sentence and anything from the CDR case are NOT used in the prompt.
**Leak guard:** a CI test fails if any 8-word window from the few-shot texts appears in any eval fixture/gold file (`backend/tests/data`, 07's fixture dir). Few-shot file carries `split: "prompt"`; 07 must exclude it from eval sets.
Prep LLM prompt: act definitions plus 2 inline examples drawn from the rule table (not from the ADHD set). Tier names only, never model ids.

### R11 Config (additive to 01's `Settings`)
`PREP_CHUNK_TURNS` 30, `PREP_CONTEXT_TURNS` 3, `ATOMIZE_MAX_CLAIMS` 30, `ATOMIZE_MAX_TOKENS` 4000, `ATOMIZE_RETRY` true. Stored in `Settings` (small edit to 01's `config.py`, coordinated).

### R12 Logging
Counts only: turns, undecided turns, chunks, degraded chunks, claims per section, dropped, gaps, retries, failures by reason, token usage via `MetricsCollector`. No turn text, section text, segments, or propositions (01 R9 allowlist).

## Architecture
```
turns -> rules -> [unsure] -> chunks (cheap LLM, parallel) -> merge -> pairing -> PrepResult
sections+roster -> per-section mid call (parallel) -> parse -> resolve/validate -> (retry once) -> SectionAtoms[]
```
Public surface:
```python
async def prep(turns: list[Turn], ctx: RequestContext) -> PrepResult      # turns, qa_pairs, degraded_chunks
async def atomize_section(section: Section, roster: list[Speaker], ctx) -> SectionAtoms   # single unit, called per section by 06 (per 06 R8)
async def atomize_sections(sections: list[Section], turns: list[Turn], ctx) -> list[SectionAtoms]   # now a thin gather wrapper over atomize_section
def prep_rules_only(turns: list[Turn]) -> PrepResult   # no LLM; 06 uses it on prep soft timeout (per 06 R3/R8)
# SectionAtoms: section_index, status, claims: list[AtomicClaim], links, dropped, coverage_gaps, failure|None
```
Uses 01's `LLMClient.complete_json(LLMRequest)`, `LLMResult`, `Settings`, `RequestContext` (deadline, metrics), tiers `cheap/mid`, `StubClient`/`FixtureClient` for tests, `Role`/`Turn`/`Section` models. Assumption for 02: 02's `ClaimDraft` uses these same names: `segments`, `span_hint`, `scope`, `oos_category` (fixed reason string). 02 re-verifies the hint and falls back to `locate()` [RESOLVED O-1].

## Decisions
| # | Decision | Choice | Alternatives considered | Why |
|---|---|---|---|---|
| D1 | Prep and atomize | Independent, run concurrently | Atomizer sees dialogue acts | Saves latency; claims come from assessment text only |
| D2 | LLM use in prep | Only unsure turns | Every turn per chunk | Fewer tokens; rules are cheap and testable |
| D3 | Act granularity | Turn-level multi-label | Sentence spans | Enough for 04/05; contract evidence is turn/char anyway |
| D4 | Pairing basis | `speaker` string, not role | Role-based | Roles optional; never inferred |
| D5 | Polar "yes" | Answer after other speaker's question, else backchannel | Always backchannel | Mission: short "yes" can support a claim |
| D6 | Claim text | Verbatim `segments` plus separate `proposition` | One rewritten text | Highlight needs real substrings; checking needs self-contained text |
| D7 | Offsets | 03 resolves exact, 02 fuzzy fallback | LLM emits offsets; all in 02 | LLMs miscount offsets; exact matching is cheap here |
| D8 | Out-of-scope reason | Category from LLM, sentence from code | LLM writes reason | Contract rule 9 (no copied text), fixed neutral wording |
| D9 | Reported-by | List of `{source, speaker}`; separate claim per account | Single reporter | Check 11 and "Alex estimates" need the accounts kept apart |
| D10 | Scales | Two ordinal scales, numeric -> bucket in code | LLM scores 1-10; one shared scale | Comparable both directions; code beats LLM on numbers |
| D11 | Few-shot data | Fictional "Jordan" set, 8-gram leak test | Reuse approved ADHD sentences | Keeps acceptance set honest |
| D12 | Section failure | Fail, do not guess | Fall back to sentence-split claims | Locked decision; unreliable claims erode trust |
| D13 | Label/score | Not sent to the atomizer | Send as context | Never audited; avoids spurious claims |

## Risks / Open Questions
- [RESOLVED: 02 `locate()` is the single locator, called by 03 per segment; `ClaimDraft` adopts `segments` + `span_hint`; 02 re-verifies the hint and falls back to `locate()`; field names are 03's] O-1 Align output model with 02.
- [OPEN] O-2 Corrections without a marker (ADHD turn 23 "I think it's less than that") are tagged `answer`, not `correction`; relies on 04/05 and check 11. Measure recall in eval.
- [OPEN] O-3 Is every assessor observation `unobservable`? Borderline: "he seemed distracted" vs report text. Benchmark with reviewers.
- [OPEN] O-4 Whether to surface coverage gaps / dropped claims as an `other` warning; default is logs only.
- [OPEN] O-5 Scale bucket boundaries (4/wk, monthly) and mixed numeric/relative handling are judgment calls; tune on dev split.
- [OPEN] O-6 Retry fits the 5-10 s atomize budget only on fast tiers; verify with live latency.
- [DEFERRED] Duration, certainty beyond `hedged`, other link types (contrast, parallel accounts), sentence-level acts, cross-speaker correction detection by rules.

## Acceptance Criteria
1. Rule tests: ADHD turn 1 not hypothetical, turn 5 not correction, turn 11 correction, turn 13 `backchannel` (not answer), turn 3 `answer`; `?` tag tails not questions; polar token rules. (Used as behavior checks on the test fixture only, never in the prompt.)
2. A transcript with zero unsure turns makes zero LLM calls; unsure turns produce one call per chunk, parallel; an invalid chunk falls back to rules with `degraded_chunks` set.
3. Pairing: ADHD turns (0->1; 2->3; 12->13 including backchannel) pair correctly with missing `role` on every turn; `role` stays `None`.
4. Atomizer with stub/fixture client: "She is forgetful because she forgets medicines and skips dinner" yields 3 claims, the conclusion out_of_scope, and one causal link; Alex/Sam sentence yields separate informant and self claims, the latter `hedged`.
5. All returned segments are exact substrings (or flagged `unresolved`); no two claims overlap; `span_hint` falls inside the section; non-contiguous claims set `span_quality`.
6. Numeric normalisation: "three or four times a week" -> 3.5/wk -> `often`; "twice a month" -> `sometimes`; "every day" -> `always`; "some difficulty" vs "poor" -> step_diff -2; unit tests cover both directions.
7. Section with a timeout, bad JSON twice, or only unresolvable claims returns `status=failed` with a reason; other sections still return; one invalid-then-valid reply succeeds via exactly one retry.
8. Leak test passes: no 8-gram overlap between `atomize_fewshot.json` and any eval fixture.
9. No turn/section text in logs or in failure reasons (sentinel test).
10. Whole suite runs offline (`LLM_MODE=stub`/`fixture`).
