# n8n GTM Orchestration — FDA Signal → Scored Outreach Queue

A scheduled n8n workflow that turns live FDA enforcement filings into a ranked,
grounded outreach queue — with the failure handling a production GTM pipeline
actually needs.

**This workflow was executed headlessly and its real output is committed**, in
`output/sample-run.json`. It is not a screenshot of a canvas.

## The problem it solves

Signal-based outbound only works if the signal refreshes on its own. The moment
it depends on someone remembering to re-run a script, it goes stale, and reps
start calling accounts whose trigger expired months ago. Worse, when the refresh
*fails*, the queue doesn't empty — it silently serves yesterday's data and
nobody knows.

So the interesting part of this workflow isn't the happy path. It's what happens
when the API is down.

## The workflow

```
Daily 07:00 ─┐
             ├─→ Fetch openFDA ──→ Roll Up + Score ──→ Filter ≥60 ──→ Build Briefs
Manual Run ──┘        │
                      └── (on error, after 3 retries) ──→ Alert: Fetch Failed
```

Two entry points by design: the **Schedule Trigger** is the production entry
(daily 07:00); the **Manual Trigger** exists so the workflow can be executed from
CLI and CI, which is how the committed run below was produced.

## Production markers

These are the specific things GTM Engineer postings ask for, and where each lives:

| Marker | Where |
|---|---|
| **Retry with backoff** | `retryOnFail: true`, `maxTries: 3`, `waitBetweenTries: 5000` on the HTTP node |
| **Error branch** | `onError: continueErrorOutput` routes a failed fetch to a dedicated alert node instead of killing the run |
| **Rate limiting** | `batching: { batchSize: 1, batchInterval: 1000 }` — deliberate throttling against a public API |
| **Timeout** | 30s ceiling on the fetch |
| **Fail-closed guard** | the scoring node *throws* on an empty response rather than emitting an empty queue — a silent empty queue is worse than a loud failure |
| **Observability** | `alwaysOutputData` so a zero-result fetch is still visible in the run log |

The alert node emits a structured payload — severity, stage, message, and an
explicit `action: "Queue is stale. Do not send from it until this clears."`
Wiring that to Slack or PagerDuty is scheduler config; it is **not** claimed here
as built.

## Verified run

Executed via `n8n execute` on n8n **2.35.7**, against the live openFDA API:

| Node | Status | Items out | Time |
|---|---|---|---|
| Manual Run | success | 1 | 1 ms |
| Fetch openFDA Enforcement | success | 1 | 1,886 ms |
| Roll Up + Score Firms | success | 34 | 42 ms |
| Filter ≥ 60 | success | 11 | 42 ms |
| Build Grounded Briefs | success | 10 | 13 ms |

**Funnel:** 100 raw enforcement records → 34 firms after roll-up → 11 cleared the
score threshold → 10 briefs.

These numbers move between runs because the data is live — that is the point of a
scheduled signal pipeline, and the committed `sample-run.json` is one dated
snapshot, not a fixed fixture.

## Run it yourself

```bash
npm install n8n
export N8N_USER_FOLDER="$PWD/.n8n-home"
npx n8n import:workflow --input=workflows/fda-signal-to-outreach-queue.json
npx n8n execute --id=fdaSignalQueue01
```

No API key and no paid data — openFDA is a public federal endpoint.

## Scoring

Mirrors the disclosed rubric in
[atlas-signal-engine](https://github.com/Aditya-chouhan/atlas-signal-engine)
(max 100): severity by worst classification, recency of the most recent event,
systemic count of distinct `event_id`s, a quality-system reason flag, and an
ongoing-status bonus. It is a **buying-trigger score, not a quality verdict** — it
measures how acute a firm's compliance pain is right now, not whether the firm is
good or bad.

## Honesty guardrails

- Every fact in a generated brief is copied verbatim from the FDA filing. Nothing
  is inferred or embellished.
- **No contact is ever invented.** Outreach targets a *role*
  (`Chief Quality Officer / VP Quality`), never a named person or guessed email.
- **No email is sent.** This workflow produces a queue; delivery is out of scope.
- The committed run is real output from a real execution against the live API.
- `node_modules/` is gitignored — n8n is a 2.5 GB install and is not vendored here.

## Files

```
workflows/fda-signal-to-outreach-queue.json   the workflow (importable)
output/sample-run.json                        committed real execution output
```
