---
status: draft
summary: Pure-Python final stage that resolves verbatim excerpts/claim text to code-point spans, drops what it cannot find, repairs contract-rule violations, and schema-validates the AuditResponse.
date: 2026-10-11
---
# PRD: 02 Assembler + contract enforcement

Repo/branch: iaudit / `feat/backend-poc` · Depends on: 01-scaffold-api · Owns: `backend/src/iaudit_backend/assembler/` (+ tests under `backend/tests/`); package `iaudit_backend` per SP01

## TL;DR
Stage 4 of the pipeline. Takes internal claim/evidence results that carry **verbatim text, never offsets**, and returns a contract-1.0.0 `AuditResponse` dict plus a counts-only report. This is the guard that blocks invented quotes: an excerpt that cannot be found in the cited turn is dropped, never guessed. When dropping breaks a contract rule, a fixed repair table downgrades the verdict conservatively. Deterministic, no network, no LLM, <1 s.

## Problem
LLM stages cannot be trusted to emit offsets or to keep verdict, evidence and concerns consistent. The contract needs code-point spans (`docs/contracts/README.md` "How to read a span"), 12 rules the schema only partly enforces, and no transcript text in the response (rule 9). Mission: never invent quotations.

## Goals / Non-Goals
**Goals**
- Excerpt to span: exact, then normalized, then guarded fuzzy; offsets always in the ORIGINAL strings.
- Claim text to section span; non-overlap/non-nesting.
- Rules 1–12 enforced with a deterministic repair policy; warnings, `status`, ids, `request_id` echo.
- Final JSON-schema validation; failure means `internal_error`, never a half audit.

**Non-Goals**
- Deciding verdicts, concerns or explanations (SP05); linking (SP04); atomizing (SP03).
- LLM rewriting of text that fails the rule 9 check (parked, see O-5).
- Searching outside the cited turn(s) (see D-3). HTTP, error bodies (SP01), timeouts and parallelism (SP06).

## Scope tiers
**Core** (must ship): R1 locator (exact, normalized, guarded fuzzy, ellipsis, hint), R2 evidence resolution and span/cross-turn rules, R3 claim resolution with overlap policy for nested and identical spans (partial overlap: drop the less severe claim, see fallback), R4 repair table rows 1-12 (rules 1-12), R5 reduced rule 9 check (quoted span of 4+ words, timestamp patterns, drop `note` fields that are exact substrings of a turn text, `suggested_revision` null and fixed fallback text on violation), R6 warnings/status, R7 final validation and schema check, D-1 to D-7, D-10, D-11, O-8. Matching acceptance criteria.

**Optional (gated: after all core work in every SP)**:
- [OPTIONAL] Rule 9 8-gram copy check (R5, D-9, O-3). Core fallback: strip `note` fields that are exact substrings of turn text; the quoted-span and timestamp checks stay core.
- [OPTIONAL] Partial-overlap trimming finesse (R3 overlap policy, D-8 trim part). Core fallback: drop the less severe overlapping claim and add an `other` warning with `claim_id`.

## Requirements

**Internal input model (ASSUMPTION: SP03/04/05 produce exactly this; SP06 assembles it). Pydantic models in `assembler/models.py`:**

| Model | Fields |
|---|---|
| `ExcerptSegment` | `turn_index`, `text` (verbatim, one turn only), `char_hint?` |
| `EvidenceDraft` | `key`, `role` (supporting/contradicting/qualifying), `segments[]` (ordered; length 1 = single turn), `note?` |
| `ConcernDraft` | `type` (12 enum), `explanation`, `evidence_keys[]` |
| `ClaimDraft` | `key`, `section_index`, `segments[]` (1-3 verbatim strings of section text, document order; same as SP03 `AtomicClaim.segments`), `span_hint?` ((start, end) code points in the section, located by SP03 via `locate()`; same as SP03), `outcome` (`ok`/`stalled`), `verdict`, `explanation`, `out_of_scope_reason?`, `suggested_revision?`, `concerns[]`, `evidence[]`, `ambiguous_turns[]` (turns with missing role the claim relied on) |
| `InternalAudit` | `claims[]`, `section_failures[]` (`section_index`, `reason` timeout/error), `truncated` flag |

Entry point: `assemble(request, internal, cfg) -> AssembleOutcome(response: dict, report: AssembleReport)`. `report` holds counts only (match tier and score per evidence, drops by reason, repairs by code); never text. Raises `AssemblyError` if the final validation fails; SP01 maps it to `internal_error` (500, generic message, no body echo).

**R1 Locator (shared function, exported).** `locate(text, needle, hint, cfg) -> Match(start, end, tier, score) | None` over one string. Tiers in order:
1. Exact `str.find`.
2. Normalized: build a normalized copy plus an index map (normalized char to original index). Per-char normalization: NFKC, casefold, curly quotes/dashes to ASCII, zero-width removal, whitespace runs collapse to one space (map points at run start). A hit `[a,b)` maps back to `[map[a], map[b-1]+1)`. Trim to non-space; never slice the normalized string.
3. Fuzzy: `rapidfuzz.fuzz.partial_ratio_alignment` on normalized strings, then map back. Only if needle has at least `fuzzy_min_len` (12) normalized chars and score >= `fuzzy_threshold` (90). Guards: aligned window length within 0.8–1.25x needle; digits, number words (one..ten, twice, daily…) and negators (no, not, never, n't, none) must match as multisets between needle and window, so fuzzy cannot turn "twice" into "once" or "did not" into "did"; snap to word boundaries.
- Ellipsis ("...", "[…]") in a needle: split into fragments, locate each in order in the same string, span = first start to last end; reject if hull > 2.5x total fragment length.
- Repeated occurrences: choose the one nearest `hint`; no hint, first. Count of occurrences goes in the report.
- Offsets are computed on request strings exactly as received: no strip/NFC/newline normalization anywhere before this stage (`\r\n` is two code points).
`locate` is the single offset-resolution implementation: SP03 calls it per segment within the section (it has no matcher of its own) and the assembler calls it for fallback. [RESOLVED: SP03 O-1 / SP02 alignment]

**R2 Evidence resolution.** Search ONLY the cited turn for each segment. Unmatched segment: drop. Evidence whose segments are all unmatched is dropped, and its key is removed from every `concerns[].evidence_keys`.
- Single segment: span `{turn, start}` to `{turn, end}`, `end > start`.
- Multi segment: merge into one cross-turn span only if turns are consecutive, the first segment runs to the end of its turn, and the last starts at the turn start (whitespace slack only; `cross_turn_slack` default 0 chars), because the walk includes the rest of the first turn and the head of the last. Else split into separate evidence items per segment (same role, note on the first). Concerns pointing at the original key point to all its parts.
- Cross-turn rule: end char is always `> 0` by construction (segments are non-empty after trim; an empty last segment is dropped).
- Final ids: `e1..eN` per claim, assigned by the assembler (checker ids are internal keys). Identical span+role duplicates collapse.

**R3 Claim resolution.** Verify `span_hint` first: valid if within the section and the slice equals or contains every segment (normalized comparison, in order). If absent or invalid, run `locate(section.text, segment, hint)` per segment within one section (same tiers, document order, each after the previous end), and take the covering span (first start to last end; non-contiguous segments keep the covering span, as SP03 R7.5). Any segment unlocatable: claim unlocatable. A stale hint is a hint only; threshold for claims is configured separately (`claim_fuzzy_threshold`, default 90). Span trimmed to non-whitespace, `start.index == end.index == section_index`. Claim ids: `c1..cN` in document order after sorting by `(section, start)`.
- Unlocatable claim: drop it and add an `other` warning naming no text (section index and claim id of the draft). If a processed section has non-blank text and zero located claims, mark it `section_not_processed` (reason `no_claims_located`) and status partial.
- **Overlap policy (rule 8), applied per section after location:**
  - Nested: drop the outer, keep the narrower claims (finer atomic claims; gaps are legal).
  - Identical span: keep the more severe verdict (contradicted > partially_supported > insufficient_evidence > supported > out_of_scope); tie: first.
  - Partial overlap: drop the less severe claim (tie: the later one) and add a warning. [OPTIONAL: trim the less severe claim's overlapping end instead; if fewer than 3 non-space chars remain, drop it.]
  - Every drop or trim adds one `other` warning with `claim_id`.

**R4 Rule enforcement and repair.** Runs per claim after R2, in this order; each repair appends a code to the report and, when the verdict changes, replaces `explanation` with a fixed template (the old explanation described evidence that no longer exists) and adds an `other` warning with `claim_id`.

| Situation (after drops) | Repair |
|---|---|
| Concern whose evidence keys were all dropped | Remove concern. Concerns that had no evidence keys originally (e.g. `unsupported_claim`) stay. Duplicate concern types merge (union of ids, first explanation). |
| `supported`, no supporting excerpt | `insufficient_evidence`; add `unsupported_claim` concern with `evidence_ids: []` |
| `supported`, a contradicting excerpt survives (rule 2) | `partially_supported` + `contradictory_evidence` concern pointing at it; flagged for review |
| `contradicted`, no contradicting excerpt (rule 3) | `insufficient_evidence` (never `supported`); keep remaining non-contradicting evidence |
| `partially_supported`, zero concerns (rule 4) | `insufficient_evidence` |
| `insufficient_evidence` with contradicting excerpt (rule 5) | `contradicted` (the contract names this verdict) |
| `out_of_scope` (rule 6) | Strip concerns and evidence; if `out_of_scope_reason` missing use a fixed generic reason. Other verdicts: strip `out_of_scope_reason`. |
| `outcome=stalled` | Build `insufficient_evidence`, fixed explanation, no evidence, plus `other` warning naming the claim |
| Rule 7 | Concern `evidence_ids` filtered to this claim's final ids; deduped |

**R5 Rule 9 copy check** (`copycheck.py`), applied to `explanation`, concern explanations, `note`, `suggested_revision`, `out_of_scope_reason`, warning messages:
- Core violation = a quoted span of 4+ words found in the transcript; or any request `timestamp` string / `\d\d:\d\d` pattern; or a `note` that is an exact substring of any turn text (note dropped).
- [OPTIONAL] Additional violation = any word 8-gram (lowercased, NFKC, punctuation stripped) shared with any transcript turn and NOT present in the assessment text (assessor's own words are allowed).
- Repair: `note` dropped; `suggested_revision` set to null; a violating explanation/reason loses the offending sentences; if nothing is left use the fixed per-verdict (or per-concern-type) fallback text plus an `other` warning.
- [OPTIONAL] Transcript n-gram set built once per request (8-gram check only).

**R6 Warnings, status.** Order: section warnings, claim warnings in claim order, then aggregates.
- `section_not_processed` + `section_index`: from `section_failures` and R3.
- `span_out_of_range` + `claim_id`: only if the independent bounds check (below) rejects a span after resolution; the evidence or claim is dropped. Defensive, expected to be unreachable.
- `ambiguous_speaker`: one per claim with non-empty `ambiguous_turns`; message names turn indexes, never text, and no speaker is guessed.
- `input_truncated`: only when `internal.truncated`.
- `other`: R3/R4/R5 events, one per claim (counts), fixed templates only.
- `status` = `partial` iff any `section_not_processed` exists (rule 11); else `complete`. `request_id` copied unchanged when present, omitted otherwise (rule 10).

**R7 Final validation.** An independent `check_rules()` re-verifies rules 1, 7, 8 and 10 from the final dict (span walk: start before end, pointers inside items, `char` equals length only on end pointers, claim `start.index == end.index`, no cross-turn end `char: 0`). Then `jsonschema` Draft 2020-12 against `audit-response.schema.json` (loaded once at startup via SP01's schema loader). On errors: group by `claims[i]` path; quarantine offending claims (drop + `other` warning), re-validate once; any error outside `claims`, or a second failure, raises `AssemblyError`. Log json-pointer and keyword only.

## Architecture
```
InternalAudit + AuditRequest
  -> claim locate (R3) -> evidence locate (R2) -> repair (R4) -> copycheck (R5)
  -> warnings/status (R6) -> check_rules + schema (R7) -> (response dict, report)
```
Modules: `models.py`, `normalize.py` + `locate.py`, `claims.py`, `evidence.py`, `repair.py`, `copycheck.py`, `warnings.py`, `validate.py`, `__init__.py` (`assemble`, `locate`). Pure functions, thresholds from SP01 `Settings` (env-overridable): `fuzzy_threshold`, `claim_fuzzy_threshold`, `fuzzy_min_len`, `copy_ngram`, `cross_turn_slack`. Shared with: SP01 (models, schema loader, error mapping, settings, logging), SP03 (`locate`, produces `ClaimDraft` fields), SP04 (produces `ExcerptSegment`s: verbatim, one per turn, `turn_index` from the shortlist it supplied; SP04 rejects indexes not in the shortlist), SP05 (produces verdict/concerns/explanation), SP06 (calls `assemble`, supplies `section_failures`, `stalled`, `truncated`), SP07 (reads `AssembleReport` for tier/drop stats; fixtures reproduce `examples/response.json` spans).

## Decisions
| # | Decision | Choice | Alternatives considered | Why |
|---|---|---|---|---|
| D-1 | Who computes offsets | Assembler, from verbatim text | LLM returns offsets | LLM offsets are unreliable; blocks invented quotes |
| D-2 | Match ladder | exact, normalized with index map, guarded fuzzy | Fuzzy only; always normalize | Exact is free and certain; map keeps original offsets when normalization changes length |
| D-3 | Search scope | Cited turn only; no widening | Widen to neighbors/whole transcript | Relocating to another turn can change the speaker, silently invalidating checks 3 and 9 |
| D-4 | Fuzzy guards | Min length, window ratio, numeric/negation multiset | Score threshold only | A 90 score can still flip "twice" to "once" |
| D-5 | Cross-turn merge | Merge only when walk adds no extra text; else split | Always merge | Walk includes whole first turn tail and last turn head; over-inclusion would highlight text the linker never cited |
| D-6 | Dropped evidence | Downgrade verdict conservatively with fixed template text | Keep verdict; ask LLM to redo | Never leave a verdict with no visible basis; pure Python; old explanation would be stale |
| D-7 | Never upgrade to `supported` or invent concerns | Repairs only move toward `insufficient_evidence` (or contradicted per rule 5) | Re-derive verdict | Assembler has no meaning judgement |
| D-8 | Overlap | Keep narrower; severity wins ties; partial overlap drops the less severe claim [OPTIONAL: trim instead of drop] | Merge claims; drop later | Finest-grained claims serve the reviewer; problems must not be hidden |
| D-9 | Rule 9 check | Core: quote and timestamp checks plus exact-substring `note` drop; [OPTIONAL: 8-gram vs transcript minus assessment words]; strip, no rewrite | Embedding similarity; LLM rewrite | Cheap, deterministic, low false positives |
| D-10 | Validation failure | Quarantine bad claims once, else `internal_error` | Return best effort | Consumer rejects invalid bodies and must not render half an audit |
| D-11 | Ids | Assembler assigns `cN` / `eN` | Pass through checker ids | Guarantees rule 7 uniqueness |

## Risks / Open Questions
- O-8 [RESOLVED: `ClaimDraft` uses SP03 `segments` + `span_hint`; `locate()` is the only matcher; SP03 passes hints located via `locate()`, 02 re-verifies and falls back] Alignment with SP03 O-1 / SP07 O-14.
- O-1 [OPEN] Fuzzy threshold (90), `fuzzy_min_len` (12) and window ratio are guesses. Tune on the ADHD fixtures plus recorded linker output: report per-tier hit rate and false-match rate; pick the lowest threshold with zero wrong-turn/wrong-text matches.
- O-2 [OPEN] `cross_turn_slack` default 0: may split Q+"yes" evidence too often when the linker trims trailing punctuation. Check split rate in linked-mode runs.
- O-3 [OPEN] [OPTIONAL] Rule 9 n-gram size 8 and quote length 4: tune against false positives on short transcripts (idiomatic phrases) and misses on paraphrase-lite explanations.
- O-4 [RESOLVED: 05 never emits `supported` with a contradicting item; mixed support+contradiction = `partially_supported` + specific concern, per 05 decision table D5/D8, matching this repair] Repair for rule 2 violation (supported plus contradicting) is `partially_supported`; SP05 should never emit it, so confirm with SP05 whether to prefer `contradicted`.
- O-5 [DEFERRED] LLM rewrite of rule-9 violators and relocating evidence outside the cited turn: only if drop-rate in benchmark justifies.
- O-6 [DEFERRED] Lone surrogates / grapheme clusters in request text; NFKC per char differs from whole-string NFKC for combining sequences (match-only impact).
- O-7 [OPEN] `no_claims_located` marking a section `section_not_processed`: confirm the frontend renders this acceptably; alternative is an `other` warning with `complete`.
- R-1 [RESOLVED: code-point offsets] Python `str` indexes by code point, same as the contract; no conversion needed.
- R-2 [RESOLVED: dropped, not guessed] Unmatched evidence is never kept with an approximate span.

## Acceptance Criteria
- Feeding drafts equivalent to `docs/contracts/examples/` into `assemble` reproduces every claim and evidence span in `examples/response.json` (e.g. `c-contradicted` is `{0,57}`–`{0,97}`, `e-appointments` is `{2,0}`–`{3,45}` as one merged cross-turn span).
- Unit tests, no network, cover: exact; case/whitespace/curly-quote variant mapped to original offsets; length-changing normalization (ß, NFKC ligature, multiple spaces); ellipsis excerpt; repeated phrase picks hint then first; emoji before the match keeps code-point offsets; fuzzy accepts a typo and rejects "once" vs "twice" and "did not" vs "did"; unmatched excerpt dropped.
- Cross-turn: merged span never ends at `char: 0`; non-adjacent or partial-turn segments split into separate items.
- One test per repair row in R4 asserts the resulting verdict/concerns pass the real schema and `check_rules`; none ever yields `supported` from a lower verdict.
- Overlap tests: nested, identical, partial (core: less severe claim dropped with warning; [OPTIONAL] trimmed to non-overlapping remainder); resulting claim spans never overlap or nest.
- Rule 9: a quoted 4+ word transcript span in `explanation` is stripped or replaced; a `note` that is an exact substring of a turn is dropped; [OPTIONAL] an 8+ word verbatim transcript run in `explanation` is stripped or replaced; `suggested_revision` becomes null; assessor-only phrases pass; timestamps are caught.
- Warnings: failed section yields `section_not_processed` + `section_index` and status `partial`; no failures yields `complete`; `request_id` echoed or omitted; missing-role claim yields `ambiguous_speaker`.
- A forced schema failure raises `AssemblyError`; a single bad claim is quarantined with a warning and the rest returns.
- `report` and logs contain counts and ids only (test greps outputs for a canary transcript string).
