# Althea AI — three-person build handoff pack

Prepared 16 October 2026 for a five-minute, English-only group-project demo. Hosted APIs are allowed; local NLP models are unnecessary. The intended live provider is Google Cloud/Vertex-compatible Gemini, using eligible existing credits. Credit eligibility and live account access still need verification.

## Give each person one complete document

| Person | Document | Working folder | Owns |
|---|---|---|---|
| 1 | [01_PERSON_1_INGESTION_AND_STORAGE.md](01_PERSON_1_INGESTION_AND_STORAGE.md) | team_1_ingestion/ | Audio, transcription, transcript validation/review, SQLite repository |
| 2 | [02_PERSON_2_CLAIMS_AND_DISAGREEMENTS.md](02_PERSON_2_CLAIMS_AND_DISAGREEMENTS.md) | team_2_analysis/ | Claim ledger, scope preservation, normalization, matching, disagreement analysis |
| 3 | [03_PERSON_3_EVIDENCE_AND_DASHBOARD.md](03_PERSON_3_EVIDENCE_AND_DASHBOARD.md) | team_3_evidence_ui/ | Web evidence, per-claim verdicts, citations, standalone Streamlit UI |
| Final phase | [04_FINAL_INTEGRATION.md](04_FINAL_INTEGRATION.md) | integration/ plus root setup | Connect completed modules, persistent budgets/cache, integration tests, final demo |

Each of the four documents is self-contained: it embeds the **same frozen contract, fixture specification, limits, ownership rules, and official references**. Give an agent the full file as its task. Give it access to only its working folder for edits during parallel work. Do not give all three agents the original broad proposal as an instruction to build the entire app.

## Suggested kickoff message

> Read the attached handoff document completely. Implement the assigned component inside its named folder. Treat the frozen contract and acceptance criteria as requirements. Build against the supplied fixture/fake providers so you do not wait for the other group members. Deliver runnable code, tests, README.md, and HANDOFF.md. Do not modify the other people's folders. Report live provider checks honestly; fixture success does not establish live success.

## Parallel sequence

1. Agree that the v1 contract is frozen. Distribute one role document per person; no implementation prework by another team is required.
2. Each person builds their component and offline tests in their own folder. Give all three the optional reference assets as a convenience; their brief contains the full specification if assets are unavailable.
3. Each delivers HANDOFF.md, public APIs, tested dependency versions, and fixtures. Request interface changes through the group; never rename a field privately.
4. Bring the three completed folders into one repository. Run the integration agent with document 04 and the handoffs.
5. Validate the full pipeline with a real authorized clip and current account access, then rehearse the saved sample and live demo.

For equal day-to-day progress, Person 3 can build the dashboard against FixtureBackend while implementing evidence retrieval; Person 1 can implement repository behavior while waiting for live transcription access; Person 2 can complete normalization/comparison tests without cloud credentials. Person 3 has the broadest presentation work, so the optional graph/YouTube work is deliberately excluded. If someone finishes early, they can help review tests or record a sample through the group, without taking over another folder during parallel edits.

## Reference assets

* `reference/contract_v1.py`: exact frozen Pydantic types embedded in every brief. This is an interface reference, **not an implemented application**.
* `reference/contract_v1.schema.json`: generated JSON Schema for Snapshot.
* `reference/reference_snapshot.json`: a validated fictional end-to-end example. No API calls were made to produce it; evidence is synthetic.

Use references as copied fixtures, not as a new shared runtime dependency. Components exchange dictionaries, so independently mirrored private validators do not cause cross-package Pydantic type incompatibilities. Integration checks contract hashes and cross-stage semantics.

## Scope choices

* Python + Streamlit + SQLite; no servers or local ML weights.
* Audio upload and transcript import required; YouTube and graph postponed.
* Two speakers recommended, 300-second maximum, English.
* Up to 30 claims, 60 candidate pairs, eight highlighted conflicts, four checked pairs per job.
* Up to 20 provider attempts per job; this bounds calls, not guaranteed spending in rupees.
* Google Search grounding and structured adjudication use separate calls. No automatic switch from Vertex to another billing route.
* Correct source quotes, comparable scope, abstention, both-wrong results, and transparent synthetic/live labelling are more important than impressive-looking confidence scores.

The four briefs prescribe future implementation work. This handoff pack does not claim that the app, provider access, live accuracy, or the final demo has already been built or tested.
