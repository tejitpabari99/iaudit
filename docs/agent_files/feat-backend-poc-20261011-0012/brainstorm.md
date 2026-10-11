# Brainstorm: Interview Evidence Auditor backend (POC)

> Copied from branch ab/20261010-214303-6edc3e35. Locked overrides from dev-design are in README.md.

Branch: `ab/20261010-214303-6edc3e35` · Phase: BUILD · Governing docs: [mission](../../mission.md), [audit contract 1.0.0](../../contracts/README.md)

This file records the agreed backend design. It is not a normative spec. Implementation follows it and the contract. The contract wins on request and response shape.

## TL;DR

A Python (FastAPI) `backend/` implements the audit contract in one synchronous call (target 30–45 s, hard cap 60 s). It runs three stages — atomize the rater's reasoning into claims, link each claim to transcript spans, then check — with P0 checks done well (1 support, 2 contradiction, 5 negation, 3 attribution, 9 question-as-answer, 7 frequency/severity) and a single low-fidelity pass for the remaining checks. Quality is measured per bucket on mock data split by interview.

**Understated severity is a concern.** A claim that is materially weaker than the interview is `partially_supported` with `frequency_severity_inflation`. The explanation states that the claim understates the report. The contract type is not renamed for the POC.

## Problem

Raters (human or AI) justify interview-based scores with written reasoning that can drift from what was said: wrong speaker, inflated or understated frequency, a question treated as an answer, a denial reversed. Reviewers need claim-level findings anchored to transcript spans. Research shows final-label accuracy overstates evidence fidelity (AnchorSIPS: label endorsement 0.949–1.000, evidence-link F1 ≤ 0.290), so evidence linking is the hard, valuable part.

## Inputs and outputs (contract 1.0.0)

Owned by the contract session. Implement [docs/contracts](../../contracts/README.md) as-is.

- Request: `transcript[]` (`id`, `speaker`, `role`, `text`, `timestamp?`) and `assessment[]` sections (`id`, `label?`, `score?`, `text`). `role` is always supplied. `label` and `score` are display only and are never audited.
- Response: `claims[]` with non-overlapping assessment spans, one verdict (`supported | partially_supported | contradicted | insufficient_evidence | out_of_scope`), `concerns[]` (12 types), and `evidence[]` spans (`supporting | contradicting | qualifying`). Positions only: Unicode code-point offsets, no quote text, no timestamps.
- Synchronous. No job id. A section that times out yields `status: partial` and `section_not_processed`.
- The contract treats an omitted concern type as "checked, nothing found." P1 is covered by the low-fidelity pass so that reading stays honest enough for the POC.

## Design

### Pipeline

| Stage | What | How | Model tier | Budget |
|---|---|---|---|---|
| 0. Prep | Index turns; tag dialogue acts (question, answer, backchannel, hypothetical, correction) | Rules first, then one cheap LLM call per chunk, in parallel | Cheap | 3–8 s |
| 1. Atomize | Split each section into claims with slots: subject, reported-by, polarity, frequency/severity, timeframe, link to other claims. Mark conclusions and scoring rules `out_of_scope` | One call per section, in parallel | Mid | 5–10 s |
| 2. Link | Shortlist turns (BM25 + embeddings, plus the question before and the clarification after each hit). The LLM returns verbatim excerpts, evidence role, and link type (quote, paraphrase, or inference) | One call per claim, in parallel (cap about 20) | Mid | 8–12 s |
| 3. Check | Verifier call (groups B, C, P1) plus code checks (group A) | One call per claim, in parallel. Cache the transcript prefix | Mid | 8–12 s |
| 4. Assemble | Resolve excerpts to code-point offsets (drop anything not found), enforce contract rules 1–12, validate the schema | Pure Python | — | <1 s |

When a transcript is small enough, linking may send it whole. That size threshold is set by benchmark, not assumed.

### Check groups

| Group | Buckets | Judgment needed | Implementation |
|---|---|---|---|
| A: structure | 3 attribution, 9 question-as-answer, quote existence | Who said it, and was it a question? | Code over linker output. Bucket 3 compares the claim's reported-by slot with the evidence turn's speaker and role. Bucket 9 flags support made only of interviewer turns without a clear respondent endorsement. Fuzzy quote matching uses `rapidfuzz`. The LLM is used only for the experiencer-versus-reporter case. |
| B: meaning | 1 support, 2 contradiction, 5 negation | Does the evidence entail, contradict, or not address the claim? | Verifier call with the taxonomy and a few examples (FacTax style). Polarity is also compared in code. |
| C: amounts | 7 frequency/severity (4 timeframe as a stretch) | Is the claim stronger or weaker than the evidence? | The atomizer extracts the claim value. The verifier extracts the evidence value. Code compares ordinal scales in both directions. |
| P1 quick pass | 4, 6, 8, 10, 11, 12 | Obvious cases only | Extra output fields in the same verifier call |

Inference links (TROVE: the hardest type) must list premise turns and the inferential step. They are `supported` only when every premise is supported and the step is one a neutral reader would accept. Otherwise they are `insufficient_evidence`. Claims that need clinical criteria are `out_of_scope`.

### Atomization example

"She is forgetful because she forgets medicines and skips dinner" becomes three claims:

- "She is forgetful": `out_of_scope` (conclusion). Counterevidence on conclusions is deferred.
- "forgets medicines": checked.
- "skips dinner": checked, with the "because" link checked as bucket 12.

### Understated severity

Locked for the POC. Do not ask for a contract rename yet.

- Compare frequency and severity on an ordinal scale in both directions.
- If the claim is one or more steps **stronger** than the interview, verdict is `partially_supported` or `contradicted`, concern is `frequency_severity_inflation`, and the explanation says the claim inflates the report.
- If the claim is one or more steps **weaker** (the real CDR case: "some difficulty" where the son said "Poor"), verdict is `partially_supported`, concern is the same type, and the explanation says the claim **understates** the report.
- A tie, or a difference too small to be material, is not a concern.
- `suggested_revision` may propose wording closer to the interview. It is never applied.

The type name says "inflation." The explanation carries the direction. A later minor contract bump can rename it to `frequency_severity_mismatch` if reviewers are confused. That bump is not part of this POC.

### Models and tools

- Provider-neutral wrapper (for example LiteLLM). Choose models per tier by benchmark on the dev split:
  - cheap: Haiku 4.5 class
  - mid: Sonnet 5.5 class
  - strong: Opus 5.5 class, offline only (labelling and benchmarking)
- Reference baselines for bucket 1, not live dependencies: MiniCheck-Flan-T5-L (0.8B, MIT) and GPT-4o-mini.
- Prompting: taxonomy plus a few examples everywhere. No live self-consistency. It costs latency, and none of the gathered papers validated it.
- Libraries: `rapidfuzz`, `rank_bm25`, a small embedding model, negation detection (for example negspaCy), `jsonschema`.

### Evaluation

| Mode | Claims | Evidence | A bad score means |
|---|---|---|---|
| Gold | Labels | Labels | The checker is weak |
| Linked | Labels | Linker | The linker is weak, if gold is good |
| End-to-end | Atomizer | Linker | The atomizer is weak, if linked is good |

- Report precision, recall, and F1 per bucket, plus evidence-span recall. Never report one overall score (AggreFact).
- Target: precision ≥ 0.8 on P0 buckets.
- Add a separate stage or model only when it gains at least 0.05 F1 on that bucket at acceptable cost. F1 combines precision and recall. A gain from 0.62 to 0.67 is the minimum that justifies more machinery.
- Plain JSONL plus a script harness. No Phoenix at first.

### Data

- The POC is non-commercial. Usable later, each with its own access step:
  - E-DAIC / DAIC-WOZ (PHQ-8; needs a signed USC licence for academic or non-profit researchers)
  - DementiaBank / ADReSS (free membership)
  - AnchorSIPS (CC BY-NC; evidence-anchored labels)
  - ACI-Bench and MTS-Dialog (CC BY; no ratings; clean controls only)
- Kaggle re-uploads of DAIC-WOZ are excluded. Their "CC0" label conflicts with USC's academic-only terms.
- No public dataset has rater write-ups. Faithful rationales are synthesized from DSM-style templates, a person reviews them, then faults are injected one bucket at a time.
- Planned mock set: about 20 ADHD interviews (adult DSM-5, DIVA-style, with an informant) and about 20 Alzheimer's/CDR interviews. Each interview yields a faithful version, about 5 single-fault versions, and one multi-fault version.
- Split by interview: 60 / 20 / 20 train / dev / locked test. The user's real CDR case is **test only**, once the source file is in the repo.
- The attached CDR transcript was not in this workspace, so it is not checked in. Do not invent its turns. Cleanup rules, when the file arrives:
  - Concatenate the informant interview and the patient interview, in that order (344 turns in the prior review).
  - `Speaker 1` → `clinician1` / interviewer. Informant-interview `Speaker 2` → `son1` / informant. Patient-interview `Speaker 2` → `patient1` / respondent.
  - Drop watermark lines.
  - Keep diarization merges (turns 68 and 166 in the prior review). A claim that relies on one gets `ambiguous_speaker`. Never repair the text.
- Gold labels from that review, accepted by the user, stay the expected test labels. Two real discrepancies: orientation "rarely" versus the son's answers on date and day, and problem solving "some difficulty" versus "Poor" (understated severity).
- The approved synthetic ADHD example for review is below. Generate the rest of the set only after that example is in the repo and the user has the source CDR file to compare against.

#### Synthetic ADHD example (approved)

Fictional. Speakers: `clinician1` (interviewer), `alex1` (respondent), `sam1` (informant).

| # | Speaker | Text |
|---|---|---|
| 0 | clinician1 | Thanks for coming in. I'll ask about attention, first now as an adult, then as a child. Do you often have trouble keeping your attention on tasks at work? |
| 1 | alex1 | Yeah, in meetings especially. If it goes past twenty minutes I'm gone, I'm thinking about something else. |
| 2 | clinician1 | How often does that happen? |
| 3 | alex1 | Most meetings. So three or four times a week. |
| 4 | clinician1 | Do you make careless mistakes in your work, like in emails or reports? |
| 5 | alex1 | Not really. My manager actually says my reports are pretty clean. I double-check everything. |
| 6 | clinician1 | Do you often lose things you need, like keys, phone, wallet? |
| 7 | alex1 | My phone, constantly. Probably every day I'm hunting for it. |
| 8 | clinician1 | And keys? |
| 9 | alex1 | Keys, no. I have a hook by the door. That one I solved. |
| 10 | clinician1 | Do you have trouble finishing tasks, like household projects? |
| 11 | alex1 | Sometimes. I've got a half-painted bathroom right now. Actually, it's been like that a month mostly because we ran out of paint. |
| 12 | clinician1 | Are you easily distracted by things going on around you? |
| 13 | alex1 | Hm. |
| 14 | clinician1 | Let's go back to childhood, ages five to twelve. Did teachers comment on your attention? |
| 15 | alex1 | Oh, constantly. Every report card said "daydreams" or "doesn't finish work." |
| 16 | clinician1 | Did you have trouble with homework? |
| 17 | alex1 | I'd forget it at home maybe once a week. That stopped by high school. |
| 18 | clinician1 | Sam, you've lived with Alex for six years. What do you notice? |
| 19 | sam1 | He forgets things we've agreed on. I'll ask him to pick up groceries and he comes home without them, maybe twice a month. |
| 20 | clinician1 | Anything else? |
| 21 | sam1 | Honestly, I'm the forgetful one with appointments. I rely on his calendar. He's good with appointments. |
| 22 | clinician1 | Alex, do you agree about the groceries? |
| 23 | alex1 | I think it's less than that. Maybe once a month, if that. |
| 24 | clinician1 | Is any of this causing problems at work? |
| 25 | alex1 | I got a comment in my review about seeming checked out in meetings. That's it. I wasn't written up or anything. |

Faithful rationale (`label: "Inattention (DSM-5 A1)"`, `score: "5/9"`). Every factual claim should come back `supported`. The score itself is not audited.

> Alex reports losing attention in most meetings, about three to four times a week. He reports misplacing his phone daily but has a system for keys. He describes some difficulty finishing household projects, although the current unfinished project is partly because they ran out of paint. His partner reports he forgets agreed errands about twice a month; Alex estimates once a month. Childhood report cards repeatedly noted daydreaming and unfinished work. He received a review comment about seeming disengaged in meetings. He denies careless mistakes at work.

Single-fault variants, each its own request:

| # | Injected text | Bucket | Expected result | Evidence turns |
|---|---|---|---|---|
| 1 | loses attention in every meeting, several times a day | 7 | `partially_supported` + `frequency_severity_inflation` | 3 |
| 2 | He makes frequent careless mistakes in reports | 2/5 | `contradicted` + `negation_reversal` | 5 |
| 3 | He frequently forgets appointments | 3 | `contradicted` + `wrong_attribution` | 21 |
| 4 | He is easily distracted by his surroundings | 9 | `insufficient_evidence` + `question_as_answer` | 12–13 |
| 5 | He currently forgets work assignments weekly | 4 stretch | `insufficient_evidence` + `timeframe_mismatch` | 17 |
| 6 | Both Alex and his partner agree he forgets errands twice a month | 11 P1 | `partially_supported` + `conflicting_accounts_flattened` | 19, 23 |
| 7 | The unfinished bathroom shows his difficulty completing tasks | 8 P1 | `partially_supported` + `context_stripping` | 11 |
| 8 | His distractibility caused a negative performance review | 12 P1 | `partially_supported` + `unsupported_causal_link` | 25 |

## Decision log

| Decision | Alternative rejected | Why |
|---|---|---|
| Implement contract 1.0.0 as-is | Our own contract | The other session owns it |
| One synchronous call, ≤ 60 s, parallel per section and claim | Background job | Demo speed |
| Three stages: atomize, link, check | Single long-context judge | TROVE: retrieval 51.18 vs 9.61 F1; failures must be localized |
| Checks grouped by judgment type | One model per bucket, or one call for all | Fewer calls; code handles structural checks |
| P0 = 1, 2 (+5), 3, 9, 7; 4 stretch; P1 via quick pass | All 12 at equal depth | ROI; DiaSumFact macro-F1 ≈ 0.25 for fine-grained types |
| LLM returns excerpts; code computes offsets and drops unmatched ones | LLM returns offsets | The contract needs positions; this blocks invented quotes |
| Model per tier, chosen by benchmark | Claude by default | Cost versus reasoning |
| Taxonomy + few-shot prompting | Zero-shot only, or fine-tuning | FacTax +3.5%; little data for fine-tuning |
| Evaluate in three modes | Four modes | The sample has no rater-cited quotes |
| Precision ≥ 0.8; +0.05 F1 before adding a stage | No targets | Accepted |
| `role` always supplied | Infer roles | User |
| Counterevidence on out-of-scope conclusions deferred | Attach qualifying evidence now | Keep the output simple. Distant denial search is also deferred; nearby denials still count if the shortlist includes them |
| Understated severity is a concern, same concern type, explanation says "understates" | Ignore it, or rename the contract type now | User delegated the call. Renaming needs a contract bump and is not required to flag the case |
| Non-commercial datasets allowed | Commercial-only sets | User |
| Real CDR sample is test only, and is not invented if the file is missing | Use it for prompt examples, or reconstruct the transcript | Honest evaluation; do not invent quotations |
| Diarization merges get `ambiguous_speaker`, never a repair | Repair the merged turns | User accepted the mock review |
| YAGNI items kept as next steps | Drop them from the design | Review them after solution 1 |
| MiniCheck-Flan-T5 and GPT-4o-mini are bucket-1 baselines only | Use either as the live checker | Prove the verifier adds value before depending on them |

## Next steps (YAGNI, review after solution 1)

| Parked | Bring back when | Expected gain | Effort |
|---|---|---|---|
| Dedicated vector DB | Cross-interview search, or slow indexing | Speed | Medium |
| Evidence graph | Buckets 10/11 fail even with good evidence sets | Corrections and conflicting accounts | High |
| Fine-tuning a small checker | A bucket stalls below target after prompt work | Cost and accuracy | High |
| Separate NLI service | LLM cost or latency is too high for cheap checks | Cost | Medium |
| Full slot extraction (time, certainty) | Bucket 4 or 6 is promoted to P0 | Those buckets | Medium |
| Self-consistency for inference | The latency budget allows it | Inference accuracy | Low |
| Dedicated counterevidence retrieval, plus qualifying evidence on conclusions | Output complexity is acceptable | Buckets 8, 10, 11 | Medium |
| Background job API | Audits exceed 60 s | Long transcripts | Medium (contract change) |
| `checks_run` coverage field | P1 low-fidelity output misleads reviewers | Honesty of omissions | Low (minor contract bump) |
| Rename `frequency_severity_inflation` to a direction-neutral name | Reviewers misread understatement flags | Clearer concern labels | Low (minor contract bump) |
| Multilingual input, speech-to-text, diarization | Product decision | — | — |

## Non-goals

- Diagnosis, scoring, or score validation. Protocol or consent review. Real-world truth.
- Frontend.
- Speech-to-text or diarization repair.
- Background jobs or streaming.
- Inventing the missing CDR transcript.

## Open risks

| Risk | Cheapest test |
|---|---|
| Synthetic rationales do not resemble real rater writing | Compare this ADHD rationale with the real CDR rationale once the source file is available |
| Linking recall is too low (AnchorSIPS ≤ 0.29) | Compare gold versus linked mode on the dev split first |
| The 60 s cap is breached on long transcripts | Time the real CDR case (344 turns) and a 3× synthetic transcript |
| P1 low-fidelity pass produces false positives | Measure P1 precision on dev; suppress flags below 0.6 |
| MiniCheck underperforms on dialogue | Bucket-1 baseline on dev before relying on it |
| No licence signer for DAIC-WOZ | Proceed with DementiaBank, AnchorSIPS, and synthetic data |
| Reviewers read `frequency_severity_inflation` as "only stronger" | The explanation must say "understates" or "inflates"; revisit the rename if that fails in review |

## Open questions

1. Who can sign the USC DAIC-WOZ licence as an academic or non-profit researcher? Not blocking. Synthetic data and, when access exists, DementiaBank and AnchorSIPS are enough to start.

## Q&A

- **Contract?** Use `docs/contracts` on `main`.
- **Stack?** Python / FastAPI.
- **Models?** Choose per task tier by cost versus reasoning, then benchmark.
- **Interview length?** Do not assume short.
- **Sync or job?** Sync. Over 1 minute is too long for the demo.
- **P0/P1?** Accepted. P1 goes in the outputs via a quick pass. No `checks_run` field for now.
- **Atomization?** Accepted. Linking is its own stage.
- **Counterevidence on conclusions?** Tabled, including a separate denial search.
- **Role?** Always present.
- **Precision target?** 0.8. Add a stage only for a gain of at least 0.05 F1.
- **Licensing?** Non-commercial.
- **Data?** Mock ADHD and Alzheimer's from DSM-style templates. The user reviews rationales and faults. The real CDR file was not in this workspace.
- **Understated severity?** Flag it. Same concern type. Explanation states the direction. No contract rename in this POC.
- **Mock review?** Accepted, including the two CDR discrepancies and `ambiguous_speaker` on diarization merges.
- **Prompting?** Taxonomy plus a few examples. Not live multi-sample voting.

## Research appendix

### Papers

| Paper | What it compared | What it concluded | Headline number |
|---|---|---|---|
| [FActScore](https://arxiv.org/abs/2305.14251) | Atomic-fact precision of model-written biographies | Binary whole-text judgments are too coarse | ChatGPT 58%; the estimator lands within 2% of humans |
| [VeriScore](https://arxiv.org/abs/2406.19276) | 16 models, 8 tasks, scoring only verifiable claims | Results do not transfer across tasks | Not stated |
| [TROVE](https://arxiv.org/abs/2503.15289) | 11 LLMs tracing provenance, with and without retrieval | Retrieval is essential; inference is the hardest relation | F1 51.18 with retrieval vs 9.61 without |
| [FacTax](https://arxiv.org/abs/2402.12821) | Taxonomy in the prompt vs zero-shot, 10 datasets | The taxonomy helps | +3.5% over ChatGPT zero-shot; +3.8% average with GPT-4o |
| [AggreFact](https://arxiv.org/abs/2205.12854) | Factuality metrics across 9 datasets | No one metric is superior in all settings | — |
| [DiaSumFact](https://arxiv.org/abs/2305.16548) | Detectors on dialogue summaries, 6 error classes | Fine-grained dialogue errors are hard | Macro-F1 ≈ 0.25 |
| [SummaC](https://arxiv.org/abs/2111.09525) | Sentence-level vs document-level NLI | Granularity matters | Balanced accuracy 74.4% (+5) |
| [QAFactEval](https://arxiv.org/abs/2112.08542) | QA-based vs entailment-based checks | Complementary; combining them helps | +14% over prior QA-based metrics |
| [ALCE](https://arxiv.org/abs/2305.14627) | Quality of LLM citations | Citations are often unsupported | Best models lack full support about half the time (ELI5) |
| [AIS](https://arxiv.org/abs/2112.12870) | Human attribution protocol | A common framework for attribution | — |
| [AnchorSIPS](https://arxiv.org/abs/2608.12329) | 7 LLMs on 10K synthetic Mini-SIPS interviews | Final labels overstate evidence competence | Query endorsement 0.949–1.000; evidence-link F1 ≤ 0.290 |

### Tools (LLM-AggreFact averages, 2024)

| Tool | What it is | Score | License |
|---|---|---|---|
| Bespoke-MiniCheck-7B | Grounding checker | 77.4 | CC BY-NC 4.0 |
| MiniCheck-Flan-T5-L (0.8B) | Grounding checker; about 400× cheaper than GPT-4 | 75.0 | MIT |
| AlignScore (0.4B) | Alignment model | 70.5 | MIT |
| Arize Phoenix | Observability platform with a generic LLM-judge hallucination evaluator | No published accuracy | Elastic License 2.0 |

Reference points:

- On LLM-AggreFact, GPT-4o-mini scores 74.0, GPT-4o 75.9, and Claude 3.5 Sonnet 77.2. On the MeetingBank subset, strong LLMs score 79–85.
- Claude prices per million tokens, input/output: Haiku 4.5 $1/$5; Sonnet 5.5 $2/$10; Opus 5.5 $4/$20 ([pricing](https://platform.claude.com/docs/en/about-claude/pricing)). The Batch API halves these.
- Non-Claude prices were not gathered. A benchmark run of GPT-4o-mini should cost dollars, not hundreds. Live Sonnet-class audits of the short sample were estimated under $1 and are not measured yet.

### Datasets

| Dataset | Rating | Size | Access |
|---|---|---|---|
| [DAIC-WOZ](https://dcapswoz.ict.usc.edu/) / E-DAIC | PHQ-8 | 189 sessions / 275 participants (163/56/56 split) | Academic and non-profit only |
| [ADReSS / DementiaBank](https://luzs.gitlab.io/adress/) | AD vs control, MMSE | 156 | Free membership |
| AnchorSIPS | Mini-SIPS items with evidence-anchored turns | 10K synthetic | CC BY-NC |
| [ACI-Bench](https://github.com/wyim/aci-bench) | None | 207 | CC BY 4.0 |
| [MTS-Dialog](https://github.com/abachaa/MTS-Dialog) | None | 1.7K | CC BY 4.0 |

No public ADHD interview dataset with ratings was found.
