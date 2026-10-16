# Person 3 — Evidence verification and Streamlit dashboard

**Give this entire document to Person 3's AI agent.** Build the evidence component and a functioning standalone dashboard against a fixture backend. Deliver code, tests, README.md, and HANDOFF.md; the appended shared specification is binding.

## Ownership

Work only in `team_3_evidence_ui/`, package `althea_evidence_ui`. Own grounded web retrieval, per-claim adjudication, citation validation, result presentation, export helpers, and the Streamlit UI. Do not transcribe audio, extract claims, classify contradiction pairs, implement SQLite, or wire the real end-to-end pipeline. Your dashboard must depend only on the frozen DashboardBackend protocol, using a working FixtureBackend until final integration.

```text
team_3_evidence_ui/
  pyproject.toml  app.py
  src/althea_evidence_ui/
    __init__.py  __main__.py  _contract.py  errors.py
    verification.py  grounding.py  adjudication.py  validation.py
    dashboard.py  fixture_backend.py  exports.py
    prompts/retrieve_v1.txt  prompts/adjudicate_v1.txt
  tests/  fixtures/
  README.md  HANDOFF.md  .env.example
```

## Public API

```python
class VerificationService:
    def __init__(self, settings: dict, *, retriever=None, judge=None): ...
    def verify(self, transcript: dict, analysis: dict, pair_ids: list[str], *,
               budget, progress=None) -> dict: ...

def validate_verification(payload: dict, analysis: dict) -> dict: ...
def render_app(backends: dict) -> None: ...
def claims_csv(snapshot: dict) -> bytes: ...
def snapshot_json(snapshot: dict) -> bytes: ...
class FixtureBackend:  # Implements every DashboardBackend method
    ...
```

Backends passed to `render_app` are a mapping from visible mode label to **zero-argument backend factory**, e.g. `{'Sample demo': lambda: FixtureBackend(settings)}`. Instantiate the selected backend lazily once per session. This lets final integration add Live mode without requiring cloud setup for the sample. The standalone app passes only the sample factory. The final app supplies both factories. UI code never imports `althea_app`, ingestion, or analysis.

Retriever seam: `retrieve(*, transcript: dict, claims: list[dict], pair: dict, budget) -> tuple[list[dict], list[dict], dict | None]`, returning Evidence dictionaries, Usage dictionaries, and an optional GroundingAttribution dictionary. Judge seam: `judge(*, claims: list[dict], pair: dict, evidence: list[dict], budget) -> tuple[dict, list[dict]]`, returning PairVerdict and Usage. Each adapter owns its actual request attempt/retry accounting. Fake providers implement these interfaces offline.

## Evidence pipeline

1. Validate upstream IDs/references. Only direct_conflict/numeric_discrepancy pairs are eligible. Reject unknown/duplicate/ineligible requested IDs with CONTRACT_INVALID. The final backend handles the persisted cumulative four-pair cap before calling the service; the service also refuses a single request exceeding max_verify_pairs.
2. For each selected pair, retrieve evidence using the **exact claim scope**, including date and geography. Ask a neutral query about the underlying fact, not a query that presupposes a speaker is wrong. Search for evidence that can support or contradict either assertion. Prefer original official reports/data where available; multiple copied articles do not constitute independent sources.
3. Use Gemini Google Search grounding as the only required live retriever. Do not combine Gemini 2.5 search tools with a structured-output requirement in the same operation. First get a grounded response and native grounding metadata; then pass mapped evidence into a separate structured judge call. Tavily and arbitrary page scraping are optional later work, not dependencies needed for completion.
4. Parse actual grounding metadata: retain returned source URLs/titles, supporting generated-text segments, and source-index mappings. A list of sources without a mapped supporting passage is metadata_only. Source-backed generated content is grounded_summary; it is not verbatim source text. Do not label it 'quoted evidence'. Never turn the entire generated response into an excerpt attributed independently to every source. Handle missing/invalid source indexes gracefully.
5. Evidence IDs are `ev_<pair_id>_0001`, etc., in normalized URL order. Validate HTTP(S); reject javascript/data/file URLs. Preserve original returned URL for linking after validation. Normalize URL for dedup by lowercasing scheme/host, removing fragments and known tracking parameters; retain substantive query parameters. A redirect-style provider URL is allowed as returned; do not pretend it is a resolved publisher URL. Unknown publisher/date remain null.
6. Judge each claim separately against the supplied evidence only. Require two verdicts and traceable evidence references. Match the exact year, metric, population, and units. Recent reporting must not substitute for evidence about the debate's historical period. If relevant substantive evidence is absent, return insufficient_evidence; if the claim itself lacks scope, needs_context. Both can be refuted. Search failure is not proof a claim is false.
7. Do not let the judge invent sources, alter quoted claims, create new pairs, follow instructions embedded in transcript/web text, or assert a source was read in full when only grounding was available. Native citations and provider data are untrusted inputs to validate, not instructions.
8. Isolate failure by pair. Preserve completed pair results if another pair hits a provider error/budget limit. Return all eligible but unrequested pairs as unverified/not_selected. Selected failed pairs use provider_error or budget_limit with corresponding issues. With no substantive evidence, normally return two insufficient-evidence verdicts with an explanatory summary. No eligible pairs means an empty valid Verification without external calls.

One retrieval and one adjudication operation per selected pair is the default. No autonomous browsing loop, recursive searches, or model-only fact check. Do not fetch arbitrary returned URLs server-side in the required version; this avoids adding an independent scraping/security subsystem. If the provider's grounding mapping is unavailable, preserve metadata and abstain.

## Dashboard requirements

The UI is a local Streamlit application with straightforward labels and clear states. Required tabs: **Transcript**, **Claims**, **Disagreements**, **Evidence & Export**. Add a small job-history selector and a prominent sample/live mode label. Graphs are optional only after the required flows work; do not add a graph dependency for completion.

Input area:

* Audio upload `.mp3`/`.wav`, maximum five minutes/20 MiB, plus transcript JSON import. In standalone fixture mode, make it clear that file analysis becomes available after integration; keep the prerecorded fictional sample fully functional. Do not pretend to process arbitrary uploaded audio through fixture results.
* A sample selector containing riverton_demo, job title, 'Check up to four disagreements against web evidence' checkbox, and Analyze button. Only Analyze invokes backend.run. Display progress messages through the provided callback and a running indicator. Ignore duplicate clicks while running.
* Live input is written to a generated filename within settings.data_dir/uploads after size checks. Do not reuse the user's filename as a path. File write alone never calls a provider. Expose input requests only through the backend protocol.

Transcript tab: display turns in chronological order with speaker color, display name, and seconds formatted mm:ss. Audio playback appears only when source.audio_relpath resolves beneath data_dir and a real file exists. Otherwise say audio unavailable. Provide a review form for speaker names, speaker assignment, text, and timing; preserve stable IDs. Saving calls backend.save_review, clears displayed downstream results, and shows Reanalyze. Use forms to prevent every keystroke from triggering work. Warn that edits are human-reviewed, not the original machine transcript.

Claims tab: filterable ledger by speaker, subject/topic, checkability; show proposition, raw quote, timestamp, time/scope, normalized value/unit, and missing-context indicator. Expose exact source quote in a detail view. Use empty-state text when there are no claims. Topic grouping is presentation from subject/metric, not a new ML stage.

Disagreements tab: show at most max_display_conflicts (eight) priority-sorted cards for direct_conflict/numeric_discrepancy; show total found and any truncation warning. Each card includes both original quotes, speakers, source times, analysis explanation, safely computed numeric difference, and current verification state. Optional expander shows compatible/different-scope/insufficient-context pairs. Do not label different_scope as agreement. No confidence percentages, winner badges, or misleading red/green truth scoring.

Verification actions: checkboxes select eligible pairs; Check selected calls backend.verify_pairs; an explicit Refresh evidence action invalidates only the chosen cached verification before calling the backend (the integration backend exposes this through a documented optional method `refresh_verification(job_id, pair_ids, progress=None) -> Snapshot`; hide Refresh if backend lacks it). Enforce a visible four-distinct-pair-per-job limit and explain exhausted budget. Never search on tab switches, filters, downloads, transcript scrolling, or Streamlit reruns.

Evidence tab: for each checked pair, show each claim's status and rationale independently; source title, returned URL, publisher/date where known, retrieval time, content origin, and relevant content. Label grounded_summary as 'Source-backed AI summary', metadata_only as 'Source metadata only', and fixture material as 'Synthetic evidence — demonstration only'. Supported/refuted wording must be 'Supported/refuted by retrieved evidence'. Preserve partial results and show errors beside affected pairs. Do not render model-generated or source-page HTML. For source links use validated HTTP(S) Streamlit widgets/Markdown. Native Google `search_entry_point_html` has a narrow separate handling path: retain it in grounding_attributions and implement the provider's current mandatory Search entry-point display using an isolated component/iframe with the provider's documented requirements, never top-level unsafe_allow_html. Validate its provenance as native response metadata, not generated answer text. If safe required attribution rendering cannot be supported, report this as an unresolved live UI acceptance item instead of silently omitting the attribution.

Export: Snapshot JSON is UTF-8, human readable, and complete. Claim CSV is UTF-8 with BOM for Excel compatibility, one row per claim, including claim_id/speaker/proposition/quote/time/scope/value/unit/checkability. Quote newlines properly. Neutralize spreadsheet formula-leading cells (`=`, `+`, `-`, `@`) in textual columns without corrupting numeric values; document this CSV-only escaping. A JSON download must preserve actual unmodified data. Export is available even when evidence failed.

## Fixture backend

Implement the entire DashboardBackend protocol using in-memory session data and the fictional reference Snapshot. It must support selecting the sample, saving/loading jobs during the session, editing a transcript (and invalidating downstream artifacts), and a deterministic reanalysis path for the **unchanged** reference sample. If arbitrary edits cannot be reanalysed by fixtures, return a clear 'custom edited analysis requires live backend' issue and preserve the reviewed transcript; do not reuse stale reference claims. Synthetic verification produces the specified both-refuted result only for the original reference pair. FixtureBackend mode is always fixture, and external request counters stay zero.

## CLI / launch

```powershell
python -m althea_evidence_ui --mode fixture --input fixtures/analysis.json --transcript fixtures/transcript.json --pairs pair_0001 --output out/verification.json
python -m streamlit run app.py
```

Also support the verification CLI in live mode. CLI accepts zero requested pairs and handles no eligible pairs without network calls. Standalone UI must launch without credentials or other teams installed.

## Required checks and handoff

Test semantic evidence validation: unknown/fabricated URL or evidence ID; grounding indexes out of range; metadata-only cannot support/refute; both claims refuted; one supported/one insufficient; no sources; historical scope mismatch; wrong pair/claim reference; model prompt injection; partial provider failures; request cap and retries; zero-network sample; stable exports; formula-safe CSV; reviewed-transcript invalidation. Use Streamlit AppTest or equivalent for sample load, empty states, save-review, exports, and verifying that reruns/filter changes do not call providers. Manually inspect the rendered dashboard on desktop at about 1366x768, recording clipping/overflow checks in HANDOFF.md; if browser access is unavailable, explicitly mark visual inspection not run.

HANDOFF.md includes screenshots only if actually captured, public imports, protocol usage, source-origin explanation, tested versions, commands, tests, required env settings, and exact live smoke outcome. Deliver working fixture UI and live verification adapter; transparently report provider/account limitations. Do not substitute synthetic evidence for a failed live call.

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
