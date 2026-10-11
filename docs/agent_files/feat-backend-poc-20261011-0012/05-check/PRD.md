---
status: draft
summary: Stage 3 Check - group A code checks (attribution, question-as-answer, missing-role handling), one mid-tier verifier call per claim (meaning, amounts, P1 flags, inference premises), and a pure-code verdict/concern synthesis with templated neutral explanations, producing 02's ClaimDraft.
date: 2026-10-11
---
# PRD: 05 Check

Repo/branch: iaudit / `feat/backend-poc` · Depends on: 03-prep-atomize (`AtomicClaim`, `IndexedTurn`, `scales.py`), 04-link (`LinkResult`, `LinkedEvidence`), 02-assembler (`ClaimDraft`, `ConcernDraft`, `EvidenceDraft`, `locate`) · Owns: `backend/src/iaudit_backend/check/` (`models.py`, `group_a.py`, `amounts.py`, `verifier.py`, `synth.py`, `templates.py`, `checker.py`), `prompts/check_system.md`, `prompts/check_fewshot.json`, tests

## TL;DR
For each linked claim, decide verdict, concerns (each with evidence keys), explanation and optional `suggested_revision`, as 02 `ClaimDraft`s. The LLM only **observes** (per-evidence relation, polarity, amount, timeframe, certainty, experiencer, P1 flags with confidence); **code decides** (polarity compare, ordinal compare in both directions, attribution, question-as-answer, inference rule, verdict table). Explanations are code templates, so wording stays neutral and never copies the transcript. Claims with no verifiable evidence, or only question/non-answer evidence, make zero LLM calls. A failed check stalls the claim to 06 (locked).

## Problem
Link output says what evidence exists; nothing yet says whether it faithfully supports the claim. Mission rules: missing evidence is not contradiction; a bare question is not an answer; unclear speaker identity gives a narrower finding; wording stays "not substantiated by the supplied interview" (contract rule 12). LLM judgement of whole verdicts is hard to test and drifts; per-evidence observations plus a deterministic table are testable per bucket and keep contract rules 2-6 true by construction. `role` is optional, so role-dependent checks must degrade, not guess.

## Goals / Non-Goals
**Goals**
- Buckets 1, 2, 3, 5, 7, 9 done well (P0); 4 as stretch; 6, 8, 10, 11, 12 as a confidence-gated quick pass.
- Understated and inflated amounts both flagged, direction in the explanation.
- Output that passes 02's rule checks without repair on the normal path.
- Each ADHD variant maps to an explicit signal path (see Variant map).

**Non-Goals**
- Linking, offsets, atomizing, assembly (04, 02, 03). Diagnosis, scoring, protocol review, real-world truth (mission).
- LLM-written explanations; contract changes or concern renames; self-consistency voting; NLI/spaCy/embeddings.
- Counterevidence search for `out_of_scope` claims (deferred in brainstorm); they pass through.

## Requirements

### R1 Entry point and I/O
```python
async def check_claims(claims: list[AtomicClaim], links: list[LinkResult], prep: PrepResult,
                       turns: list[IndexedTurn], ctx) -> list[ClaimDraft]   # one per claim, input order
async def check_one(claim: AtomicClaim, link: LinkResult | None, prep: PrepResult, turns: list[IndexedTurn], ctx) -> ClaimDraft   # single unit, called per claim by 06 (per 06 R8); stage semaphore (CHECK_CONCURRENCY) per request
def stalled_draft(claim: AtomicClaim, reason: str) -> ClaimDraft   # outcome="stalled"; 06 uses it for deadline/exception stalls
def passthrough_oos(claim: AtomicClaim) -> ClaimDraft   # out_of_scope draft, no call
def decide(claim, judged: list[JudgedEvidence], signals: Signals, cfg) -> Decision   # pure; 07 may call it
```
`ClaimDraft` fields come from 02 (`key=claim.id`, `section_index`, `segments`, `span_hint` copied from `AtomicClaim`; `outcome`, `verdict`, `explanation`, `out_of_scope_reason`, `suggested_revision`, `concerns`, `evidence`, `ambiguous_turns`). Out-of-scope claims are passed through here (no call; reason is 03's fixed sentence) so 06 only collects drafts. `stalled` link results become `outcome="stalled"` drafts. `evidence` = 04 `LinkedEvidence.draft` with `role` rewritten by R5; inference items are not emitted (premises are). Keys stay `"{claim.id}:e{n}"`; 02 assigns final ids.

### R2 Group A: code checks (no LLM except the experiencer case)
Inputs: `AtomicClaim.reported_by`, `subject_speaker`, `IndexedTurn.speaker/role`, 04 `answer_status`, 04's located excerpts (**quote existence**: 04 already dry-runs `locate()` and drops misses; 02 re-resolves and repairs. 05 does not re-match).
- **Missing-role rule (locked).** A turn with `role is None` is excluded from every role-dependent check below (its speaker string is treated as unreliable, e.g. a diarization merge). The turn stays in `ambiguous_turns`; the check is skipped, not guessed. 02 turns `ambiguous_turns` into `ambiguous_speaker` (names indexes only).
- **A3 reporter attribution.** For each `reported_by` entry with `speaker` set: evidence items count for that account only if the turn `speaker` string equals it (roles must be non-None). For entries with source `informant`/`self` and `speaker=None`, compare the turn `role` (`informant`/`respondent`); role `other` or `None` = skip. Source `assessor`/`unspecified`/`interviewer`: no reporter check. If a claim names reporters and entailing evidence exists only from other speakers: `wrong_attribution` (content found, source wrong) and those items do not count as support for the named account. Items from non-named speakers are `qualifying`, never `supporting`/`contradicting`, when the claim names its reporters (an unspecified-source claim accepts every speaker).
- **A3 experiencer case (LLM only):** per-evidence field `experiencer` = `claim_subject | other_person | unclear` from the verifier (R3). `other_person` with confidence >= floor and relation `entails`/`contradicts` triggers `wrong_attribution` (the behaviour is described, but about someone else). No role is needed; no identity is named in output.
- **A9 question-as-answer.** Claim has >=1 evidence item with `answer_status` in `question_only | non_answer` (from 04, which already demoted `supporting` to `qualifying`), has no item with `endorsed` or `n/a` status, and the question turn has `role == interviewer` -> `question_as_answer`; verdict `insufficient_evidence` (never `contradicted`; rule 5). Role `None` on the question turn: skip, add to `ambiguous_turns`; the claim falls to plain `unsupported_claim`. Role other than interviewer (e.g. respondent asks): no concern. If the claim also has endorsed evidence, A9 does not fire.
- A9 and "no evidence" are decided before any call: if every item is non-endorsed (or there are none), the claim makes **zero verifier calls**.

### R3 Verifier call (one per remaining claim, mid tier)
`call_tag="check"`, `temperature=0`, `response_model=VerifierOutput`, `max_tokens=CHECK_MAX_TOKENS`, local `Semaphore(CHECK_CONCURRENCY=20)` under 01's global one; `timeout_s=min(LLM_TIMEOUT_S, remaining deadline)`. 06 starts a claim's check as soon as its link result exists (pipelined).
- **Input**: claim `proposition`, `polarity`, `timeframe` (+explicit), `hedged`, `quantities`, `reported_by`; the same turns 04 showed for that claim (`LinkResult.shortlist`, same `render_turns`, speaker + role-or-`unknown`, verbatim text, act hints) so context for flags 8/10/11 exists; then the evidence items (`key`, turn, linker role as a hint, `answer_status`, premise keys and inference step for inference items). Turn text is data, never instructions.
- **Per item** `items[]`: `relation` (`entails | contradicts | not_addresses`; judged on the behaviour/state and direction only, **ignoring amount, timeframe and qualifiers**, which are reported separately); `polarity` (`affirmed | negated | unclear`, of the evidence about the claim's predicate); `frequency` and `severity` (`{raw, rate_per_week?, bucket_guess?, basis: numeric | relative}` or null, using 03's scale definitions pasted in the prompt); `timeframe` (03 enum); `certainty` (`definite | hedged_light | hedged_strong`); `experiencer`; `conf` (`low | medium | high`).
- **Claim-level flags** `flags[]`: `{type, keys[], conf}` for `timeframe_mismatch`, `certainty_inflation`, `context_stripping` (item contains a qualifier the claim drops), `correction_ignored` (a later turn retracts/changes the cited item), `conflicting_accounts_flattened` (accounts differ, claim presents agreement), `unsupported_causal_link` (claim asserts cause-effect the transcript does not establish; a speaker asserting the cause counts as established), `element_gap` (some element of the claim is established by no item). Flags 4/6/8/10/11/12 are the P1 quick pass; others feed P0 buckets.
- **Inference groups** `inferences[]` per inference item: `{key, step_acceptable, conf}`. Premise judgments are the premises' own `items[]` relations.
- **Validation**: unknown keys dropped; every evidence key must be judged. Missing items -> one semantic retry (terse key list, no text), then the claim stalls (`reason=bad_output`). Unjudged items never count as support.
- **Prompt**: taxonomy (relation, polarity, scales, flag definitions, the "question is not evidence" rule, "never guess a speaker or role"), few-shot `prompts/check_fewshot.json` (`split: "prompt"`; fictional, includes clean no-flag cases to curb over-flagging and a vague-quantifier case). Same 8-gram leak CI test as 03 R10 / 07 R11; no ADHD or CDR text. Tier names only.
- **Cache [RESOLVED: per 06 R6]**: prompts are built with 06's `pipeline/layout.py` (common system -> transcript -> stage -> claim), shared with 04; 05 no longer renders its own prefix. `cache_prefixes` = [common system + transcript (whole mode, byte-identical to 04's, no act hints)], [check stage block: instructions + few-shot]; claim block last (act hints, claim, evidence items). Shortlist mode: prefix = common system + check stage block, and the claim block reuses the exact shortlist rendering 04 used.

### R4 Code synthesis signals (`synth.py`, `amounts.py`)
1. **Confidence floor.** `conf` maps low/medium/high = 0.3/0.6/0.9. P0 observations (relation, experiencer, polarity) need >= `CHECK_MIN_CONF_P0` (default 0.5); P1 flags need >= `CHECK_MIN_CONF_P1` (default 0.6, i.e. medium). Below floor: dropped (counted, logged). Eval-driven suppression tuning is deferred (O-5).
2. **Polarity.** Claim `polarity` vs item `polarity` on items with relation `contradicts` and `experiencer = claim_subject`: both known and different -> `negation_reversal`.
3. **Ordinal amounts, both directions.** Per kind (frequency, severity), claim `quantities` (03 buckets) vs verifier value mapped in code (`rate_per_week` -> `bucket_from_rate`, else `lexicon_bucket(raw)`, else `bucket_guess` flagged `basis=model`). Only `entails` items from the claim's accounts are compared. Pairs are same-basis first (numeric-numeric, relative-relative); take the pair with the smallest `|d|` (benefit of the doubt), `d = step_diff(claim, evidence)`, positive = claim stronger. Mixed-basis only if no same-basis pair.
   - **Material** iff `|d| >= CHECK_STEP_MATERIAL` (default 1) and not *adjacent-ambiguous*; adjacent-ambiguous = `|d| == 1` and (basis is mixed or the evidence rate sits within 15 percent of the bucket boundary). [OPEN O-1]
   - **At-least cues.** If the claim `raw` carries a lower-bound cue (repeatedly, frequently, regularly, often, several, many, multiple) and `d < 0`, it is not an understatement (vague quantifiers are compatible with a higher rate). Small list in `amounts.py`. "Some", "mild", "a little" are level words, not at-least cues.
   - **Stronger (d > 0):** concern `frequency_severity_inflation`, verdict `partially_supported`, explanation says "inflates". Escalates to `contradicted` when evidence bucket is `0` (never/none) and claim >= 1, or `d >= CHECK_CONTRADICT_GAP` (default 3); the compared item is then relabelled `contradicting` so rule 3 holds.
   - **Weaker (d < 0):** same concern type, `partially_supported`, explanation says "understates", never `contradicted`.
   - **Tie / immaterial / no same-kind value on either side:** no concern.
4. **Timeframe (bucket 4, stretch).** Claim `timeframe` vs item `timeframe` (or a flag). Mismatch = claim current/future vs item past/childhood/hypothetical, claim childhood/past vs item current, or any claim vs hypothetical. Implicit-current claims (`timeframe_explicit=false`) fire only at `conf = high`. Mismatched items are **demoted** to `qualifying` (they no longer count as support); the concern is emitted only if no supporting item remains (R6 row D6).
5. **Certainty (6).** Fires only for claim `hedged=false` vs item `hedged_strong`; `hedged_light` ("probably every day") is ignored.
6. **Conflicting accounts (11).** Fires on the verifier flag, or in code when the claim has >=2 `reported_by` accounts and numeric `rate_per_week` of different accounts differ by ratio >= `CHECK_RATE_RATIO` (default 1.5). Needed because "twice a month" (0.46/wk) and "once a month" (0.23/wk) fall in one ordinal bucket, so amounts alone cannot see it.
7. **Causal (12).** Considered only when the claim has a causal cue (lexicon: because, caused, due to, led to, resulted in, so, therefore; or an assessor-sourced 03 `link` naming it); otherwise the flag is ignored. Reported/believed links (`source=reported`) are not flagged.
8. **Correction (10), context (8), element gap:** verifier flags above the floor. Items cited by an accepted 8/10/12 flag count as partial support (they are the source the claim drew from), so a flagged claim lands on `partially_supported`, not `insufficient_evidence` [OPEN O-9].
9. **Inference rule.** An inference group is *supported* iff every premise item has relation `entails` and `step_acceptable` at/above the floor. A failed group: premises relabelled `qualifying`, and they supply no support. Supported group: premises are `supporting` evidence like any other.

### R5 Evidence roles out
`entails` + endorsed/n-a + not demoted -> `supporting`. `contradicts` -> `contradicting`. Demoted, non-endorsed, failed-inference premises, or items cited by a concern but not entailing -> `qualifying`. `not_addresses` items cited by nothing are dropped (counted). The verifier relation wins over the linker role (counted as disagreement), but a 04 coercion to `qualifying` is never upgraded. Every concern lists the keys of the items it rests on; `unsupported_claim` may list none.

### R6 Verdict decision table (first matching row)
| Row | Signals | Verdict | Concerns |
|---|---|---|---|
| D0 | `scope=out_of_scope` | `out_of_scope` | none (03's reason) |
| D1 | link stalled / verifier stalled | `outcome=stalled` (02 builds `insufficient_evidence` + `other` warning naming the claim) | none |
| D2 | no evidence items | `insufficient_evidence` | `unsupported_claim` (no call) |
| D3 | A9 fires | `insufficient_evidence` | `question_as_answer` on the question/non-answer item(s) (no call) |
| D4 | contradicting >= 1, supporting = 0 | `contradicted` | most specific of: `wrong_attribution` > `negation_reversal` > `frequency_severity_inflation` (zero/gap escalation) > `contradictory_evidence` |
| D5 | contradicting >= 1, supporting >= 1 | `partially_supported` (never `supported`; matches 02 R4) | `conflicting_accounts_flattened` > `correction_ignored` > `contradictory_evidence`, plus any independent concern |
| D6 | no support, no contradiction, demoted items exist | `insufficient_evidence` | `timeframe_mismatch` (timeframe demotion) or `unsupported_claim` (failed inference) |
| D7 | no support, nothing else | `insufficient_evidence` | `unsupported_claim` |
| D8 | supporting >= 1, no contradicting, >= 1 surviving concern | `partially_supported` | reporter `wrong_attribution`, `frequency_severity_inflation` (inflates/understates), `certainty_inflation`, `context_stripping`, `unsupported_causal_link`, `conflicting_accounts_flattened`, `unsupported_claim` for `element_gap` |
| D9 | supporting >= 1, no contradicting, no concern | `supported` | none |
**Contract guards** (property-tested): `supported` iff supporting >= 1 and contradicting = 0 and zero concerns; `contradicted` has a contradicting item; `partially_supported` has >= 1 concern; `insufficient_evidence` has no contradicting item; `out_of_scope` has no concerns or evidence.

**Dedup ("explain once").** A specific concern suppresses a more generic one on the same item(s): `question_as_answer` suppresses all others; `wrong_attribution` > `negation_reversal` > `frequency_severity_inflation` > `contradictory_evidence`; `conflicting_accounts_flattened` suppresses amount compare across differing accounts and `contradictory_evidence`; any specific concern suppresses `unsupported_claim` and `element_gap`. Independent aspects coexist (e.g. certainty plus inflation). One concern per type per claim; evidence keys unioned.

### R7 Explanations and suggested_revision
- Templates in `templates.py`, keyed by (verdict) and (concern type, direction). Slots are code-derived only: scale names from `scales.py`, source nouns from supplied `role` ("the respondent", "the informant", else "another speaker"), turn count. No transcript text, no timestamp (rule 9), no speaker ids, no identity guesses. Claim `explanation` = one verdict sentence plus the concern type names; the detail lives once in each concern `explanation`.
- Vocabulary is fixed neutral: "is not substantiated by the supplied interview", "may differ", "a reviewer may wish to confirm". Banned words (test): fabricat*, lie/lying, false claim, dishonest, misconduct, deliberately, misrepresent*. Inflation text contains "inflates", understatement text contains "understates" (07 gate).
- `suggested_revision`: optional single sentence from the verifier (`CHECK_SUGGEST_REVISION`, default true) only for `partially_supported`/`contradicted` with an amount, polarity or qualifier fix; null otherwise. Code rejects if it contains a timestamp or is > 40 words; 02's copycheck nulls transcript copies. Never applied by anyone (rule 12).

### R8 Failure, logging, config
- Verifier timeout/unavailable/bad output after the client retry and the semantic retry -> that claim only is `stalled` (`reason` in `timeout | unavailable | bad_output`). `check_claims` never raises per-claim; `LLMUnavailable` on every claim is surfaced to 06 (503 decision).
- Logs: counts only (calls made/skipped, flags seen / below floor / suppressed by type, verdicts, stalls by reason, role-skipped checks, tokens in/out/cache_read). Never claim, turn or excerpt text (01 R9 allowlist).
- Config additive to 01 `Settings`: `CHECK_CONCURRENCY` 20, `CHECK_MAX_TOKENS` 1500, `CHECK_MIN_CONF_P0` 0.5, `CHECK_MIN_CONF_P1` 0.6, `CHECK_STEP_MATERIAL` 1, `CHECK_CONTRADICT_GAP` 3, `CHECK_RATE_RATIO` 1.5, `CHECK_SUGGEST_REVISION` true, `CHECK_RETRY` true.

## Architecture
```
LinkResult + AtomicClaim + IndexedTurns
  -> group A (code): reporter attribution, A9, role-skips, no-evidence short-circuit
  -> [needs call?] verifier (mid, parallel, cache prefix) -> VerifierOutput
  -> synth (code): floors, polarity, ordinal both ways, timeframe, certainty, 11/12/8/10, inference, roles
  -> decision table + dedup -> templates -> ClaimDraft  --> 06 --> 02 assemble
```
Uses 01 `LLMClient`, `RequestContext`, `Settings`, tier `mid`, Stub/FixtureClient; 03 `scales.py` (`bucket_from_rate`, `lexicon_bucket`, `step_diff`), `AtomicClaim`, `IndexedTurn`; 04 `LinkResult.evidence/shortlist`, `render_turns`; 02 draft models. 07 Gold mode builds `LinkResult`s from gold evidence, so the verifier still runs there.

## Variant map (ADHD example; expected results from 07 R3)
| V | Linker gives (04) | Signal path in 05 | Result |
|---|---|---|---|
| Faithful | entailing items per claim | relation `entails`, values equal or immaterial, no flags (S5 "repeatedly" vs "every report card": at-least cue, no understatement; S2 "probably every day" = `hedged_light`; S3 "Sometimes" has no severity value) | D9 `supported`, no concerns |
| 1 | turn 3 (relative "Most meetings" 4, numeric 3.5/wk = 3) | claim relative `always`(5) vs 4 gives d=+1 same-basis, numeric 5 vs 3 gives +2; min d=+1, not ambiguous, material | D8 `partially_supported` + `frequency_severity_inflation`, "inflates"; turn 3 `supporting` |
| 2 | turn 5 "Not really" | relation `contradicts`, evidence `negated` vs claim `affirmed`, experiencer = subject -> `negation_reversal` (suppresses generic `contradictory_evidence`) | D4 `contradicted` + `negation_reversal` |
| 3 | turn 21 (sam1), `contradicting` | `contradicts`, `experiencer=other_person` (Sam is the forgetful one) -> `wrong_attribution`, which suppresses `negation_reversal` ("He's good with appointments" is also negated) | D4 `contradicted` + `wrong_attribution` |
| 4 | question 12 + "Hm." 13 as one `qualifying` item, `answer_status=non_answer` | A9: no endorsed item, question turn role `interviewer` -> `question_as_answer`; zero verifier calls | D3 `insufficient_evidence` + `question_as_answer`; evidence `qualifying` (12-13) |
| 5 | turn 17, `qualifying` | item `entails` the behaviour, `timeframe=childhood/past` vs claim current explicit -> mismatch, item demoted, no support left (or claim flag `timeframe_mismatch`) | D6 `insufficient_evidence` + `timeframe_mismatch` |
| 6 | 19 (supporting), 23 (contradicting or qualifying) | claim has 2 `reported_by` accounts; Sam 0.46/wk vs Alex 0.23/wk: ratio 2.0 >= 1.5 (+ verifier flag 11); same ordinal bucket so no amount concern | D8 (or D5 if 23 is `contradicting`) `partially_supported` + `conflicting_accounts_flattened` |
| 7 | turn 11 | `entails` with flag `context_stripping` citing 11 (qualifier "ran out of paint" dropped); cited item counts as partial support | D8 `partially_supported` + `context_stripping` |
| 8 | turn 25 | causal cue "caused"; flag `unsupported_causal_link` citing 25; cited item counts as partial support | D8 `partially_supported` + `unsupported_causal_link` |

Impact of 07 open items:
- **07 O-1 (V2 may also carry `contradictory_evidence`):** 05 emits only the specific concern by dedup, so V2 yields `negation_reversal` alone. Keeping `contradictory_evidence` in `concerns_allowed` is harmless and recommended as a safety net; the ADHD set has no isolated generic-contradiction case, so bucket 2 recall is unmeasured until one is added (O-13).
- **07 O-2 (V4 evidence role):** 05 emits the 12-13 pair as one `qualifying` item (04 snapped, 02 merges), so either "12 qualifying, 13 optional" or "empty passes" works; recommend resolving 07 O-2 as `qualifying` required-optional. V4 only reaches `question_as_answer` if 04 returns the pair and roles are present; with roles stripped the result is `insufficient_evidence` + `unsupported_claim` + `ambiguous_speaker` (07 should add a role-less V4 expectation).

## Decisions
| # | Decision | Choice | Alternatives considered | Why |
|---|---|---|---|---|
| D1 | Who decides | LLM observes, code decides verdict | LLM returns verdict | Testable per bucket; contract rules 2-6 hold by construction |
| D2 | Relation scope | Behaviour and direction only; amount, timeframe, qualifier reported separately | One holistic "supported?" | Lets code compare amounts both ways and attribute each concern |
| D3 | Confidence | `low/medium/high` enum with floors | Numeric self-scores | Verbalized numbers are poorly calibrated; enum is easy to floor |
| D4 | Explanations | Code templates | LLM prose | Neutral wording, no copying (rule 9), no accusations (rule 12) |
| D5 | Missing role | Skip role-dependent checks per turn; warn | Guess from speaker string | Locked decision; merged turns have unreliable speakers |
| D6 | Zero-call paths | No evidence, or only question/non-answer evidence | Always call | Saves latency/cost; deterministic |
| D7 | Understated | Same concern type, `partially_supported`, "understates"; never `contradicted` | New type | Locked brainstorm decision |
| D8 | Mixed support/contradiction | `partially_supported` + specific concern | `contradicted` | Contradiction of part only; matches 02 R4 |
| D9 | Timeframe | Demote mismatched evidence, concern only if no support left | Concern alongside `supported` | Evidence from another period does not substantiate the claim |
| D10 | Verifier context | Same turns 04 showed | Cited items only | Flags 8/10/11 need surrounding turns; cache reuse |
| D11 | Generic vs specific concern | Specific suppresses generic | Emit all | Contract wants no repetitive findings; avoids FP on gold |
| D12 | Quote existence | Rely on 04 dry-run + 02 | Re-match in 05 | One matcher (02 `locate`) |

## Manual steps (human-only)
- Verifier benchmark and few-shot validation need a provider API key (owner step in 01); thresholds in O-1/O-5 are set from live runs on the ADHD set only.

## Risks / Open Questions
- [OPEN] O-1 Materiality: `|d| >= 1`, adjacent-ambiguous band (mixed basis, 15 percent of boundary), `CHECK_CONTRADICT_GAP=3`, rate ratio 1.5 are guesses. Tune per bucket on dev; report false-alarm rate on faithful claims.
- [OPEN] O-2 At-least cue list and treatment of vague quantifiers (S5 faithful depends on it); 03 could instead expose `bound: exact|at_least` on `Quantity`.
- [OPEN] O-3 Claims with no quantity of a kind never get an amount check, even if the evidence is extreme; depends on 03 extracting quantities.
- [OPEN] O-4 Faithful sensitivity: strict verifiers may flag S7 ("Not really" -> "denies"), S3 ("Sometimes" -> "some difficulty"), S2 ("probably every day"); user-approved text, so failures are checker findings (07 O-12).
- [OPEN] O-5 Confidence floors (0.5 / 0.6) and enum mapping are proposals; P1 precision-based suppression (<0.6) is deferred to eval on the dev split.
- [RESOLVED: per 06] O-6 Shared cache prefix with 04: common system + transcript first, stage block after (whole mode only; shortlist mode per-stage). Live cache_read numbers still depend on 01 O-2.
- [OPEN] O-7 A9 and A3 skip entirely when roles are missing even though 04's `answer_status` uses speaker strings; locked rule kept. Relax only on user decision; effect today is a safe `insufficient_evidence`.
- [OPEN] O-8 V3: turn 21 mixes "Sam forgets" and "Alex is good"; precedence `wrong_attribution` over `negation_reversal` is a rule, not measured.
- [OPEN] O-9 "Flag-cited items count as partial support" (V7, V8) assumes 04 returns those turns; if the verifier calls them `not_addresses` without the flag firing, the claim becomes `insufficient_evidence` instead.
- [OPEN] O-10 Verifier sees 04's shortlist, not the whole transcript, in shortlist mode; flag 10/11 recall is bounded by 04's shortlist.
- [OPEN] O-11 `suggested_revision` quality and default-on; copycheck (02 R5) may null many.
- [OPEN] O-12 `ClaimDraft` for out-of-scope claims is built here; needs 03 to expose the fixed reason string on `AtomicClaim` (assumed `claim.oos_reason`).
- [OPEN] O-13 No isolated bucket-2 (generic contradiction) or bucket-10 negative in the ADHD set (07 O-8); need user-approved variants before those buckets are measurable.
- [RESOLVED: 05 never emits `supported` with a contradicting item; mixed case = `partially_supported` + concern, same as 02 repair] 02 O-4.
- [RESOLVED: `qualifying` is the role for non-answer pairs; rule 5 only forbids `contradicting` under `insufficient_evidence`] 04 O-6.
- [DEFERRED] Per-account value comparison beyond speaker/role match; self-consistency; contract rename to `frequency_severity_mismatch`; counterevidence for out-of-scope claims; LLM rewriting of explanations.

## Acceptance Criteria
1. Variant map: with a stub verifier returning the observations in the Variant map (no network), each of the 8 variants plus the faithful claims yields the listed verdict, concern(s) and evidence roles; V1 explanation contains "inflates"; a fictional severity pair "mild" vs "severe" yields `partially_supported` + `frequency_severity_inflation` with "understates".
2. Decision-table tests: one per row D0-D9, each output passes 02 `check_rules` and the response schema; property test over random signal combinations never yields `supported` with a contradicting item, `contradicted` without one, `partially_supported` without a concern, or a contradicting item under `insufficient_evidence`.
3. Ordinal tests, both directions: stronger/weaker/tie/adjacent-mixed-basis/at-least cue/bucket-0 escalation/gap-3 escalation; same-basis-first pairing; vague quantifier not flagged as understatement; rate ratio catches twice vs once a month.
4. Group A: reporter mismatch via speaker string and via role; role `None` turn skips A3/A9 and lands in `ambiguous_turns`; bucket 9 fires only for interviewer question turns; zero verifier calls for no-evidence and question-only claims (call counter).
5. Dedup: V3-style item yields only `wrong_attribution`; V2 yields only `negation_reversal`; generic `unsupported_claim` absent when a specific concern exists; one concern per type.
6. Floors: a `low` conf flag is dropped and counted; P1 flag at `medium` is emitted.
7. Inference: all premises entail + step acceptable gives support; one failing premise or step gives `qualifying` premises and `insufficient_evidence` + `unsupported_claim`.
8. Failure: verifier timeout, bad JSON twice, and an unjudged item after retry each stall only that claim; other claims complete; `check_claims` never raises.
9. Neutrality and rule 9: banned-word test over all templates; no 8-gram of any test transcript and no timestamp in any explanation or revision; sentinel test shows no claim/turn text in logs.
10. Leak test: no 8-gram overlap between `check_fewshot.json` and any eval fixture. Whole suite offline (`LLM_MODE=stub|fixture`).
