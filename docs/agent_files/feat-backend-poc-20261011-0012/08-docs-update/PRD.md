---
status: draft
summary: Final docs pass after code lands - architecture.md backend section, implementation/reference/concepts/decisions/testing/examples content following the AGENTS.md placement table, backend/README quickstart, link fixes, and an optional env-var drift test; one owner-of-truth per fact.
date: 2026-10-11
---
# PRD: 08 Docs update

Repo/branch: iaudit / `feat/backend-poc` · Depends on: 01-07 (all code landed) · Owns: `docs/{architecture.md,README.md}`, `docs/{implementation,reference,concepts,decisions,testing,examples}/`, `docs/specs/README.md`, `docs/contracts/README.md` (cross-link only), `backend/README.md`, root `README.md`, `backend/tests/test_docs_drift.py` (optional)

## TL;DR
Docs are written AFTER the code, from the code, so every fact traces to a file. Each fact has exactly one home (table below); other docs link to it. The contract stays 1.0.0 and untouched except one cross-link. `mission.md` is not edited. One new decision per key choice as short ADRs. No spec doc is needed: the contract rules plus one implementation-level verdict table cover it.

## Problem
Main has placeholder READMEs in `docs/{concepts,implementation,reference,decisions,testing,examples,specs}/` and a 5-line `architecture.md` [verified: cat]. After SP01-07, the endpoint, env vars, error mapping, pipeline stages, eval harness and the understatement behavior exist only in PRDs under `agent_files/`, which are branch scratch, not durable docs. The frontend index lists endpoint/CORS as deferred O-03-1 [verified: grep].

## Goals / Non-Goals
**Goals**
- Docs in the AGENTS.md placement table, one owner per fact, links resolve, no PHI/transcript text.
- A new contributor can run, test and call the backend from `backend/README.md` + `docs/` alone.
- Honest limits stated: low-fidelity P1 checks, "no concern is not correct", POC acceptance only.

**Non-Goals**
- Contract edits (stays 1.0.0), `mission.md` edits (see [OPEN] O-1), new features, API docs generated from OpenAPI, copying PRD text wholesale, real CDR content.
- Rewriting `agent_files/` PRDs (history stays as is).

## Scope tiers
Hackathon build: **Core** is the critical path and ships first. **Optional** items are gated: start only after ALL core work in every sub-project is finished. No core item depends on an optional one (core docs link only to core docs).

**Core**
- `backend/README.md` (R10): quickstart with `uv sync`, env/API key, run, test, smoke script (`uv run python -m eval.smoke`, from 07a).
- `docs/reference/backend-api.md` (R3): endpoint, errors -> HTTP, env vars (a short table of the `IAUDIT_*` vars from `Settings`, replacing a separate `configuration.md`), limits, warnings emitted-when (replacing a separate `warnings.md`).
- Short `docs/architecture.md` backend blurb (R2, a few lines plus link to `reference/backend-api.md`).
- Root `README.md`: Backend link (and Contract link).
- `docs/reference/README.md`: one index line for `backend-api.md`.
- Acceptance 1 (core files only), 3 (core links), 4 (no contract edit), 5 (with `backend/README.md` only; curl line may live in the README), 7.

**Optional (gated: after all core work in every SP)**
- ADRs `docs/decisions/` (R6) and index.
- `docs/concepts/` (R4), `docs/implementation/` (R5), `docs/testing/` (R7), `docs/examples/` (R9) and `docs/specs/README.md` note (R8).
- Separate `reference/configuration.md` and `reference/warnings.md` (R3), and the `docs/contracts/README.md` cross-link (R10).
- Drift test (R11) and acceptance 2, 6, 8.
- Fuller `architecture.md` (full code map, five-stage diagram) beyond the blurb.

## Requirements

### R1 Owner-of-truth map (nothing duplicated)
| Fact | Owner doc | Others do |
|---|---|---|
| Request/response shape, rules 1-12, error codes list, warning codes | `contracts/README.md` + schemas | link |
| Endpoint, HTTP status per error code, limits, env vars, warnings emitted-when | `reference/backend-api.md` | link |
| Verdict/concern semantics for a reader | `concepts/` | link |
| Stage internals, models, tiers, caching, deadline, logging | `implementation/` | link |
| Why a choice was made | `decisions/NNNN-*.md` | one-line "see ADR-N" |
| Commands to run/test/eval | `backend/README.md` (run, test) and `testing/` (eval, release) | link |

### R2 `docs/architecture.md` (extend, keep short, audience everyone)
Core: only the first bullet below (backend blurb + link); the remaining bullets are **[OPTIONAL]**.
- Add: frontend (not yet on main) -> `POST /v1/audit` -> `backend/` -> LLM providers via LiteLLM. One sync call, target 30-45 s, hard cap 60 s (`AUDIT_DEADLINE_S`), frontend client timeout 120 s.
- Five-stage flow in one diagram or list: 0 Prep, 1 Atomize, 2 Link, 3 Check, 4 Assemble (LLM proposes, code decides; LLM never emits offsets).
- Where code lives: `backend/src/iaudit_backend/{api,config,contract,llm,prep,atomize,link,check,assembler,pipeline}`, `backend/eval/`, `backend/tests/` [verify names against landed tree].
- Contract stays the single schema source: backend reads `docs/contracts/` in place.
- Links only for detail. No env vars or status codes here.

### R3 `docs/reference/` (users; lookup facts; each row traces to code)
**Core:** `backend-api.md` only, which also carries a compact env-var table, the limits and the warnings list; plus the `reference/README.md` index line. `configuration.md` and `warnings.md` below are **[OPTIONAL]** full versions that, when written, replace those sections with links.
New files, plus README index:
- `backend-api.md`: `POST /v1/audit`, `GET /healthz` (response `{"status":"ok"}`); optional `Server-Timing` response header (durations only); request checks order; **error table** code -> HTTP (invalid_request 400 body/schema/dup ids, 422 framework fallback; unsupported_version 400; payload_too_large 413 for bytes/turns/sections; unavailable 503 provider down, 504 deadline; internal_error 500; other 404/405); CORS (allowlist, default `http://localhost:5173`); no auth in POC.
- **[OPTIONAL]** `configuration.md`: every `IAUDIT_*` var with default and meaning, generated by reading `Settings` (not by memory). Groups: limits, CORS, models (3 tiers), timeouts/deadline, concurrency, `LLM_MODE` live/fixture/stub, `PIPELINE` stub/full, `SERVER_TIMING`, `CONTRACTS_DIR`, `LOG_LEVEL`, stage-specific knobs landed by 03-06. API keys are LiteLLM's standard provider env vars, not `IAUDIT_*`.
- **[OPTIONAL]** `warnings.md`: for each contract warning code, WHEN the backend emits it: `section_not_processed` (atomize failure/deadline/no claims located), `ambiguous_speaker` (claim relies on a turn with no `role`), `input_truncated` (over-cap claims), `span_out_of_range` (defensive, expected unreachable), `other` (claim-level link/check stall, unlocatable claim). Status rule: `partial` iff any `section_not_processed`.
- `limits.md` may be folded into `backend-api.md` (decide by length).

### R4 [OPTIONAL] `docs/concepts/` (users + contributors; plain language)
- `how-an-audit-decides.md`: claim -> evidence -> verdict -> concerns in user terms; what each verdict means; LLM observes, code decides; every highlight is a position, never a copied quote.
- `severity-and-frequency.md`: ordinal compare in both directions; claim weaker than interview -> `partially_supported` + `frequency_severity_inflation` with explanation saying "understates"; stronger -> "inflates"; ties/immaterial -> no concern. States plainly this reuses the existing concern type (contract has no new value) [OPEN O-1 on mission wording].
- `limits-of-an-audit.md`: **no concern does not mean correct**; omitted concern = not found, not "skipped" (no coverage field in 1.0.0); P1 checks (4, 6, 8, 10, 11, 12) are quick-pass, low fidelity, may miss or over-flag; `insufficient_evidence` is not "wrong"; fidelity is to the supplied interview only; wording is a prompt for human review, not an allegation (rule 12).
- `missing-role.md` (or section above): `role` optional; never guessed; checks 3 and 9 skipped for that turn; `ambiguous_speaker` warning; often yields `insufficient_evidence`.
- Each concept page cites the mission check numbers, not copies.

### R5 [OPTIONAL] `docs/implementation/` (maintainers/agents)
Files (one per stage family; each = purpose, inputs/outputs, key rules, file pointers, config knobs by name only):
- `pipeline.md`: 5-stage DAG, streaming per claim, deadline budget, degradation rules, partial status, prompt-cache layout, metrics.
- `llm-client.md`: `LLMClient` interface, tiers cheap/mid/strong, LiteLLM, retry/timeout, fixture/stub/record modes, typed errors.
- `prep-atomize.md`, `link.md` (BM25 + rapidfuzz + neighbors, whole-transcript threshold), `check.md` (group A code checks, verifier fields, verdict table, inference rule, templated explanations), `assembler.md` (`locate()`, drop-unmatched, repair table, rule 1-12 enforcement, copy guard).
- `logging-and-phi.md`: allowlisted fields only, never bodies/text/exception messages with user data; sentinel test.
- Detailed numbers (thresholds, k, window) live only in code constants; docs name the constant and file, not the value, unless the value is a locked decision.

### R6 [OPTIONAL] `docs/decisions/` (ADR format: Status, Context, Decision, Alternatives, Consequences; <=1 page)
Numbering `0001-slug.md`, `README.md` index table. Initial set:
1. Three-stage LLM pipeline plus code assembler
2. LLM returns verbatim excerpts; code computes code-point offsets
3. BM25 + rapidfuzz retrieval, no embeddings/spaCy in POC
4. Synchronous call, 60 s cap
5. LiteLLM with cheap/mid/strong tiers; strong offline only
6. Understatement flagged under `frequency_severity_inflation` (no contract rename)
7. `role` optional: skip role checks, warn, never guess
8. Failure policy: claim stall -> `insufficient_evidence`; section failure -> omitted + partial; nothing usable -> 503/504 (pending O-2)
9. Fixtures replay in CI; live eval opt-in
Open items stay [OPEN] inside the ADR with status `proposed` until confirmed.

### R7 [OPTIONAL] `docs/testing/`
- `README.md` index; `running-tests.md`: `uv run pytest`, `uv run ruff check`, no network/keys; markers if any; what fixtures replay means.
- `evaluation.md`: `uv run python -m eval.run --mode gold|linked|e2e [--replay|--live [--record]]`; per-bucket P/R/F1 + turn-level evidence recall, never one score; fixture refresh procedure; leakage rule; POC gate (`--gate poc`); deferred 60/20/20 split and P0 precision >=0.8.
- `latency.md`: scaled-time latency tests; how to read `Server-Timing`; p50/p95 targets 30-45 s, cap 60 s.
- `pre-release-checklist.md`: tests+lint green; gate passes on recorded fixtures; one live run with owner key (cost shown); schemas unchanged/1.0.0; docs drift test; no PHI in logs sample; CORS origin set.

### R8 [OPTIONAL] `docs/specs/`
Decision: **no new spec file**. Request/response rules are the contract; verdict synthesis table belongs in `implementation/check.md` (implementation detail, tunable). Update `specs/README.md` to say where behavior rules live. Revisit only if a verdict decision table becomes a user-visible promise.

### R9 [OPTIONAL] `docs/examples/`
- `curl.md`: curl against `contracts/examples/request.json` and the error case (oversize or bad version), expected status; response example is `contracts/examples/response.json` (link, not copy).
- `python_client.py` (stdlib `urllib` or httpx): send the example, print verdict per claim by slicing `Array.from`-equivalent (Python str is code points), then print `status` and warnings. Prints no transcript beyond the sliced example (fictional).
- README notes `IAUDIT_LLM_MODE=stub` returns an empty valid audit without a key; real output needs provider keys.

### R10 READMEs and links
- **Core** `backend/README.md`: what it is, requirements (Python 3.12, uv), `uv sync`, env (keys + key vars, link to `reference/backend-api.md`, or `configuration.md` once written), run, `/healthz` check, test, smoke script (`uv run python -m eval.smoke`, 07a), eval pointer only if 07b landed. Under 60 lines.
- **Core** Root `README.md`: add Backend and Contract links.
- **Core** `docs/reference/README.md`: add one index line for `backend-api.md`.
- `docs/README.md`: no new top-level folders, so table unchanged; section READMEs list their files.
- **[OPTIONAL]** `docs/contracts/README.md`: add one sentence linking `../reference/backend-api.md` (HTTP path, status, CORS now live there); **no other edit**; the "HTTP path and CORS stay out of scope" line stays true for the contract itself.
- Frontend index O-03-1 (on `feat/review-ui`, not main): note in the hand-off that it is resolved by `reference/backend-api.md`; do not edit that branch's files.
- `/root/projects/README.md`: verified `grep -n iaudit` returned no row [verified: grep], so add a row (folder, repo URL, one-line description). This is outside the repo; the orchestrator does it as a separate step.

### R11 [OPTIONAL] Drift guard (optional, small)
`backend/tests/test_docs_drift.py`: (a) every `IAUDIT_*` field name from `Settings` appears in `docs/reference/configuration.md`; (b) every HTTP status in the error-mapping table appears in `reference/backend-api.md`; (c) relative markdown links in `docs/` (excluding `agent_files/`) resolve. No network. Skipped if docs dir missing.

## Architecture
Docs only, plus one optional test. Process: after 07, a sub-agent per doc group reads the landed code and writes docs (groups: architecture+implementation; reference+examples+READMEs; concepts+decisions+testing). Each fact is copied from code or a run command, then link-checked.

## Decisions
| # | Decision | Choice | Alternatives | Why |
|---|---|---|---|---|
| D1 | Timing | Write after code lands | Write with each SP | Docs describe what exists; avoids drift while code moves |
| D2 | Spec doc | None; table in `implementation/check.md` | `specs/verdicts.md` | Contract holds normative rules; thresholds are tunable internals |
| D3 | ADRs | `NNNN-slug.md`, short | One long decisions file | Matches placement table, individually linkable |
| D4 | Reference form | Hand-written from `Settings` + drift test | OpenAPI export | FastAPI OpenAPI would not carry contract rules or env vars |
| D5 | Mission wording | Document extension in concepts, flag [OPEN] | Edit mission check 7 now | Scope text is user-owned (AGENTS.md) |
| D6 | Numbers | Name constants, not values | Copy every threshold | Values change with tuning; one owner |
| D7 | Machine index | Add row, orchestrator step | Skip | Global README is the machine index |

## Manual steps
- Approve or reject the mission check 7 wording change (O-1).
- Supply API keys to record live fixtures and measure real latency numbers before docs quote any.

## Risks / Open Questions
- [OPEN] O-1 Mission check 7 says "exaggerates"; behavior also flags "understates". Docs say "reuses `frequency_severity_inflation`" and cite the ADR. Edit `mission.md` and contract descriptions only with user approval; a contract text change is a wording-only same-version change but both schemas and README must move together.
- [OPEN] O-2 Nothing-usable policy (503/504 vs 200 partial) is open in 06 O-1; `backend-api.md` and ADR 8 document the landed behavior, not the proposal.
- [OPEN] O-3 Numbers (latency, token cost, thresholds, F1) are unknown until live runs; docs state "not yet measured" rather than estimates.
- [OPEN] O-4 Final module/file names may differ from PRDs; R2/R5 pointers must be re-verified against the tree.
- [OPEN] O-5 Whether the frontend README/docs should link back to the backend (other branch); owner of `feat/review-ui`.
- [DEFERRED] Generated API docs, documentation site, changelog/versioning of backend docs.
- [RESOLVED: `grep -n iaudit /root/projects/README.md` shows no row; add one].

## Acceptance Criteria
1. Core: every Core file exists (`backend/README.md`, `reference/backend-api.md`, architecture blurb, root README link, `reference/README.md` line). **[OPTIONAL]** every file in R2-R10 exists; each placeholder README lists its files.
2. **[OPTIONAL]** Every `IAUDIT_*` var in `Settings` is in `configuration.md`; every status in the code's error map is in `backend-api.md` (drift test passes if adopted).
3. All relative links in `docs/` (outside `agent_files/`) and READMEs resolve.
4. `git diff` shows no change to `docs/contracts/*.json`, `schema_version` still `1.0.0`, `docs/contracts/README.md` changed by the one cross-link only, `mission.md` unchanged unless O-1 approved.
5. Following only `backend/README.md` (core; `examples/curl.md` is Optional) yields 200 with `IAUDIT_LLM_MODE=stub` and a contract-valid body.
6. **[OPTIONAL]** `concepts/` states: understated severity -> "understates" under `frequency_severity_inflation`; no concern is not correctness; P1 checks are low fidelity with no coverage field; missing role skips checks 3 and 9 with `ambiguous_speaker`.
7. grep of docs for transcript text from the ADHD fixtures finds nothing outside `contracts/examples/` and `backend/eval/`.
8. **[OPTIONAL]** Each ADR has Status, Context, Decision, Alternatives, Consequences; index in `decisions/README.md`.
