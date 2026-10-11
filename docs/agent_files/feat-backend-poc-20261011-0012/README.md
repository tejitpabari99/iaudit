---
status: draft
summary: Backend POC (FastAPI, contract 1.0.0) - 8 sub-projects, PRDs + locked decisions
date: 2026-10-11
---
# Backend POC - index

FastAPI proof of concept in `backend/` that serves the audit contract 1.0.0 (`POST /v1/audit`) through a staged pipeline: prep, atomize, link, check, assemble. Source: [brainstorm.md](brainstorm.md). Governing scope: `docs/mission.md` (wins on conflict). Locked overrides from dev-design are listed below and win over the brainstorm.

## Sub-projects
| # | Sub-project | Depends on | PRD | Phase |
|---|---|---|---|---|
| 01 | Scaffold + API | none | [PRD](01-scaffold-api/PRD.md) | Foundation |
| 02 | Assembler + contract enforcement | 01 | [PRD](02-assembler/PRD.md) | Core stage |
| 03 | Prep + Atomize | 01 | [PRD](03-prep-atomize/PRD.md) | Core stage |
| 04 | Link | 01, 02, 03 | [PRD](04-link/PRD.md) | Core stage |
| 05 | Check | 02, 03, 04 | [PRD](05-check/PRD.md) | Core stage |
| 06 | Orchestration + latency | 01, 02, 03, 04, 05 | [PRD](06-orchestration/PRD.md) | Integration |
| 07 | Eval data + harness | 01, 02 (harness); 03-05 (Linked mode); 06 (end-to-end) | [PRD](07-eval/PRD.md) | Validation |
| 08 | Docs update | 01-07 | [PRD](08-docs-update/PRD.md) | Docs (last) |

## Dependency graph
```
01 -> {02, 03}; {01, 02, 03} -> 04; {02, 03, 04} -> 05; {02, 05} -> 06 (full: 01-05)
07: harness after 01, 02; Linked mode after 03-05; e2e after 06
all -> 08
```
Per PRD headers: 04 depends on 01, 02, 03; 05 on 02, 03, 04; 06 on 01-05.

## Locked decisions
- Code in `backend/`, src layout `backend/src/iaudit_backend/`. Python 3.12 + uv + ruff + pytest.
- `POST /v1/audit` + `GET /healthz`; CORS from env allowlist; no auth.
- `role` optional: missing role skips checks 3/9 and adds `ambiguous_speaker` (overrides brainstorm "always supplied").
- Retrieval: BM25 + rapidfuzz. No embeddings/spaCy/torch unless linked-mode gain >= 0.05 F1.
- LLM via LiteLLM, model tiers (cheap/mid); fixture replay in tests (no network); live runs opt-in.
- No body/transcript logging (allowlist logging only).
- Limits: 2,000 turns / 50 sections / 2 MB body.
- Section failure -> `section_not_processed` + `partial`. Claim stall (link/check fail, timeout) -> `insufficient_evidence` + `other` warning.
- 02 `locate()` is the single matcher; `ClaimDraft` uses `segments` + `span_hint` (no `claim_text`/`char_hint`).
- Shared prompt-cache layout: system -> transcript -> stage -> claim; warmed at t=0.
- `IAUDIT_PIPELINE` defaults to `stub` in SP01; SP06 flips default to `full`; tests always set `stub` explicitly.
- Understatement reuses `frequency_severity_inflation` with "understates" wording (no new concern type).
- Contract stays 1.0.0 (no schema change); `mission.md` not edited without user approval.
- Offsets are code points. 60 s request deadline (configurable).
- POC gate = ADHD faithful returns no concerns + 8 single-fault variants return expected verdict/concern. 60/20/20 split, P0 precision gate, P1 suppression deferred.
- Explanations are code templates (neutral, no transcript copying); LLM observes, code decides verdicts.

## Gaps found in the brainstorm
| Gap | Resolution |
|---|---|
| Role "always supplied" vs contract | Role optional (locked override) |
| No HTTP path / CORS | `POST /v1/audit`, env CORS allowlist |
| No input limits | 2,000 turns / 50 sections / 2 MB |
| No API keys on machine | Manual step (below) |
| No offline test strategy | LLM fixture replay in CI |
| PHI logging unspecified | Allowlist logging, no bodies |
| 60/20/20 split impossible with 1 interview | POC gate now; split deferred |
| Gold ambiguities V2/V4 | 07 O-1 / O-2 |
| Claim-level timeout undefined | Locked: stall -> `insufficient_evidence` + `other` warning |
| Heavy deps (embeddings, spaCy, torch) | Deferred (>= 0.05 F1 rule) |
| Python tooling unspecified | uv / ruff / pytest |
| Mission check 7 says "exaggerates"; understatement flag added | 08 O-1, needs user approval |
| Faithful S5 "repeatedly" false-understate risk | 05 at-least cue list (05 O-2) |
| 02/03 claim-shape mismatch | Resolved: `segments` + `span_hint`, single `locate()` |

## Open questions (consolidated)
Items still tagged [OPEN] in PRDs. Edit in the PRD, not here.
| ID | Question | Owner SP |
|---|---|---|
| O-01-1 | LiteLLM id form for Claude; `response_format` json_schema honored per tier | 01 |
| O-01-2 | `cache_control` passthrough and `cache_read_input_tokens` in usage | 01 (06 benchmark) |
| O-01-3 | Duplicate-id error message: count-only vs naming ids | 01 |
| O-01-4 | Streaming body-size cap under ASGI buffering | 01 |
| O-02-1 | Fuzzy threshold 90 / min len 12 / window ratio tuning | 02 |
| O-02-2 | `cross_turn_slack` default 0 may split Q+"yes" evidence | 02 |
| O-02-3 | Rule 9 n-gram size 8 / quote length 4 tuning | 02 |
| O-02-7 | `no_claims_located` as `section_not_processed` vs `other` warning; frontend render | 02 |
| O-03-2 | Unmarked corrections tagged `answer` (ADHD turn 23) | 03 |
| O-03-3 | Are all assessor observations `unobservable`? | 03 |
| O-03-4 | Surface dropped claims / coverage gaps as `other` warning | 03 |
| O-03-5 | Scale bucket boundaries and mixed numeric/relative handling | 03 |
| O-03-6 | Atomize retry fits 5-10 s budget only on fast tiers | 03 |
| O-04-1f | `ambiguous_turns` filled by 04 for evidence turns only; 06/05 merge others | 04 |
| O-04-2 | `LINK_WHOLE_MAX_TOKENS` 8,000: whole vs shortlist benchmark | 04 |
| O-04-3 | Shortlist defaults (k=8, floor 2, window 4, cap 30), stemmer, widen-on-empty | 04 |
| O-04-4 | Embedding retriever only if >= 0.05 F1 gain (deferred until then) | 04 |
| O-04-7 | BM25Okapi idf on tiny corpora (epsilon / BM25Plus / stemmer) | 04 |
| O-04-8 | Question snapping over-highlighting | 04 |
| O-04-9 | V6 turn 23 label: `contradicting` vs `qualifying` | 04 |
| O-05-1 | Materiality thresholds, ambiguous band, contradict gap, rate ratio | 05 |
| O-05-2 | At-least cue list / vague quantifiers (S5); maybe `Quantity.bound` in 03 | 05 |
| O-05-3 | No amount check when claim lacks a quantity of that kind | 05 |
| O-05-4 | Faithful sensitivity (S7, S3, S2) with strict verifiers | 05 |
| O-05-5 | Confidence floors 0.5 / 0.6, enum mapping, P1 suppression | 05 |
| O-05-7 | A9/A3 skip when roles missing (locked; relax only by user) | 05 |
| O-05-8 | V3 precedence `wrong_attribution` over `negation_reversal` | 05 |
| O-05-9 | V7/V8 flag-cited items assume 04 returns those turns | 05 |
| O-05-10 | Verifier limited to 04 shortlist in shortlist mode | 05 |
| O-05-11 | `suggested_revision` quality, default-on, copycheck nulls | 05 |
| O-05-12 | Out-of-scope `ClaimDraft` needs `claim.oos_reason` from 03 | 05 |
| O-05-13 | No isolated bucket-2 / bucket-10 negative in ADHD set | 05 (07 O-8) |
| O-06-1 | Nothing-usable policy: 503/504 vs 200 partial-all-sections | 06 |
| O-06-2 | Escalate section to `section_not_processed` when claims mostly stalled | 06 |
| O-06-3 | Cancelled non-streaming Anthropic call billing | 06 |
| O-06-4 | Over-cap claim policy and cap value (80) | 06 |
| O-06-5 | FIFO semaphore starvation vs priority field | 06 |
| O-06-6 | Min cacheable prefix; LiteLLM cache usage fields | 06 |
| O-06-7 | Stage timeouts (10/25/20/20) and assemble reserve 3 s | 06 |
| O-06-8 | Event-loop/semaphore sharing across concurrent audits | 06 |
| O-07-1 | V2 allow `contradictory_evidence` as non-required | 07 |
| O-07-2 | V4 evidence role / whether evidence required | 07 |
| O-07-3 | V5 stretch bucket: gate vs report | 07 |
| O-07-4 | Partial-support expected roles (V1, V6, V7, V8) | 07 |
| O-07-5 | Multi-fault ADHD variant content (R4) | 07 |
| O-07-6 | `fixture_key` via `RequestContext` scope vs request-hash keys | 07 |
| O-07-8 | No pure-unsupported (bucket 1) variant | 07 |
| O-07-9 | Collateral false alarms gate acceptance? | 07 |
| O-07-10 | Role handling for merged CDR turns | 07 |
| O-07-11 | CDR orientation expected verdict/concern | 07 |
| O-07-12 | Faithful sensitivity S7 / S3 (checker findings, not gold errors) | 07 |
| O-07-13 | Variant 1 wording (added subject is a proposal) | 07 |
| O-07-15 | `proposition` gold field; S1 mixed relative/numeric bucket | 07 |
| O-08-1 | Mission check 7 "exaggerates" vs understates; needs user approval | 08 |
| O-08-2 | Nothing-usable policy documented as landed (see 06 O-1) | 08 |
| O-08-3 | Numbers unmeasured until live runs | 08 |
| O-08-4 | Final file names may differ from PRDs | 08 |
| O-08-5 | Frontend docs linking back to backend (feat/review-ui owner) | 08 |

Resolved/deferred items are not listed; they are tagged inline in each PRD's "Risks / Open Questions" (counts of RESOLVED tags: 01: 4, 02: 5, 03: 2, 04: 7, 05: 4, 06: 5, 07: 3, 08: 1).
Open counts: 01: 4, 02: 4, 03: 5, 04: 7, 05: 12, 06: 8, 07: 13, 08: 5.

## Manual steps (human)
- Set provider API key(s) (`ANTHROPIC_API_KEY` or other); confirm model ids.
- Approve multi-fault ADHD variant text (07 O-5).
- Supply real CDR transcript file (07).
- Approve mission/concept wording for understatement (08 O-1).
- Agree nothing-usable policy with frontend: 503/504 vs partial (06 O-1).
- Add iaudit row to `/root/projects/README.md` (machine index, outside repo; note only).
- DAIC-WOZ licence signer (non-blocking).

## Next step
Human reviews PRDs, resolves [OPEN] items in-PRD, then run `dev-tasks`.
