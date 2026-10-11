---
status: draft
summary: The real async Pipeline replacing StubPipeline - streaming prep/atomize/link/check DAG with a 60 s deadline budget, loss-free cancellation, graceful degradation rules, one shared transcript-first prompt-cache layout with t=0 warm-up, per-request metrics, and a scaled-time latency test plan.
date: 2026-10-11
---
# PRD: 06 Orchestration + latency

Repo/branch: iaudit / `feat/backend-poc` · Depends on: 01-scaffold-api, 02-assembler, 03-prep-atomize, 04-link, 05-check · Owns: `backend/src/iaudit_backend/pipeline/` (`full.py`, `flow.py`, `budget.py`, `state.py`, `warmup.py`, `layout.py`, `guards.py`, `metrics.py`), `tests/pipeline/`, `tests/latency/`; small additive edits to 01 (config, endpoint header, app factory)

## TL;DR
`FullPipeline.run(req, ctx)` wires stages 0-4 as a streaming DAG: prep runs beside per-section atomize; each claim then flows link -> check on its own task the moment its section is atomized (no barrier across sections or claims); the assembler runs last on whatever finished. A hard deadline (60 s, configurable) minus an assemble reserve bounds all LLM work; at that point pending tasks are cancelled and every unfinished unit degrades by the locked rules (section omitted / claim `insufficient_evidence`), with finished results kept. One prompt layout (common system -> transcript -> stage block -> claim block) lets 04 and 05 share a transcript cache, warmed at t=0 while atomize runs, so warm-up costs no latency. Per-request metrics feed a log line, an optional `Server-Timing` header and 07.

## Problem
Stages 02-05 are individually correct but nothing yet guarantees the product promise: one sync call, 30-45 s typical, never past 60 s, a usable answer when parts fail. Naive wiring (stage barriers, `gather` of everything under one `wait_for`, shared task awaits) loses finished work at the deadline, cancels shared tasks by accident, serializes cache warm-up into the Link budget, and can turn a provider outage into an empty audit that reads like "no problems". The frontend waits up to 120 s [verified: shared context], so our cap is a product choice, not a client limit. [verified: brainstorm budget table: prep 3-8, atomize 5-10, link 8-12, check 8-12, assemble <1 s]

## Goals / Non-Goals
**Goals**
- Streaming DAG; p50 on the ADHD example well under target, long transcripts inside 30-45 s, never past `AUDIT_DEADLINE_S`.
- Deterministic degradation map consistent with locked decisions and contract rules 1-12 (esp. rule 11).
- No lost partial results, no leaked tasks, no semaphore leaks on cancellation.
- One cache-friendly prompt layout for 04 and 05; resolves 04 O-5 and 05 O-6.
- Metrics sufficient to tune budgets (per stage, per tier, cache) with no text in logs.
- Switch API to the real pipeline by config; stub stays for tests.

**Non-Goals**
- Stage internals (02-05), prompts, model choice/benchmarks, eval scoring (07), docs pass (08).
- Auth, rate limiting, queueing multiple concurrent audits, background jobs, streaming responses to the client, hedged/speculative duplicate calls.

## Scope tiers
Hackathon build: **Core** is the critical path and ships first. **Optional** items are gated: start only after ALL core work in every sub-project is finished. No core item depends on an optional one.

**Core**
- R1 async fan-out (prep ‖ atomize -> per-claim link -> check -> assemble), global semaphore (20), per-claim/section tasks tracked in `RunState` (no `TaskGroup`), `PIPELINE_MAX_CLAIMS`.
- R2 deadline: 60 s `AUDIT_DEADLINE_S`, `ASSEMBLE_RESERVE_S`, `llm_deadline`, `MIN_CALL_S`, per-call timeout, `PREP_SOFT_TIMEOUT_S`, `CANCEL_GRACE_S`.
- R3 section/claim degradation mapping and useless-result guard (nothing usable -> 503/504/500; D6).
- R4 cancellation and partial-result semantics (write-once `RunState`, supervisor, events not `await task`, never catch `CancelledError`, `num_retries=0`).
- R5 basic metrics only: per-stage first-start/last-end timings, claim/section counters, `deadline_hit`, one `audit_done` log line via the 01 allowlist.
- R7 wiring: `IAUDIT_PIPELINE=stub|full` with default flipped to `full`, core `Settings` keys (all except warm-up/Server-Timing keys), `DeadlineExceeded`.
- R8 sibling changes except cache-related ones (`single-claim` entry points, `atomize_section`, `build_link_context`/`link_one`, `check_one`, deadline-gated retries, `RequestContext.remaining()`).
- Acceptance 1 (basic replay test on ADHD faithful + 8 variants through `FullPipeline`), 2-6, 9, and a basic version of 8 (log + counters, no sentinel text).
- Basic deadline/cancel tests with a small injected delay (plain asyncio sleep in a stub client, tiny `AUDIT_DEADLINE_S`); no latency suite needed.

**Optional (gated: after all core work in every SP)**
- Prompt caching: R6 shared `pipeline/layout.py` cache layout, `LLMRequest.cache_prefixes`, t=0 warm-up call(s) (`warmup.py`, `CACHE_WARMUP`, `CACHE_MIN_PREFIX_TOKENS`, `WARMUP_MAX_WAIT_S`), acceptance 7, decisions D8/D9. Core fallback: 04 and 05 render their own prompts with at most the existing single `cache_prefix`, no warm-up, no cross-stage sharing.
- Latency/load test suite (`tests/latency/`: `LatencyInjector`, scaled time, S1-S10) and live acceptance 10.
- `RequestMetrics.snapshot()` beyond basic timings (tokens per tier, cache ratio, per-call records, `over_budget` flags, remaining ms at assemble), the sentinel-text part of acceptance 8.
- `Server-Timing` header (R5, D12, `SERVER_TIMING`).
- Starvation/priority handling (R1 pending-priority bullet, O-5, priority semaphore).

## Requirements

### R1 DAG and concurrency
```
t0 -> prep task (rules-only `prep()`; [OPTIONAL] soft timeout) ---------------+
   -> warm-up task (best effort, R6)                                       |
   -> per section: atomize_section ──> per in-scope claim: flow ───────────┤ await prep_ready
         (OOS claims: pass-through draft, no calls)                        v
                          claim flow = link_one -> check_one -> st.drafts[claim] (write-once)
supervisor: wait until all done or llm_deadline -> cancel -> collect -> guards -> assemble
```
- Claim flow starts when its section finishes; it waits only for `prep_ready` (an `asyncio.Event`, never `await prep_task`, see R4). Prep is usually done first (rules instant, LLM only for unsure turns), so the wait is normally free.
- Caps: global LLM semaphore 20 (01) is the real limit; stage semaphores (`LINK_CONCURRENCY` 20, `CHECK_CONCURRENCY` 20) stay as 04/05 defined. Per-request claim cap `PIPELINE_MAX_CLAIMS` (default 80, ~20 typical, 3x synthetic = about 60): admission counter; a claim over the cap is not processed, `internal.truncated=true` (02 emits `input_truncated`), status stays `complete` [OPEN O-4].
- Per-claim and per-section tasks are plain `asyncio.create_task`s tracked in `RunState`; no `TaskGroup` (one bug-induced exception would cancel siblings and lose finished work).
- CPU-bound pieces (turn-index build, `assemble`, schema validation) run via `asyncio.to_thread` when > about 50 ms so the loop keeps serving timers and other requests.
- **[OPTIONAL]** Pending-priority: FIFO on the global semaphore means late-stage calls queue behind new link calls. Accepted for POC; measure in S3 [OPEN O-5].

### R2 Deadline budget
All times from `t0` = request receipt (`ctx.deadline` set by 01 = `t0 + AUDIT_DEADLINE_S`, so body parse and validation count).
| Quantity | Default | Meaning |
|---|---|---|
| `AUDIT_DEADLINE_S` | 60 | hard cap (01); startup warns if > 100 so cap + grace + network stays under the 120 s client timeout |
| `ASSEMBLE_RESERVE_S` | 3 | held back for cancel cleanup, assemble (<1 s), schema validation, serialization |
| `llm_deadline` | `deadline - reserve` (57 s) | 06 passes stages `ctx.deadline = llm_deadline`, so 03/04/05 `remaining` math and retry gating use it unchanged |
| `MIN_CALL_S` | 4 | no new call starts with less remaining; unit stalls with reason `deadline` |
| Per-call timeout | `min(stage cap, remaining)` | stage caps (env): prep 10, atomize 25, link 20, check 20; `LLM_TIMEOUT_S` 30 stays the ceiling |
| `PREP_SOFT_TIMEOUT_S` | 10 | prep LLM refinement is abandoned and the rules-only `PrepResult` is used (`degraded_chunks` counted); prep never gates link longer |
| `CANCEL_GRACE_S` | 1 | wait for cancelled tasks to unwind; stragglers are logged and abandoned |
Soft per-stage budgets (elapsed from first call start to last call end in that stage; **observability only, never enforced**): prep 3-8, atomize 5-10, link 8-12, check 8-12, assemble <1 s. Metrics flag `over_budget:<stage>`; sustained overrun is the signal to change tier or prompt, not to cut calls. Critical path = slowest section atomize + slowest claim link + slowest claim check, roughly 10+12+12 = 34 s [estimate].
Retries: client retry and 03/04/05 semantic retries are allowed only if `remaining >= call timeout + margin`; 01 must gate its transport retry on `deadline` too (otherwise a 30 s call plus retry can exceed the cap).

### R3 Degradation map (single source; 02 does the contract work)
| Event | Result |
|---|---|
| Prep chunk bad/timeout | rule fallback inside 03; no output change |
| [OPTIONAL] Prep soft timeout | rules-only `PrepResult`; continue (core: `prep()` is already rules-only, no LLM) |
| Atomize failure (any 03 reason) or section not done at `llm_deadline` | no claims for that section; `section_failures` entry (`timeout`/`deadline` -> `timeout`; others -> `error`); 02 emits `section_not_processed` and `status: partial` (rule 11) |
| Link stalled, check stalled, or claim unfinished at `llm_deadline` | `ClaimDraft.outcome="stalled"` (05 `stalled_draft`); 02 builds `insufficient_evidence` + `other` warning naming the claim (locked). Evidence is dropped, even if link finished |
| Claims over `PIPELINE_MAX_CLAIMS` | omitted; `truncated=true` |
| Unexpected exception in a unit | same as a stall/section failure (reason `error`), class logged, no message |
`status` is never set by 06: 02 sets `partial` iff a `section_not_processed` exists, so claim-level stalls alone stay `complete` with `other` warnings, which rule 11 allows. 06 also unions `LinkResult.ambiguous_turns` into `draft.ambiguous_turns` [RESOLVED: 04 O-1f].

**Useless-result guard (`guards.py`).** The run is *useful* iff (a) at least one section was atomized, and (b) there is no in-scope claim, or at least one in-scope claim finished `outcome=ok`. Not useful -> no body is assembled:
- any unit failed with `unavailable` (provider down/auth, `LLMUnavailable`) -> raise `LLMUnavailable` -> 503 `unavailable`;
- else if any unit failed with an unexpected exception -> `internal_error` (500);
- else (timeouts/deadline/bad output) -> raise `DeadlineExceeded` -> 504 `unavailable`.
Rationale: an all-`insufficient_evidence` or empty audit reads like "nothing wrong"; a retryable error is more honest. Not triggered by a transcript whose claims are all `out_of_scope` (valid). [OPEN O-1 on the 200-partial alternative.] Escalating a section whose in-scope claims all stalled into `section_not_processed` is off by default [OPEN O-2].

### R4 Cancellation and partial-result semantics
1. **Results are written, not returned.** Each flow stores its output in `RunState` (write-once dicts) as soon as it completes; the supervisor never depends on `gather` return values, so cancelling the rest loses nothing already finished.
2. **Supervisor**: `RunState.outstanding` counts live tasks; a section flow registers its claim flows before it exits, so the counter reaches 0 only when everything is done. Supervisor `await asyncio.wait_for(idle.wait(), remaining_to_llm_deadline)`; on timeout cancel every live task, then `asyncio.wait(tasks, timeout=CANCEL_GRACE_S)`. After that, any admitted claim with no draft becomes a stalled draft (`deadline`); any section with no atoms becomes a section failure (`deadline`).
3. **Never catch `CancelledError`.** Stage/flow code uses `except Exception` only; `finally` for cleanup. Retry loops and the LLM client re-raise cancellation. Only the client's own `asyncio.timeout` converts to `LLMTimeout`; an outer cancel passes through as `CancelledError`.
4. **Shared-task trap.** Cancelling a task that directly awaits another task cancels the awaited task. Flows therefore wait on `prep_ready` / `warm_ready` events (or `asyncio.shield`), never on `await prep_task`, so one cancelled claim cannot kill prep for the rest.
5. **LiteLLM.** `acompletion` is httpx-async: cancel closes the connection, the provider may still finish and bill the partial generation [OPEN O-3]. Client must set LiteLLM `num_retries=0` and no fallbacks, because hidden internal retries bypass our budget. Post-call logging callbacks may outlive the task; harmless, verified by the leaked-task test. Semaphore waiters cancel cleanly; test asserts the semaphore value returns to its limit.
6. **Client disconnect.** Starlette does not cancel the handler; the run continues to its deadline. [DEFERRED] disconnect watcher (`request.is_disconnected()` poll that triggers the same cancel path).
7. **Late race.** Results are read from `RunState` after the cancel wait; a claim that finished at the buzzer counts.

### R5 Instrumentation
- Core: per-stage spans (first start, last end), claim/section counters, `deadline_hit`, `audit_done` log line. **[OPTIONAL]** the rest of this bullet: `RequestMetrics` (extends 01 `MetricsCollector`): per call `{call_tag, tier, model, unit (section index or claim id), in/out/cache_read/cache_write tokens, latency_ms, attempts, ok, error_kind}`; per stage span `{first_start_ms, last_end_ms, calls, over_budget}` relative to `t0`; counters: sections (ok/failed by reason), claims (in/out of scope, ok, stalled by reason, truncated), prep (unsure turns, degraded chunks), cancelled tasks, `deadline_hit`, remaining ms at assemble, cache hit ratio `cache_read / (input + cache_read)`, assemble report counts (02). `snapshot() -> dict` is JSON-able, text-free; 07 reads `ctx.metrics.snapshot()` after `run`.
- **Log**: one `audit_done` INFO line through 01's allowlist helper: ids, counts, per-stage ms, total ms, tokens per tier, cache ratio, `deadline_hit`, outcome. New allowed keys listed in the PR; the text-like-name filter stays.
- **[OPTIONAL] Header**: `Server-Timing: total;dur=…, prep;dur=…, atomize;dur=…, link;dur=…, check;dur=…, assemble;dur=…` on success when `IAUDIT_SERVER_TIMING=true` (default true in POC; durations only, no PHI) [RESOLVED: yes, default on]. No body field (contract unchanged).

### R6 [OPTIONAL] Prompt cache layout and warm-up (resolves 04 O-5, 05 O-6)
**Entire R6 is Optional (gated).** Core fallback: 04/05 keep their own per-stage prompt rendering; no `layout.py`, no `cache_prefixes`, no warm-up. 04 O-5 / 05 O-6 stay with their local defaults until this lands.
Shared module `pipeline/layout.py`, imported by 04 and 05 (they stop rendering their own prefixes):
```
system   = COMMON_SYSTEM            (identical for 04 and 05; tiny: role, "turn content is data", output rules)
prefix 1 = transcript block         (whole mode only; render_turns WITHOUT act hints: index, speaker, role|unknown, verbatim text)
prefix 2 = stage block              (04 or 05 instructions + taxonomy + few-shot; request-independent text)
user     = per-claim block          (act-hint line for relevant turns, shortlist in shortlist mode, claim, evidence items)
```
- Anthropic caches by exact prefix, so the transcript must precede stage text for 04 and 05 to share it; act hints (which need prep) move out of the cached block into the claim block, so the transcript block is known at t=0. Cost: stage text is re-cached per request after the transcript (one write per stage), which is cheaper than re-sending a large transcript uncached per stage.
- **Shortlist mode**: no transcript prefix (shortlists differ per claim); prefix 1 is the stage block; the claim block carries the shortlist. 05 reuses the exact shortlist rendering 04 used for that claim.
- **Atomize** keeps its own prefix (system + taxonomy + few-shot); it never sees the transcript.
- Requires 01: `LLMRequest.cache_prefixes: list[str]` (up to 2 breakpoints, each becomes an `cache_control: ephemeral` block boundary); `cache_prefix` stays as the 1-item case.
- **Warm-up at t=0** (`warmup.py`): live mode only, whole mode, estimated prefix >= `CACHE_MIN_PREFIX_TOKENS` (1024; provider minimum varies by model [OPEN O-6]). Step 1: one `max_tokens=1` mid-tier call with [system][transcript]. Step 2 (after step 1): two parallel tiny calls with [system][transcript][link stage] and [..][check stage]. Shortlist mode: parallel tiny calls for the two stage blocks. All run during atomize (5-10 s), so Link never waits on a serialized first call; 04's `LINK_WARMUP` is superseded (default off). Link/check flows wait for `warm_ready` at most `WARMUP_MAX_WAIT_S` (2) if warm-up is still running; warm-up failure or timeout is ignored. Tag `call_tag="warmup"`; skipped in fixture/stub mode (no `FixtureMiss`). Short transcripts (the ADHD example, under 1k tokens) skip warm-up entirely.
- Cache hit ratio is reported per request; the live smoke test asserts `cache_read > 0` on a transcript above the minimum (depends on 01 O-2).

### R7 Wiring and config
- `IAUDIT_PIPELINE=stub|full`; 06 flips the default to `full`. `build_pipeline(settings, llm)` in the 01 lifespan; `StubPipeline` is unchanged and used by API tests. Live mode with no provider key: startup logs a one-line warning (presence only); requests then fail 503 `unavailable`.
- New `Settings` keys (additive): `ASSEMBLE_RESERVE_S`, `MIN_CALL_S`, `CANCEL_GRACE_S`, `PREP_SOFT_TIMEOUT_S`, `PREP_CALL_TIMEOUT_S`, `ATOMIZE_CALL_TIMEOUT_S`, `LINK_CALL_TIMEOUT_S`, `CHECK_CALL_TIMEOUT_S`, `PIPELINE_MAX_CLAIMS`, `CACHE_WARMUP`, `CACHE_MIN_PREFIX_TOKENS`, `WARMUP_MAX_WAIT_S`, `SERVER_TIMING` [OPTIONAL]; `CACHE_WARMUP`, `CACHE_MIN_PREFIX_TOKENS`, `WARMUP_MAX_WAIT_S` are [OPTIONAL] (R6).
- Exceptions: `DeadlineExceeded` (504 `unavailable`, same mapping as 01's backstop); `LLMUnavailable` (503). The 01 backstop `asyncio.timeout(D + grace)` stays as the last line of defence (expected unreachable).

### R8 Interface changes requested of sibling SPs
| SP | Change |
|---|---|
| 01 | `LLMRequest.cache_prefixes` [OPTIONAL]; retry gated on `deadline`; LiteLLM `num_retries=0`; `RequestContext`: `remaining()`, `call_timeout(stage)` (None if under `MIN_CALL_S`), `fixture_scope` (request_id) so stages pass `fixture_key` (07 O-6); metrics `unit` + span API + `snapshot()`; log allowlist keys; `Server-Timing` [OPTIONAL]; `DeadlineExceeded` mapping |
| 03 | `atomize_section(section, roster, ctx) -> SectionAtoms` (batch fn becomes a gather wrapper); `prep(turns) -> PrepResult` (rules-only, no LLM) |
| 04 | `build_link_context(turns, prep, ctx)` (TurnIndex + mode, once) and `link_one(claim, lctx, ctx) -> LinkResult`; use `layout.py` and drop built-in warm-up [both OPTIONAL] |
| 05 | `check_one(claim, link, prep, turns, ctx) -> ClaimDraft`, `stalled_draft(claim, reason)`, `passthrough_oos(claim)`; use `layout.py` [OPTIONAL] |
| 02 | none; 06 supplies `InternalAudit(claims, section_failures, truncated)` |
| 07 | e2e adapter = `FullPipeline.run` + `ctx.metrics.snapshot()` |
[Applied 2026-10-11: R8 asks written into 01, 03, 04, 05 PRDs; 02/07 need none. Conflicts noted as [OPEN] in 01 (O-5) and 04 (O-10).]
If 04/05 keep only batch APIs, 06 calls them with one-element lists; the cost is that their intra-batch logic (warm-up, local semaphores) is per call, which is why the single-claim functions are requested.

## Architecture
```
POST /v1/audit -> 01 validation -> ctx(t0, deadline, metrics) -> FullPipeline.run
   RunState{ prep_ready, drafts{}, atoms{}, section_failures[], outstanding, errors }
   tasks: prep | warmup | section_flow[s] -> claim_flow[c]
   supervisor(llm_deadline) -> cancel -> fill stalled/failed -> guards
   -> to_thread(assemble(req, InternalAudit, cfg)) -> AuditResponse -> 01 schema gate -> 200
```
Typical ADHD timeline [estimate, live mid-tier]: atomize 4-7 s; links from about 5 s in parallel, done about 11-13 s; checks done about 17-21 s; assemble <1 s; total about 20 s. Long transcript: link and check each 8-12 s after atomize 5-10 s, about 30-40 s.

## Decisions
| # | Decision | Choice | Alternatives considered | Why |
|---|---|---|---|---|
| D1 | Stage barriers | Per-claim streaming | Barrier per stage | Slow section/claim no longer delays others; recovers tail latency |
| D2 | Deadline unit | One hard deadline minus reserve; soft stage budgets observe only | Enforce per-stage budgets | Stage times vary by transcript; cutting early wastes finished work |
| D3 | Results store | Write-once `RunState`, supervisor cancels | `gather` + `wait_for` | Preserves partial results; no loss at cancel |
| D4 | Task supervision | Own tracking, no `TaskGroup` | TaskGroup | Sibling cancel on first exception loses work |
| D5 | Shared waits | Events, not `await task` | Await prep task | Awaiter cancel would cancel prep for everyone |
| D6 | Nothing usable | 503/504 `unavailable` (guard) | 200 partial, all sections not processed | Empty audit looks clean; error is retryable [OPEN O-1] |
| D7 | Claim stall status | Stays `complete` with `other` warnings | Escalate to partial | Rule 11 ties partial to `section_not_processed`; locked rule |
| D8 [OPTIONAL] | Cache layout | Common system -> transcript -> stage -> claim, acts out of the prefix | Per-stage prefixes (04/05 default) | One transcript write shared by both stages; prefix known at t=0 |
| D9 [OPTIONAL] | Warm-up | t=0 tiny calls during atomize | Serialize first link call (04) | Zero added latency; Link budget untouched |
| D10 | Prep gating | Soft timeout to rules-only | Wait for LLM refinement | Prep is a hint source, never a blocker |
| D11 | Over-cap claims | Omit + `input_truncated` | Stall them | Fabricating `insufficient_evidence` for unprocessed claims misleads [OPEN O-4] |
| D12 [OPTIONAL] | Timing header | `Server-Timing`, default on | Debug body field; none | No contract change, no PHI, visible in devtools |
| D13 | Hedged duplicate calls | Not in POC | Hedge slowest calls | Cost and cap pressure; revisit with p95 data |
| D14 | Cap vs client timeout | Keep 60 s | Raise to about 90 s | Locked product target; configurable, startup guard < 100 |

## Manual steps (human-only)
- Live latency/cache runs and CDR timing need a provider key (01 owner step). The real CDR transcript (344 turns) does not exist in the repo; its timing run is deferred and never uses invented turns.

## Risks / Open Questions
- [OPEN] O-1 Nothing-usable policy: 503/504 vs 200 partial-all-sections. Confirm with the frontend owner; the contract allows both.
- [OPEN] O-2 Escalate a section to `section_not_processed` when all (or >= 50 percent of) its in-scope claims stalled. Default off (locked claim-level rule); decide after S5 looks.
- [OPEN] O-3 Does a cancelled non-streaming Anthropic call stop billing at disconnect? Measure tokens billed vs received on a forced cancel (live, owner key).
- [OPEN] O-4 Over-cap claim policy and a cap value (80) are guesses; admission order is first-come, so truncation is non-deterministic only in extreme inputs.
- [OPEN] O-5 FIFO semaphore may starve check calls behind a link wave; alternative is a `priority` field in 01's client. Decide from S3 stage overlap numbers.
- [OPEN] O-6 Provider minimum cacheable prefix and whether LiteLLM surfaces `cache_read_input_tokens` (01 O-2); until verified the warm-up is a no-risk no-op.
- [OPEN] O-7 Stage call timeouts (10/25/20/20) and `ASSEMBLE_RESERVE_S` 3 are estimates; tune on p95 from live runs.
- [OPEN] O-8 Event-loop sharing: one slow audit and a second concurrent one share the global semaphore and loop; POC is single-user. Add an in-flight audit cap only if needed.
- [DEFERRED] Client-disconnect cancel, hedged calls, priority semaphore, per-section escalation, queueing.
- [RESOLVED: sharing via common system + transcript-first, whole mode only; shortlist mode per-stage] 04 O-5 / 05 O-6.
- [RESOLVED: 06 unions ambiguous turns] 04 O-1(f).
- [RESOLVED: Server-Timing header, default on].

## Acceptance Criteria
All offline. Core tests use a small injected delay and tiny deadline in stub/fixture clients; the full scaled-time suite below is **[OPTIONAL]**.
1. Stub/fixture full pipeline on the ADHD faithful + 8 variants returns the 07 expected verdicts/concerns through `FullPipeline` (not stub), response valid against the schema, `request_id` echoed.
2. Streaming: with two sections where B atomizes slowly, A's first link call starts before B's atomize ends; peak in-flight LLM calls <= 20.
3. Failure maps: one link timeout -> that claim `insufficient_evidence` + `other` warning naming it, status `complete`; one atomize failure -> `section_not_processed` + `partial`, other sections intact.
4. Deadline: inflated latencies make check overrun; response returns within `AUDIT_DEADLINE_S` (+0.5 s scaled), finished claims keep real verdicts, unfinished get the stall rule, `deadline_hit=true`, no leaked tasks (`asyncio.all_tasks()` equals baseline), semaphore value restored.
5. Shared-task trap test: cancelling one claim flow does not cancel prep or sibling flows.
6. Guard: all calls `LLMUnavailable` -> 503; all timeouts -> 504; unexpected exception everywhere -> 500; all-out-of-scope assessment -> 200.
7. **[OPTIONAL]** Cache layout: link and check requests in whole mode share byte-identical `system` + transcript prefix; the warm-up request prefix equals it; a prefix-cache fake shows hit ratio above 0.8 on the 3x run and below that with warm-up off; short transcripts skip warm-up; fixture mode never warms up.
8. Metrics: core = stage spans, counters, `deadline_hit` and `audit_done` log line present. **[OPTIONAL]** `snapshot()` with tokens, cache ratio; `Server-Timing` present; sentinel text never appears in logs, metrics or header.
9. Config: `IAUDIT_PIPELINE=stub` serves `StubPipeline`; default serves `FullPipeline`; D > 100 warns at startup.
10. **[OPTIONAL]** Live (opt-in, owner key): ADHD e2e p50/p95 over 5 runs recorded in the report; target p95 < 45 s; cache_read > 0 on an above-minimum transcript.

## [OPTIONAL] Latency / load test plan (`tests/latency/`)
Entire section is Optional (gated). Core uses the small injected-delay tests in Acceptance Criteria instead.
- **Harness**: `LatencyInjector(inner)` wraps `FixtureClient`/`StubClient`; per `call_tag` lognormal latency (p50/p95 start: prep 1.5/3, atomize 5/9, link 4/8, check 4/8 s), failure injection (timeout, unavailable, bad output by rate or by unit id), records start/end per call. **Scaled time**: `AUDIT_DEADLINE_S`, reserve, caps and latencies all multiplied by one factor (default 1/50), so 60 s becomes 1.2 s and ratios hold; no new dependency. Nightly/opt-in run at factor 1.
- S1 ADHD faithful + 8 variants with injected latencies (total, per-stage spans vs soft budgets).
- S2 section skew (pipelining proof). S3 3x synthetic long transcript (generated from a fictional "Jordan" template, about 78 turns, about 60 claims; not ADHD or CDR text): total vs 45 s, peak concurrency, FIFO starvation, token totals. S4 failure injection matrix (link/check/atomize/prep). S5 deadline overrun. S6 outage matrix (guard). S7 cancellation hygiene (CancelledError delivery, semaphore, leaked tasks, shared-task trap). S8 cache simulation. S9 metrics/log hygiene.
- S10 real CDR (344 turns): deferred until the file exists; test-only, never in repo, prompts, fixtures or tuning; reports timing and cache stats only.
