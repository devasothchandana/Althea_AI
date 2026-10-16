# Final phase — Integrate the three completed workstreams

**Give this entire document to the integration AI agent after all three handoffs are ready.** Complete the working local demo by connecting the components. The three component folders must already exist; inspect their HANDOFF.md files and code before changing anything. Do not declare integration successful from import checks alone.

## Responsibility and prerequisites

You own `integration/`, root setup files, unified settings, dependency resolution, the real DashboardBackend, job orchestration, cache invalidation, persisted request accounting, integration tests, and final documentation. You may make small documented compatibility fixes inside component folders only during this final phase; preserve their public APIs and tests. Do not replace an incomplete component with hardcoded sample outputs.

Read each brief's frozen contract. Compare `_contract.py` source hashes; resolve any deviation against the supplied v1 contract before wiring. Check each package's offline tests and public imports. If a component fails, identify and repair the concrete integration-blocking fault; do not paper over it with fake data in live mode.

## Final repository deliverables

```text
althea-ai/
  team_1_ingestion/
  team_2_analysis/
  team_3_evidence_ui/
  integration/
    pyproject.toml  app.py
    src/althea_app/
      __init__.py  __main__.py  settings.py  backend.py
      pipeline.py  budget.py  cache_keys.py  validation.py
    tests/
  requirements-dev.txt   # Tested compatible versions + editable packages
  .env.example  .gitignore  README.md
  DEMO_SCRIPT.md  ACCEPTANCE_REPORT.md
  data/                 # Ignored runtime files
```

Do not move all three projects into a new monolith just to integrate them. Install the four packages together; use plain Python imports and frozen dictionary interfaces. No HTTP microservices, Docker, message broker, Redis, FastAPI, Celery, or cloud deployment is required.

## Step 1 — Establish one environment

Use Python 3.12. Resolve a single compatible dependency set across the handoffs; run the three test suites together before and after any changes. Root requirements-dev.txt uses exact tested third-party versions and editable references to all four folders. Do not copy mutually incompatible pins blindly. Include google-genai, Pydantic 2, Streamlit, pytest, and the packages actually used; avoid heavyweight ML dependencies.

Root PowerShell setup must work from the repository root:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements-dev.txt
python -m pytest team_1_ingestion/tests team_2_analysis/tests team_3_evidence_ui/tests integration/tests
python -m streamlit run integration/app.py
```

Also provide a console entry through `python -m althea_app`, with `--help`, `--mode`, `--input`/`--sample`, `--verify`, and `--output`.

## Step 2 — Settings, credentials, and live preflight

Only the final application loads `.env`; libraries receive settings. Support environment variables `GOOGLE_CLOUD_PROJECT`, `GOOGLE_CLOUD_LOCATION`, `ALTHEA_TRANSCRIBE_MODEL`, `ALTHEA_TEXT_MODEL`, `ALTHEA_GROUNDING_MODEL`, and `ALTHEA_DATA_DIR`. Map them to the exact frozen settings keys. Default location global, local data directory beneath repository, and sample mode in the UI. Validate numeric limits and model identifiers early. Resolve data_dir to an absolute path. Preserve a separate data namespace/mode in caches so fixture artifacts can never satisfy live requests.

Use Google Cloud ADC, e.g. the user's existing `gcloud auth application-default login` setup, if available. Document enabling the necessary API and project permissions, but do not create paid resources, purchase credits, enable billing, or claim the ₹1,000 credit covers a SKU without checking the account's credit terms. Before a live run verify the selected models/endpoints in the user's project. Google Cloud transcription and Developer API transcription differ in SDK surface/limits; use the handoff's actually tested adapter.

Provide a preflight command that checks local dependencies/configuration without spending API credits. A separately explicit live-smoke option may make a minimal accounted request. Missing authorization is a clear live limitation, not an excuse to block the offline demo. Do not automatically route Vertex failures to AI Studio, Groq, or another paid provider.

## Step 3 — Backend contract and real orchestration

Implement `LiveBackend(settings)` with every DashboardBackend method. Instantiate Repository explicitly, then ingestion, analysis, verification services. `integration/app.py` calls Person 3's `render_app({'Sample demo': fixture_factory, 'Live analysis': live_factory})`. The live factory must be lazy; launching Sample demo does not initialize a cloud client or demand credentials.

`run(source, verify, progress)` performs:

1. Validate InputRequest; create UUID job with running Snapshot and persist it before reserving attempts.
2. Obtain transcript through ingestion cache/service, validate and persist the immutable artifact and active Snapshot.
3. Obtain analysis through analysis cache/service; validate exact source spans and references and persist it.
4. If verify false, do not call verification; set verification null, retain disagreements as 'not checked', complete the job.
5. If verify true, select at most four eligible pairs by `(priority, pair_id)` and reserve allowed distinct pairs transactionally. Call verification for allowed IDs; no eligible pairs yields a valid empty Verification with zero calls.
6. Merge verification with existing successful results (see below), persist state, and return complete/partial Snapshot. Publish progress using plain stage labels; callbacks must not start additional work.

Create `JobBudget(repository, job_id, max_attempts)` implementing consume by Repository.consume_attempt. All providers share this instance. A new Streamlit rerun/backend instance resumes the persisted counter; it never resets it. Track completed/failed attempts in safe events and report per-session changes separately from cached artifact history. Prevent simultaneous pipeline execution for the same job in the local app; atomic counters remain a second line of enforcement.

`load` and `list_jobs` are pure reads. `save_review` calls Repository.review_transcript. `reanalyze` uses the active reviewed transcript without retranscribing or modifying its source. `verify_pairs` validates eligibility, reserves cumulative distinct pair slots, checks per-pair caches, and merges results. `refresh_verification` explicitly bypasses/invalidate selected pair evidence caches, keeps all other results, consumes new attempts, and retains the same distinct-pair reservations. Budget exhaustion never resets automatically. A user can start a new job explicitly, with a new budget; the UI should make it clear it is a new analysis.

### Merging evidence results precisely

VerificationService returns a complete accounting of eligible pairs for its individual request. The backend accumulates successful results for the current analysis_id only. Merge keyed by pair_id; a fresh successful result replaces that pair's previous result/evidence/grounding attribution. Retain completed results and native attribution for untouched pairs. Recompute unverified as all eligible pairs without an active result, using the most specific current reason (provider_error/budget_limit/missing_context before not_selected). An unsuccessful explicit refresh removes that pair's active result and attribution and marks it unverified with the failure reason; keep the historical artifact accessible in stored history but do not display it as freshly verified. Drop orphaned evidence and recompute verification_id. Never merge data from a previous analysis_id. Preserve meaningful issues and deduplicate them.

## Step 4 — Correct cache behavior

Use Repository cache; do not rely on Streamlit's decorator cache for billable side effects. Exact stage key inputs:

| Stage | Include in key |
|---|---|
| Transcription | mode, schema, source byte hash, language, transcribe model, actual normalization/transcription config version |
| Analysis | mode, schema, transcript_id, text model, extraction/comparison/normalization versions, shortlist/claim cap |
| Pair verification | mode, schema, analysis_id, pair_id, grounding/judge models, retrieval/adjudication versions, evidence format version |

Local path, user-facing job title, timestamps of execution, and remaining attempt budget are not semantic inputs. Cache only successful validated artifacts. Text/speaker review invalidates downstream content identity; title changes do not. Changing model/prompt/normalization settings must cause a cache miss. Pair caches permit incremental checking without re-running all earlier pairs. Cache hits must report no new API attempt. Historical evidence remains dated; refresh is explicit.

## Step 5 — State and error handling

Transcription/analysis failure produces failed Snapshot, retained input/upstream artifacts, a clear stage issue, and no automatic downstream call. Evidence failure produces partial Snapshot with usable transcript/claims/disagreements and export. A source-backed but inconclusive verdict is a completed assessment; insufficient evidence is not a system crash. A valid empty claim ledger or zero eligible disagreements produces complete status and informative empty states. Distinguish failed provider access from 'no evidence found'.

Validate stale IDs before every downstream stage and before displaying loaded jobs. Do not repair stale pair references by guessing. Interrupted running jobs are marked partial if valid analysis exists, otherwise failed, on explicit recovery or next startup, with an interruption issue; do not resume billable calls automatically. Do not re-run jobs just because the app restarts.

## Step 6 — Integration tests and realistic evaluation

Required automated tests with fake providers/temp databases:

1. Reference sample end to end: eight turns, seven factual/ambiguous claims, three pairs, one eligible disagreement, both budget claims refuted by synthetic evidence.
2. Launch/use fixture UI without Google credentials/network; external attempt count stays zero. Fixture and live cache namespaces cannot mix.
3. Fake-live audio -> transcript -> analysis -> verification; verify interfaces, persisted IDs, and exact stage order. Mock provider adapters, not the whole backend. Assert shared budget increments across actual fake-provider attempts.
4. Re-run same content/config: cached stages; no new provider call. Change prompt/model and assert relevant misses. New job may reuse cache, but counters belong to that job and only new attempts increment them.
5. Review one number/name/time: new transcript ID, old artifacts retained, active analysis/verdicts cleared, no stale UI cards, explicit reanalysis required.
6. Incremental checking preserves prior results; failed refresh marks only that pair unverified; each eligible pair appears once in results/unverified. Evidence matches each pair after merge.
7. 20-attempt hard cap across failures/restarts, four-distinct-pair cap, transaction race safety, and no nested retry multiplication.
8. Provider auth/rate-limit/schema failure, missing ffmpeg, overlong/oversized audio, no claims, no contradictions, no evidence, broken reference, imported path escape, source URL validation, and safe exports.
9. UI rerun/filter/tab/download does not invoke a provider. Job history persists across application restarts. Interrupted jobs do not auto-resume paid work.

Live acceptance needs a real 1–5 minute English two-speaker recording whose use is authorized, manually checked for speaker turns and key numbers. Test one real upload and one real evidence request if credentials/budget permit. Report actual timings, calls, model IDs, and observed errors. A private pre-recorded human script is acceptable; do not misrepresent synthetic text fixtures as audio/model accuracy testing.

For evaluation, prepare at least 3 short English examples (or clearly report the smaller set), including: a real same-scope conflict; apparent conflict from different years/definitions; and compatible normalized values/opinions. A human reviews the gold ledger and relations. Report claim extraction precision/recall, disagreement precision/recall, correct speaker attribution on checked claims, numeric transcription errors, and citation relevance/support on the assessed claims. Keep gold labels separate from outputs; synthetic unit fixtures do not establish these metrics. Record sample size and denominator; no unsupported percentage claims. Accuracy thresholds should be agreed after this baseline, not invented as a guarantee in the report.

## Step 7 — Demo and delivery

DEMO_SCRIPT.md gives a 3–5 minute presentation: start labelled fixture sample; show exact quotes and numeric normalization; show different-year pair excluded; show both-refuted possibility; demonstrate one real upload if live access passed; show cited evidence with retrieval time; demonstrate transcript review invalidating results; explain one honest limitation. Keep an explicit saved sample fallback for an unreliable network, always labelled. A cached real result retains its real provenance and retrieval date; it must not be described as a fresh live run.

Root README includes setup, ffmpeg/ADC instructions, exact launch/test commands, data flow/ownership, sample vs live behavior, budget cap limitations, credit eligibility caveat, data persistence, and troubleshooting. .gitignore excludes .venv, .env, credentials, data/, generated media/DBs, raw provider responses, and caches. The example fixture is deliberately fictional and may be committed.

ACCEPTANCE_REPORT.md is a table of each required behavior with pass/fail/not-run, command/evidence, and limitations. Separate offline automated checks, UI visual checks, live provider smoke, and real-data accuracy evaluation. Deliver the runnable app even if live access remains blocked, but label it 'offline integration complete; live verification pending' when appropriate. Never call the whole project fully complete if the requested live path has not actually been exercised.

## Final acceptance checklist

* All three folders integrate without circular imports or changed wire names.
* One environment installs, all offline suites pass, fixture app works offline.
* User can upload an allowed clip in live mode, inspect/correct the transcript, inspect claims/disagreements, request evidence, and export/save results.
* Grounded sources are linked accurately; no invented quotation, forced winner, or confidence percentage.
* Cache, corrections, partial failures, persistent attempt limits, and incremental verification work as specified.
* Live smoke/evaluation gaps are explicitly reported; no fixtures concealed as live results.
* Setup and presentation instructions are complete enough for another group member to run the demo.

---

## Shared project specification — frozen for all three people

### Scope and decisions

Build Althea AI as a local, single-user Python demo for prerecorded English audio, maximum **300 seconds**, normally **two speakers**. No local speech, embedding, or NLI models are required. Hosted model inference is acceptable. Python performs validation, arithmetic, persistence, and UI work. Use Google Cloud/Vertex-compatible hosted Gemini APIs where available, subject to the user's credit eligibility. Do not assume Google AI Studio billing and Vertex credit eligibility are interchangeable.

The product finds factual claims, identifies **potential disagreements**, and assesses each claim against retrieved evidence. It must preserve uncertainty. It does not select a debate winner or claim to provide definitive truth. Both opposing claims may be refuted. Opinions, rhetoric, questions, and future predictions are excluded from the first version's factual ledger.

Required experience: audio or transcript input; labelled speaker timeline; editable transcript and speaker display names; claim ledger; disagreement cards; optional evidence check; citations; saved jobs; JSON/CSV export. Include a clearly labelled offline sample. Graph visualization, YouTube input, streaming, authentication, deployment, mobile polish, multiple languages, fine-tuning, and multi-user concurrency are outside the required demo. Do not quietly add them.

### Ownership and repository layout

```text
althea-ai/
  team_1_ingestion/       # Person 1: package althea_ingestion; audio + SQLite repository
  team_2_analysis/        # Person 2: package althea_analysis; extraction + comparisons
  team_3_evidence_ui/     # Person 3: package althea_evidence_ui; verification + Streamlit
  integration/           # Final phase: package althea_app + real orchestration
  data/                  # Runtime files; ignored by Git
```

Each person writes only inside their assigned folder. Each folder is a separately installable package with `pyproject.toml`, `src/<package>/`, `tests/`, `fixtures/`, `.env.example`, `README.md`, and `HANDOFF.md`. UI workstream also owns its standalone `app.py`. Use Python 3.12, Pydantic 2, pytest, and the current `google-genai` SDK. Each person records the exact successfully tested dependency versions. Integration resolves a single tested dependency set. Do not invent package versions or use legacy SDKs simply because a remembered example does.

Libraries exchange JSON-compatible Python dictionaries, **not** each other's Pydantic instances, SDK responses, database connections, or DataFrames. This makes each workstream independently testable. No workstream imports another workstream during parallel development. All public outputs must pass the contract and semantic checks below. Each package may keep an identical private `_contract.py` copy of the frozen contract. Its class names stay private; the wire dictionaries are the integration boundary. Do not publish multiple competing packages named `althea_contracts`.

### Frozen runtime settings

All service constructors receive an ordinary settings dictionary. Never read secrets or construct cloud clients at import time. Required keys/defaults:

| Key | Meaning/default |
|---|---|
| `mode` | `fixture` or `live`; fixture must make zero network calls |
| `project_id` | User's Google Cloud project; required only in live mode |
| `location` | `global`; confirm selected model availability |
| `transcribe_model` | `gemini-3.5-transcribe`, configurable |
| `text_model` | `gemini-2.5-flash`, configurable |
| `grounding_model` | `gemini-2.5-flash`, configurable |
| `data_dir` | Absolute path to the local runtime directory |
| `max_audio_seconds` | 300; validation rejects longer input |
| `max_upload_bytes` | 20 * 1024 * 1024 |
| `max_claims` | 30 |
| `max_candidate_pairs` | 60 |
| `max_display_conflicts` | 8; all assessed pairs remain available in JSON |
| `max_verify_pairs` | 4 per job, cumulative distinct pairs |
| `max_provider_attempts` | 20 per job, cumulative including failed attempts/retries |
| `max_output_tokens_extraction` | 8192 |
| `max_output_tokens_comparison` | 8192 |
| `max_output_tokens_retrieval` | 2048 per pair |
| `max_output_tokens_adjudication` | 2048 per pair |
| `request_timeout_s` | 90 |

Use Application Default Credentials for live Google Cloud calls; `.env.example` contains identifiers, model settings, and paths only. Do not require a checked-in credential file or switch billing providers automatically. Keep text, transcription, and grounding clients separate so a later model/location change is contained. Verify the provider's current SDK configuration and output format against the official links at the end of this brief.

### Provider seams and request accounting

Implement small, dependency-injected providers so offline tests can supply fakes. Services accept a shared duck-typed `budget` object. It exposes `consume(stage: str) -> None`, which reserves one provider attempt atomically or raises an exception with `code='BUDGET_LIMIT'`. Call it immediately **before every** external model attempt, including retries and schema repairs. The final backend supplies a persisted job-wide budget; a standalone fake supplies the same behavior in memory. Successful cached reads do not consume it. This is an attempt cap, not a guaranteed rupee spending cap.

Maximum three attempts per logical provider operation, within the shared budget: original attempt plus at most two retries for 429/5xx/transient transport failures, with bounded exponential backoff and jitter. No retry on bad credentials, unsupported model/configuration, validation failure, or other permanent 4xx errors. One schema-repair call per logical text operation is permitted **within the same three-attempt ceiling**, with no extra hidden loop. Configure SDK retries so SDK and application retry loops do not multiply. Preserve usage when available; unavailable token counts are null, never fabricated zero. Cache hits retain the original artifact's usage but the UI must distinguish historical artifact usage from new calls this session.

Public service failures raise a `RuntimeError` subclass carrying `code: str`, `retryable: bool`, `stage: str`, and a safe user-facing message. Integration maps this to `Issue`; it does not depend on the concrete exception class. Safe codes: `INVALID_INPUT`, `AUDIO_TOO_LONG`, `AUDIO_TOO_LARGE`, `FFMPEG_MISSING`, `PROVIDER_AUTH`, `PROVIDER_UNAVAILABLE`, `RATE_LIMIT`, `PROVIDER_RESPONSE_INVALID`, `BUDGET_LIMIT`, `CONTRACT_INVALID`, `NOT_FOUND`, `STORAGE_ERROR`. Never put credentials or raw private audio in logs. Ordinary no-claim/no-disagreement/no-evidence outcomes are valid results, not exceptions.

### Wire-format rules beyond the type definitions

1. Schema version is exactly `1.0.0`. Include every declared field; use explicit nulls and empty arrays. Reject unknown fields, non-finite numbers, duplicate IDs, and broken references. Serialize with `model_dump(mode='json')`. Preserve Unicode text exactly.
2. Timestamps in audio are numeric seconds; character offsets are Python Unicode string indexes, start-inclusive/end-exclusive. Wall-clock timestamps are RFC3339 UTC ending in `Z`; publication dates are `YYYY-MM-DD` or null. Validate those formats beyond the base types.
3. Speakers are stable IDs `speaker_01`, `speaker_02`, etc.; speaker display names are editable. Segment IDs are `seg_0001`, etc., in chronological order. Diarization IDs do not establish real-world identity. Allow overlapping intervals; require `0 <= start_s < end_s <= duration_s + 0.25`. Allow 1–4 speakers; warn above two, reject above four for this demo. Never force overlapped speech into a false non-overlapping sequence.
4. Every claim has at least one source span. Require exact equality `segment.text[start_char:end_char] == quote`. All referenced segments must belong to the claim's speaker. A proposition can resolve pronouns only when the transcript actually supplies the referent. Missing dates/entities remain null and appear in `missing_context`; do not infer them from today's date, search results, or general knowledge.
5. Claims are one checkable proposition each. Assign `cl_0001`, etc., in source order. Preserve qualifiers such as 'at least', 'about', and 'per capita'. `checkability='checkable'` needs an identifiable proposition and relevant scope; ambiguous claims use `needs_context`. Empty claims are valid.
6. Decimal numbers cross the boundary as finite plain decimal strings, not floats or scientific notation. Use `Decimal` for arithmetic. A quantity with operator `range` has null `value`, non-null `lower`/`upper`, and `lower <= upper`; all other operators have a non-null `value` and null bounds. Canonical units include `currency`, `percent`, `percentage_point`, `count`, `m`, `kg`; another explicit physical unit is allowed. Currency must be a three-letter code when unit is `currency`, otherwise null. Bare `$` without sufficient context is ambiguous, not automatically USD. Normalize million/billion/k/lakh/crore through explicit parsing; preserve the raw text. Do not automatically convert currencies, nominal/real prices, or reporting periods.
7. Candidate pairs are cross-speaker only, unique unordered claim pairs; store `claim_a_id < claim_b_id` lexicographically. Pair IDs are `pair_0001`, etc., sorted by the claim ID tuple. Priority 1 is highest. A pair flagged `direct_conflict` or `numeric_discrepancy` is a potential disagreement, not proof that either claim is false. No probability/confidence percentage is part of this contract.
8. Deterministic numeric comparison requires matching subject, metric, time, geography, population, measurement basis, unit/currency, and compatible operators. Null scope is unknown, not a wildcard. Only exact `eq` quantities may use simple subtraction. Define `numeric_difference = abs(b.value - a.value)` in canonical units only for safely comparable `eq` values; otherwise null with a null difference unit. Different values alone do not prove a contradiction. Approximate/bounded/range claims require reasoning or abstention. A percent change and a percentage-point change can both describe the same event.
9. `Evidence.url` must be HTTP(S), obtained from the provider or a labelled synthetic fixture. Only a provider-returned source passage is `retrieved_excerpt`. A provider-generated answer tied to source grounding metadata is `grounded_summary`, never a quotation from the website. `metadata_only` has null content and cannot establish support/refutation. Never invent a publication date, source passage, URL, or publisher. Deduplicate sources by normalized URL within a pair; the same URL may legitimately have separate records for different pairs.
10. Every checked pair produces exactly two `ClaimVerdict` entries, one for each claim. `supported`/`refuted` requires at least one matching support/refutation reference to substantive evidence for that pair. A summary or source can be relevant without settling the exact claim; use `context` or abstain. Both claims may be refuted. No result is generated from the model's memory alone. `grounding_attributions` contains at most one native Google Search entry-point record per checked pair, with only the provider's actual returned search-entry-point HTML, never model-generated HTML. It is empty for fixtures and providers without this metadata. Keep this field through storage/export and incremental merging. It does not constitute factual evidence.
11. Verification's eligible universe is only analysis pairs with relation `direct_conflict` or `numeric_discrepancy`. Every eligible pair occurs exactly once in `results` or `unverified`; no overlap. Results and evidence IDs must resolve to the same pair and its claims. Unselected eligible pairs use `not_selected`; exhausted distinct-pair cap uses `budget_limit`. With no eligible pairs both arrays are empty. No sources can yield two insufficient-evidence verdicts and an explanatory issue, or `unverified/no_sources`; choose the former unless the provider failed to produce a usable assessment.
12. Snapshot references must match: analysis.transcript_id equals transcript.transcript_id; verification.analysis_id equals analysis.analysis_id. Analysis without a transcript and verification without analysis are invalid. `complete` means all requested stages finished, including legitimate abstentions; `partial` means usable upstream results with a failed/requested downstream stage or unverified pairs selected for checking; `failed` means required transcription/analysis cannot finish. `ready` represents saved transcript input awaiting analysis. `running` is transient. Intentional `not_selected` pairs alone do not make a job partial.

### Stable IDs, cache keys, and transcript corrections

Canonical JSON uses `json.dumps(payload, sort_keys=True, separators=(',', ':'), ensure_ascii=False, allow_nan=False).encode('utf-8')`; digest is SHA256 hex. Normalize audio timestamps to three decimal places before assigning IDs.

* `transcript_id = 'tr_' + digest({content_sha256, language, duration_s, speakers, segments})`, using exactly those key names and values from Transcript; excludes usage, source path, issues, and origin. Speaker name or text edits therefore change the ID.
* `analysis_id = 'an_' + digest({transcript_id, claims, pairs})`.
* `verification_id = 've_' + digest({analysis_id, evidence, results, unverified, grounding_attributions})`.
* IDs are full digests, not process-random hashes. Job IDs are random UUID strings. Hashes are content identities; **cache keys separately include** stage name, schema version, input artifact ID/hash, all relevant model IDs, normalization/extraction/comparison/retrieval/adjudication prompt versions, and output-affecting settings. Budget limits do not change an already-computed semantic result. Never cache provider failures as successful artifacts.

On a transcript correction, retain the original immutable artifact, save the reviewed transcript with a new ID and `origin='human_reviewed'`, and clear active analysis/verification pointers in one SQLite transaction. Reanalysis is an explicit button action. Do not show old verdicts against corrected quotes. Changing only a title does not change the transcript ID. Evidence is historical: display retrieval time and use a user-invoked refresh; do not silently re-search on every rerun. The final backend supports `refresh_verification` by invalidating only the selected pair cache before `verify_pairs`, without resetting the attempt/distinct-pair limits. A reservation identifies a pair by `analysis_id + ':' + pair_id`, since local pair IDs restart in each analysis. Storage treats this qualified token as its opaque pair_id reservation key; the backend converts allowed tokens back to local pair IDs before calling VerificationService. Rechecking the same analysis/pair consumes no additional distinct-pair slot, but a pair from a changed analysis does.

### Reference fixture: fully fictional, no internet required

Use sample ID `riverton_demo`, title `Riverton sample — fictional`, English, duration 65.0 seconds, speakers `speaker_01/Alex` and `speaker_02/Blair`. Source kind is `sample`, origin `fixture`, filename and audio path null. `content_sha256` is the SHA256 of canonical JSON of this sample's segment array. There is no sample audio unless a human records the script; show 'Transcript sample; audio unavailable'. Never fabricate a spoken recording.

| Segment | Speaker | Start–end seconds | Exact text |
|---|---|---|---|
| seg_0001 | speaker_01 | 0–8 | In 2024, Riverton's transport budget was 10 million US dollars. |
| seg_0002 | speaker_02 | 8–16 | In 2024, Riverton's transport budget was 12 million US dollars. |
| seg_0003 | speaker_01 | 16–24 | Riverton's unemployment rate was 5 percent in 2023. |
| seg_0004 | speaker_02 | 24–32 | Riverton's unemployment rate was 6 percent in 2024. |
| seg_0005 | speaker_01 | 32–40 | Riverton's 2025 library allocation was 2.5 million US dollars. |
| seg_0006 | speaker_02 | 40–48 | Riverton's 2025 library allocation was 2500000 US dollars. |
| seg_0007 | speaker_01 | 48–56 | I think this is the best transport policy. |
| seg_0008 | speaker_02 | 56–65 | In another report, it increased by five percent. |

Fixture claims are `cl_0001`–`cl_0006` from the first six segments plus `cl_0007` from seg_0008. Each uses its full original segment as source span. Exclude the opinion in seg_0007. For cl_0007 leave subject, metric, period, geography, population, measurement_basis null; checkability is needs_context. First six have geography Riverton; budgets/allocation use quantity unit currency, currency USD, basis annual allocation; unemployment uses percent and explicitly unspecified statistical definition so differing years are not comparable. Do not invent a population definition. Fixture pairs:

* `pair_0001`: cl_0001 vs cl_0002, numeric_discrepancy, difference `2000000`, unit currency, priority 1.
* `pair_0002`: cl_0003 vs cl_0004, different_scope because 2023 vs 2024, null difference, priority 3.
* `pair_0003`: cl_0005 vs cl_0006, compatible because both normalize to `2500000` USD in 2025, difference `0`, unit currency, priority 3.

The budget fixture uses a clearly synthetic evidence record at `https://example.com/althea-fixtures/riverton-budget-2024`, title `Fictional Riverton budget record`, publisher `Synthetic demo fixture`, date `2024-12-31`, and content `Riverton's adopted transport budget for calendar year 2024 was USD 11000000.` Its origin is retrieved_excerpt **only within the fictional fixture**, explicitly labelled as such throughout the UI. Both claims are refuted. This is not a live citation or a source to fetch. All fixture usage has provider fixture, model fixture-v1, request_count 0, token counts 0. Fixture functionality must work with networking unavailable and all Google credentials unset.

### Local package and command conventions

Each README gives PowerShell commands from its own folder:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -e ".[dev]"
python -m pytest
```

Implement module CLIs with `--help`, `--mode fixture|live`, `--input`, and `--output` where relevant. Outputs are UTF-8 JSON with actual newlines; successful empty analysis exits 0; invalid inputs/permanent failures exit nonzero with a short message. No import should trigger API calls, downloads, UI startup, or database creation. Unit tests use fake providers and temporary storage; live smoke tests are explicitly marked, excluded by default, and accurately reported as run/skipped. Never claim live success based on fixtures.

### Frozen executable contract

Create the following as `src/<your_package>/_contract.py` verbatim. The document's additional semantic rules are required even where the base model alone cannot enforce them. Every package must expose `validate_<stage>` helpers that enforce its applicable reference and semantic rules before returning a dictionary. Integration tests enforce the cross-stage rules.

```python
"""Althea wire contract v1.0.0. Private copies must remain byte-identical."""
from __future__ import annotations
from typing import Annotated, Any, Callable, Literal, Protocol
from pydantic import BaseModel, ConfigDict, Field

NonEmpty = Annotated[str, Field(min_length=1)]
NonNegative = Annotated[float, Field(ge=0, allow_inf_nan=False)]
DecimalText = Annotated[str, Field(pattern=r"^-?(0|[1-9][0-9]*)(\.[0-9]+)?$")]

class Wire(BaseModel):
    model_config = ConfigDict(extra="forbid", allow_inf_nan=False)

class Issue(Wire):
    stage: Literal["input", "transcription", "analysis", "verification", "storage", "integration"]
    code: NonEmpty
    message: NonEmpty
    retryable: bool
    related_ids: list[str]

class Usage(Wire):
    stage: Literal["transcription", "extraction", "comparison", "retrieval", "adjudication"]
    provider: Literal["vertex", "fixture"]
    model: NonEmpty
    request_count: Annotated[int, Field(ge=0)]
    input_tokens: Annotated[int, Field(ge=0)] | None
    output_tokens: Annotated[int, Field(ge=0)] | None

class InputRequest(Wire):
    kind: Literal["audio", "transcript", "sample"]
    local_path: str | None
    sample_id: str | None
    title: NonEmpty

class Source(Wire):
    kind: Literal["audio", "transcript", "sample"]
    title: NonEmpty
    original_filename: str | None
    content_sha256: Annotated[str, Field(pattern=r"^[a-f0-9]{64}$")]
    audio_relpath: str | None

class Speaker(Wire):
    speaker_id: NonEmpty
    display_name: NonEmpty

class Segment(Wire):
    segment_id: NonEmpty
    speaker_id: NonEmpty
    start_s: NonNegative
    end_s: NonNegative
    text: NonEmpty

class Transcript(Wire):
    schema_version: Literal["1.0.0"]
    transcript_id: NonEmpty
    origin: Literal["api", "imported", "human_reviewed", "fixture"]
    language: Literal["en"]
    duration_s: Annotated[float, Field(gt=0, le=300, allow_inf_nan=False)]
    source: Source
    speakers: list[Speaker]
    segments: list[Segment]
    issues: list[Issue]
    usage: list[Usage]

class SourceSpan(Wire):
    segment_id: NonEmpty
    start_char: Annotated[int, Field(ge=0)]
    end_char: Annotated[int, Field(gt=0)]
    quote: NonEmpty

class Quantity(Wire):
    raw_text: NonEmpty
    value: DecimalText | None
    lower: DecimalText | None
    upper: DecimalText | None
    operator: Literal["eq", "approx", "gt", "ge", "lt", "le", "range"]
    unit: NonEmpty
    currency: str | None
    basis: str | None

class Claim(Wire):
    claim_id: NonEmpty
    speaker_id: NonEmpty
    proposition: NonEmpty
    subject: str | None
    metric: str | None
    period: str | None
    geography: str | None
    population: str | None
    measurement_basis: str | None
    qualifiers: list[str]
    checkability: Literal["checkable", "needs_context"]
    missing_context: list[str]
    source_spans: list[SourceSpan]
    quantity: Quantity | None

class PairAssessment(Wire):
    pair_id: NonEmpty
    claim_a_id: NonEmpty
    claim_b_id: NonEmpty
    relation: Literal["direct_conflict", "numeric_discrepancy", "compatible", "different_scope", "insufficient_context"]
    explanation: NonEmpty
    numeric_difference: DecimalText | None
    numeric_difference_unit: str | None
    priority: Literal[1, 2, 3]
    method: Literal["deterministic", "llm", "hybrid", "fixture"]

class Analysis(Wire):
    schema_version: Literal["1.0.0"]
    analysis_id: NonEmpty
    transcript_id: NonEmpty
    claims: list[Claim]
    pairs: list[PairAssessment]
    issues: list[Issue]
    usage: list[Usage]

class Evidence(Wire):
    evidence_id: NonEmpty
    pair_id: NonEmpty
    claim_ids: list[str]
    url: NonEmpty
    title: NonEmpty
    publisher: str | None
    published_date: str | None
    retrieved_at: NonEmpty
    content: str | None
    content_origin: Literal["retrieved_excerpt", "grounded_summary", "metadata_only"]
    queries: list[str]

class EvidenceRef(Wire):
    evidence_id: NonEmpty
    stance: Literal["supports", "refutes", "context"]

class ClaimVerdict(Wire):
    claim_id: NonEmpty
    status: Literal["supported", "refuted", "insufficient_evidence", "needs_context"]
    rationale: NonEmpty
    evidence_refs: list[EvidenceRef]

class PairVerdict(Wire):
    pair_id: NonEmpty
    verdicts: list[ClaimVerdict]
    summary: NonEmpty

class Unverified(Wire):
    pair_id: NonEmpty
    reason: Literal["not_selected", "budget_limit", "provider_error", "no_sources", "missing_context"]

class GroundingAttribution(Wire):
    pair_id: NonEmpty
    search_entry_point_html: str | None

class Verification(Wire):
    schema_version: Literal["1.0.0"]
    verification_id: NonEmpty
    analysis_id: NonEmpty
    evidence: list[Evidence]
    results: list[PairVerdict]
    unverified: list[Unverified]
    grounding_attributions: list[GroundingAttribution]
    issues: list[Issue]
    usage: list[Usage]

class Snapshot(Wire):
    schema_version: Literal["1.0.0"]
    job_id: NonEmpty
    title: NonEmpty
    mode: Literal["fixture", "live"]
    status: Literal["ready", "running", "complete", "partial", "failed"]
    created_at: NonEmpty
    updated_at: NonEmpty
    transcript: Transcript | None
    analysis: Analysis | None
    verification: Verification | None
    issues: list[Issue]

class JobSummary(Wire):
    job_id: NonEmpty
    title: NonEmpty
    mode: Literal["fixture", "live"]
    status: Literal["ready", "running", "complete", "partial", "failed"]
    updated_at: NonEmpty

Progress = Callable[[str], None]

class DashboardBackend(Protocol):
    mode: Literal["fixture", "live"]
    def run(self, source: dict[str, Any], *, verify: bool,
            progress: Progress | None = None) -> dict[str, Any]: ...
    def load(self, job_id: str) -> dict[str, Any]: ...
    def list_jobs(self, limit: int = 20) -> list[dict[str, Any]]: ...
    def save_review(self, job_id: str, transcript: dict[str, Any]) -> dict[str, Any]: ...
    def reanalyze(self, job_id: str, *, verify: bool,
                  progress: Progress | None = None) -> dict[str, Any]: ...
    def verify_pairs(self, job_id: str, pair_ids: list[str], *,
                     progress: Progress | None = None) -> dict[str, Any]: ...
```

### Official references and model-availability gate

Documentation was checked on **16 October 2026**. Model availability, SDK field names, and billing remain account/region dependent. Before a live implementation, read the current pages, then make a minimal smoke call using existing authorized credentials. Do not invent an SDK field or silently change providers when a feature is unavailable. Return a clear provider error and preserve the offline mode; record the unresolved live limitation in HANDOFF.md.

* Google Cloud transcription: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/gemini/3-5-transcribe
* Google Cloud structured output: https://docs.cloud.google.com/vertex-ai/generative-ai/docs/multimodal/control-generated-output
* Google Cloud grounding: https://docs.cloud.google.com/vertex-ai/generative-ai/docs/grounding/grounding-with-google-search
* Google Cloud model pricing: https://cloud.google.com/gemini-enterprise-agent-platform/generative-ai/pricing
* SDK: https://github.com/googleapis/python-genai
* ADC setup: https://cloud.google.com/docs/authentication/provide-credentials-adc
* Gemini Developer API billing distinction: https://ai.google.dev/gemini-api/docs/billing

Google Cloud currently documents Gemini 3.5 Transcribe with diarization/timestamps but marks these features experimental. Do not combine readability-oriented Smart cleanup with the verbatim evidence requirement. Extract and validate provider word/segment annotations; do not assume the transcription endpoint supports a generic response JSON schema. Gemini 2.5 Flash grounded retrieval and structured adjudication are **separate calls**. Grounding citations must come from native metadata, not URLs improvised by the judge model.
