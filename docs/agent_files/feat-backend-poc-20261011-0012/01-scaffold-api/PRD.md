---
status: draft
summary: Runnable FastAPI backend/ with POST /v1/audit + /healthz, contract-schema validation, contract-valid errors, env config, async LiteLLM client with fixture/stub mode, PHI-safe logging, and a stub pipeline seam returning an empty-claims response.
date: 2026-10-11
---
# PRD: 01 Backend scaffold + API

Repo/branch: iaudit / `feat/backend-poc` · Depends on: none · Owns: `backend/` (pyproject, `src/iaudit_backend/{api,config,contract,llm,pipeline,logging}`, `tests/`), except stage internals owned by 02-06

## TL;DR
A runnable service that accepts contract 1.0.0 requests, rejects bad ones with contract-valid error bodies, and runs a pluggable pipeline (stub = valid empty audit). Also ships the config object and the async LLM client that stages 03-06 all call, with a no-network fixture client for tests. Schemas are read from `docs/contracts/` in place (single source).

## Problem
Stages 02-06 need a shared home: app, validation, errors, config, one LLM interface, logging rules. Without it each stage invents its own, and the API cannot be tested end-to-end until all land. The frontend expects one sync JSON call and rejects any non-2xx body that is not the error schema. [verified: contracts README "Consumer rules"]

## Goals / Non-Goals
**Goals**
- `POST /v1/audit`, `GET /healthz`, CORS allowlist, no auth.
- Request validated against `audit-request.schema.json` itself, plus rules the schema cannot express.
- Every non-2xx body validates against `error-response.schema.json` (tested, including FastAPI-native errors and 404/405).
- Env-driven settings; LLM client interface (tiers, JSON output, timeout, retry, metrics, cache hint); stub/replay client.
- Pipeline seam + stub; PHI-safe logging.

**Non-Goals**
- Any stage logic (prep, atomize, link, check, assemble) and response-rule enforcement (02, 06).
- Auth, rate limiting (code `rate_limited` exists in schema; never emitted in POC), persistence, Docker/deploy.
- Prompts, model benchmarking, API keys (owner step).
- Contract edits (stays 1.0.0).

## Scope tiers
**Core** (must ship): R1, R2, R3 steps 1-6 and the core of 7 (parse, pipeline call, response validation; body size check = Content-Length when present, else read the body then check length), R4, R5 (all but `SERVER_TIMING`), R6 (tiers, JSON output, per-call timeout, global semaphore, deadline-gated retry, typed errors, `FixtureClient`, `StubClient`/`FailingClient`, single `cache_prefix`, basic per-call metrics), R7, R8 (`StubPipeline`, factory, 504 backstop), R9 logging allowlist, R10 tests for core items. Decisions D1-D12. Acceptance criteria 1-8, 10, 11, and 9 without optional parts.

**Optional (gated: after all core work in every SP)**:
- Streaming body-size cap (R3 step 1 streaming; O-4). Core fallback: check Content-Length, else read the whole body then check length.
- Metrics depth beyond basic per-call latency/tokens: stage span API, `unit` field, `snapshot()` richness (R6 `MetricsCollector`). Core fallback: append-only list of `{call_tag, tier, model, in/out/cache tokens, latency_ms, attempts, ok}` plus a simple `snapshot()`.
- `Server-Timing` header and `IAUDIT_SERVER_TIMING` (R3 step 7, R5 Misc).
- Multiple cache breakpoints, `cache_prefixes` beyond one prefix (R6). Core fallback: single `cache_prefix`.

## Requirements

**R1 Layout/tooling.** `backend/` with `pyproject.toml` (uv, Python 3.12, src layout, package `iaudit_backend`), ruff (lint+format), pytest (+ pytest-asyncio or anyio), `uv.lock` committed. Deps: fastapi, uvicorn, pydantic v2, pydantic-settings, jsonschema (+ `referencing`), litellm, httpx (tests). Declared now so later SPs add only rapidfuzz / rank_bm25. No spaCy/torch/embedding deps. Run: `uv run uvicorn iaudit_backend.app:app`; tests: `uv run pytest`; lint: `uv run ruff check`.

**R2 Contract loader** (`contract.py`). Loads the three schemas from `<repo>/docs/contracts/` (resolved from package path; override `IAUDIT_CONTRACTS_DIR`). Draft 2020-12 (`$schema` verified in file). Builds a `referencing.Registry` keyed by each schema's `$id` (they are bare `audit-request.schema.json` etc.; internal `$ref`s are `#/$defs` only [verified: grep]) and exposes:
- `validate_request(raw) -> list[Issue]`, `validate_response(obj) -> list[Issue]` (08/02 use it as the final gate), `validate_error(obj)`.
- Compiled validators cached at import; a missing/invalid schema fails startup, not first request.

**R3 Request pipeline in the endpoint** (order matters):
1. Body size check (Content-Length when present, else read the body and check its length) > `MAX_BODY_BYTES` → 413 `payload_too_large`. [OPTIONAL: streaming cap that aborts mid-read instead of reading the full body; see O-4.]
2. Parse JSON myself from raw bytes (route takes `Request`, not a Pydantic body model, so FastAPI's own 422 never fires for the body). Not JSON / not an object → 400 `invalid_request`.
3. `schema_version` present and != `"1.0.0"` → 400 `unsupported_version` (checked BEFORE schema validation, because the schema uses `const`, which would otherwise surface as a generic invalid_request). Missing `schema_version` → `invalid_request`.
4. Turn count > `MAX_TURNS` or section count > `MAX_SECTIONS` → 413 `payload_too_large` (checked before full schema walk, on the parsed lists, so a 5k-turn body fails cheap).
5. JSON-Schema validation; any error → 400 `invalid_request`. Message = first N issues as `path: message` (paths and schema messages only; never echo instance values, which may be transcript text).
6. Extra checks: unique turn `id`s; unique section `id`s → 400 `invalid_request` (names the duplicate id? ids are "log only", non-PHI by contract; message names the count and the first duplicated id) [see O-3].
7. Parse into internal Pydantic models; call the pipeline; validate the result with R2 `validate_response`; failure → 500 `internal_error` (never ship an invalid audit). Status 200. [OPTIONAL] On success, when `IAUDIT_SERVER_TIMING=true`, adds a `Server-Timing` header (durations only, no PHI; values come from 06's metrics snapshot; per 06 R5/R8).

`request_id` is echoed on errors whenever the parsed object has a string `request_id` (even if the rest is invalid); otherwise omitted. Response success echo is the pipeline's job but the seam asserts it (rule 10).

**R4 Error mapping** (`api/errors.py`). One `ApiError(code, message, status)` and handlers so that all non-2xx are contract bodies:
| Source | HTTP | code |
|---|---|---|
| Bad JSON / schema / dup ids | 400 | invalid_request |
| Version mismatch | 400 | unsupported_version |
| Size / turns / sections | 413 | payload_too_large |
| Pipeline unavailable (LLM provider down/auth, `LLMUnavailable`) | 503 | unavailable |
| Pipeline overall deadline exceeded (backstop timeout or 06 `DeadlineExceeded`) | 504 | unavailable |
| Any other exception | 500 | internal_error (fixed generic message; trace goes to logs, no body text) |
| Unknown route / method (Starlette 404/405) | 404/405 | other |
| FastAPI `RequestValidationError` (defensive: query/headers) | 422 | invalid_request |
The contract lists 400 and 422 as typical for `invalid_request`; we use 400 for body problems and 422 only for the framework fallback. Handlers also cover `StarletteHTTPException` and bare `Exception`. Every handler output is self-checked against the error schema in tests.

**R5 Config** (`config.py`, pydantic-settings, prefix `IAUDIT_`, optional `.env`; single frozen `Settings` object injected via dependency):
| Group | Vars (defaults) |
|---|---|
| Limits | `MAX_BODY_BYTES` 2_000_000, `MAX_TURNS` 2000, `MAX_SECTIONS` 50 |
| CORS | `CORS_ORIGINS` comma list (default `http://localhost:5173`); no `*` with credentials; methods POST/GET/OPTIONS |
| Models | `MODEL_CHEAP` `claude-haiku-4-5`, `MODEL_MID` `claude-sonnet-5-5`, `MODEL_STRONG` `claude-opus-5-5` (LiteLLM ids, e.g. `anthropic/...` form decided at wiring; provider switch = change string only) |
| Timeouts | `LLM_TIMEOUT_S` per call (default 30), `AUDIT_DEADLINE_S` 60 (hard cap; 06 consumes), `LLM_MAX_RETRIES` 1 |
| Concurrency | `LLM_MAX_CONCURRENCY` 20 (global semaphore in the client; 06 may sub-limit) |
| Mode | `LLM_MODE` = `live` \| `fixture` \| `stub` (default `live`); `FIXTURE_DIR` |
| Misc | `LOG_LEVEL` INFO, `PIPELINE` `stub` \| `full` (default `full`, per 06 R8), [OPTIONAL] `SERVER_TIMING` true (per 06 R5) |
Defaults exist for all; startup logs the non-secret effective config (model names, limits). Keys are read by LiteLLM from standard env vars, never stored in Settings.

**R6 LLM client** (`llm/`). Async-first interface used by every stage:
```python
class LLMClient(Protocol):
    async def complete_json(self, req: LLMRequest) -> LLMResult: ...
# LLMRequest: tier (cheap|mid|strong), system, messages/user, response_model | json_schema,
#   cache_prefix: str|None (hint: stable leading text to mark cacheable; the 1-item case of cache_prefixes),
#   [OPTIONAL] cache_prefixes: list[str] (up to 2 ordered cache breakpoints, each ends an Anthropic cache_control block; per 06 R6), max_tokens, temperature=0,
#   timeout_s|None, call_tag (stage name for logs/metrics, e.g. "link"), fixture_key|None
# LLMResult: parsed (pydantic obj), usage (in/out/cache_read/cache_write tokens), latency_ms, model, attempts
```
- `LiteLLMClient`: `litellm.acompletion` with JSON-schema response format; validates into the Pydantic `response_model`; one repair-free re-ask is NOT done (retry covers transport/429/5xx/invalid-JSON once, then raises). `cache_prefix` becomes an Anthropic `cache_control` block on the system/prefix message; ignored by providers that don't support it (passthrough, not interpreted). Per-call `asyncio.timeout`; global semaphore from R5. Per 06 R8: `acompletion` is called with `num_retries=0` and no fallbacks (hidden LiteLLM retries would bypass the budget); the client's own transport retry is **deadline-gated**: it retries only if `ctx.deadline` leaves at least one call timeout plus margin, otherwise it raises the original error. `CancelledError` is never caught or converted; only the client's own `asyncio.timeout` becomes `LLMTimeout`.
- Errors typed: `LLMTimeout`, `LLMUnavailable` (auth/provider down), `LLMBadOutput` (unparseable after retry). Stages decide the degraded behavior (locked: section omitted / claim insufficient_evidence); the client never swallows.
- `MetricsCollector` (per-request, contextvar or passed object). Core: list of `{call_tag, tier, model, unit, in/out/cache tokens, latency_ms, attempts, ok}` and a simple `snapshot() -> dict` (JSON-able, text-free); 06 reads it for the log line. [OPTIONAL: `unit` field (section index or claim id, per 06 R8) and a stage span API (first start / last end per stage) for the latency budget.] No prompt or output text stored.
- `FixtureClient` (mode `fixture`): deterministic replay keyed by `sha256(tier + call_tag + canonical request)` or explicit `fixture_key`, from JSON files in `FIXTURE_DIR`; missing key raises `FixtureMiss` (loud, never a network fallback). Record helper (opt-in env `LLM_RECORD=1` with live client) writes fixtures; used by 07 and stage tests.
- `StubClient` (mode `stub`): returns a caller-registered canned object per `call_tag` (in-code, for unit tests); also a `FailingClient(exc)` for timeout/failure paths.
- Client is created once in app lifespan, injected via FastAPI dependency; tests override it.

**R7 Internal models vs contract** (`models/`). Pydantic v2 models for request (`Turn.role: Role|None`, `Section`) and response (`Claim`, `Span`, `Pointer`, `Evidence`, `Warning`...) with enums mirroring the contract (`VERDICTS`, `CONCERN_TYPES` in check order 1-12). They are convenience types, NOT the validator: the JSON schema is the authority; a test asserts the enum sets equal the schema's enums, so drift fails CI. Models serialize with `exclude_none` and field names identical to the contract. Pipeline-internal types (claims with slots, shortlists) belong to 03-05.

**R8 Pipeline seam** (`pipeline/__init__.py`):
```python
class Pipeline(Protocol):
    async def run(self, req: AuditRequest, ctx: RequestContext) -> AuditResponse: ...
# RequestContext: settings, llm (LLMClient), metrics, request_id, deadline (monotonic), log
#   + remaining() -> float, call_timeout(stage) -> float|None (None when under MIN_CALL_S), fixture_scope (= request_id; stages pass fixture_key built on it), per 06 R8
```
`StubPipeline` returns `{schema_version, request_id?, status: "complete", warnings: [], claims: []}`. App selects the pipeline by factory in lifespan (`IAUDIT_PIPELINE=stub|full`, default `stub` in SP01, the only pipeline that exists then; 06 flips the default to `full` per R7/R8; tests always pin `stub` explicitly) [RESOLVED: O-5]. Endpoint wraps `run` in `asyncio.timeout(AUDIT_DEADLINE_S + small grace)` → 504 backstop only; 06 does graceful partials inside it.

**R9 Logging.** stdlib logging, JSON-ish key=value lines. Per request one INFO line: `request_id`, `audit_uuid` (server-generated), turn/section counts, body bytes, status, outcome code, total ms, per-tier call count and token sums, model names. Allowed fields are an allowlist in one helper (`log_event(**fields)`; 06 adds its `audit_done` keys, per 06 R8), which rejects/drops any field named like `text|body|transcript|excerpt|message`. No request/response body, no transcript or assessment text, no pydantic `ValidationError` string with `input_value`, no exception messages from user data (log exception class + traceback frames only; `uvicorn` access log default format has no body). Test: send a request with a sentinel string in text, force each error path, assert sentinel absent from captured logs and from error messages.

**R10 Tests** (no network, `LLM_MODE=stub`). Using the example files from `docs/contracts/examples/` and TestClient: 200 on `request.json` (stub response validates against response schema); each error code row of R4; error bodies validate; request_id echo on success and error; CORS preflight allow/deny; healthz; limits at boundary (exactly MAX ok, +1 fails; body size); fixture client hit/miss; retry/timeout behavior with a fake LiteLLM; logging sentinel test; enum-drift test.

## Architecture
```
client -> CORS -> POST /v1/audit (raw Request)
   size -> json -> version -> limits -> jsonschema -> uniq ids -> models
   -> Pipeline.run(req, ctx{settings, llm, metrics, deadline})   [stub now; 02-06 later]
   -> validate_response -> 200   |  any failure -> ApiError -> contract error body
lifespan: Settings, contract validators (fail fast), LLMClient (live|fixture|stub), Pipeline
```
Schemas: read in place from `../docs/contracts` (repo layout is fixed; the frontend uses the same files; a copy would drift). Packaging a wheel/container later must include or mount that dir; `IAUDIT_CONTRACTS_DIR` is the escape hatch.

Async design for 06: everything awaits; the client holds one global semaphore (cap 20 concurrent calls) so 06 can `asyncio.gather` per-section/per-claim tasks without managing sockets; per-call timeout plus the request `deadline` in `ctx` lets 06 compute remaining budget and pass `timeout_s=min(call, remaining)`; metrics are append-only and safe under concurrency (single event loop).

## Decisions
| # | Decision | Choice | Alternatives considered | Why |
|---|---|---|---|---|
| D1 | Schema source | Read `docs/contracts/` in place | Copy into backend/; git submodule | Contract is single source of truth; copying drifts |
| D2 | Body parsing | Raw `Request` + own JSON + jsonschema | Pydantic body model (FastAPI 422) | Contract schema is authoritative; version check must precede schema; exact error codes |
| D3 | invalid_request status | 400 for body, 422 only for FastAPI-native fallback | 422 everywhere | Contract lists both; one predictable code for the UI, 422 stays a rare framework path |
| D4 | Version check order | Before schema validation | After | Schema `const` would hide `unsupported_version` |
| D5 | Layout | src layout, package `iaudit_backend` | flat `app/` | Avoids import-from-cwd bugs in tests |
| D6 | Pydantic models | Convenience mirror, schema is the validator, enum-drift test | Generate from schema (datamodel-codegen); models as validator | No codegen dep; contract wins; drift caught by test |
| D7 | LLM client | Own thin Protocol over `litellm.acompletion` | Call litellm directly in stages; Anthropic SDK | Provider-neutral per locked decision; single place for metrics/retry/fixtures |
| D8 | Retry policy | Default 1 retry on transport/429/5xx/invalid JSON, with the same timeout; none on auth | Exponential multi-retry | 60 s cap leaves no room; failures degrade per stage |
| D9 | Test LLM | Hash-keyed fixture replay + explicit stub + failing client | VCR-style HTTP recording | Provider-independent, no network, tiny files |
| D10 | Concurrency | Global semaphore in client | Per-stage semaphores only | Prevents provider 429 storms; stages can still sub-limit |
| D11 | Default pipeline | `stub` in SP01; 06 flips to `full`; tests pin `stub` explicitly | Wire early | API testable now |
| D12 | Error messages | Schema paths + schema message only, never instance values | Echo pydantic/jsonschema output | PHI: jsonschema messages include offending values |

## Manual steps (human-only)
- Set `ANTHROPIC_API_KEY` (or other provider key) in the backend environment for live mode; none on this machine today.
- Confirm LiteLLM model id strings against the account's available models before first live run.

## Risks / Open Questions
- [OPEN] O-1 LiteLLM id form for Claude (`anthropic/claude-sonnet-5-5` vs bare) and whether `response_format` json_schema is honored for each tier; verify with one live call at wiring; fallback is tool-use/JSON-in-prompt inside the client only.
- [OPEN] O-2 Prompt-cache hint through LiteLLM: `cache_control` on a content block should pass through to Anthropic; verify token-usage fields (`cache_read_input_tokens`) surface in `usage`. Owner: 06 benchmark.
- [OPEN] O-3 Duplicate-id message naming an id: ids are non-PHI by contract intent but could be derived from names; safer default is count-only. Choose count-only unless reviewers want ids.
- [RESOLVED: SP01 default `stub`; SP06 flips to `full`; tests/fixtures always set `IAUDIT_PIPELINE=stub` and never rely on the default] O-5.
- [OPEN] [OPTIONAL] O-4 Streaming body size cap: ASGI servers may buffer; enforce cap while reading chunks. Acceptable for POC.
- [RESOLVED: yes] Schema validation cost on 2 MB bodies: jsonschema runs in a worker thread (`run_in_threadpool`) to avoid blocking the loop; no further optimization.
- [RESOLVED: contract shows no request-level `role` need] Missing `role` is valid input; this SP does no role handling (05).
- [DEFERRED] `rate_limited` emission, auth, request-id generation when absent, multi-worker deployment.
- [DEFERRED] Whether `unavailable` should surface partial LLM outages: 06 decides degradation before an error is raised.

## Acceptance Criteria
1. `uv run pytest` and `uv run ruff check` pass with no network and no API keys.
2. `IAUDIT_PIPELINE=stub uv run uvicorn iaudit_backend.app:app` starts; `GET /healthz` returns 200 `{"status":"ok"}`.
3. With `IAUDIT_PIPELINE=stub` pinned (tests set it explicitly), `POST /v1/audit` with `docs/contracts/examples/request.json` returns 200, body validates against `audit-response.schema.json`, `request_id` echoed, `claims: []`, `status: complete`.
4. Table-driven tests produce each expected HTTP status + code from R4; every error body validates against `error-response.schema.json`; `request_id` echoed when readable.
5. Wrong `schema_version` -> 400 `unsupported_version` (not `invalid_request`); duplicate turn/section ids -> 400; 2001 turns, 51 sections, or >2 MB -> 413 `payload_too_large`; a turn without `role` is accepted.
6. Forced pipeline exception -> 500 `internal_error`; forced `LLMUnavailable` -> 503 `unavailable`; unknown route -> 404 `other`.
7. Schemas load from `docs/contracts/` with no copies in `backend/`; deleting that dir makes startup fail with a clear message.
8. CORS: preflight from `http://localhost:5173` allowed, from another origin not; origins change via `IAUDIT_CORS_ORIGINS`.
9. `FixtureClient` returns recorded JSON offline and raises `FixtureMiss` on unknown keys; `LiteLLMClient` is unit-tested against a faked `acompletion` (timeout, one retry, bad JSON, usage/latency captured, cache_control present when `cache_prefix` set; [OPTIONAL] two `cache_prefixes` yield two breakpoints).
10. Log-capture test: sentinel text in a transcript never appears in logs or error messages on any path.
11. Enum-drift test: model enums equal schema enums for verdicts, concerns, roles, warning codes, evidence roles.
