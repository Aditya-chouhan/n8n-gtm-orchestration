# n8n GTM Orchestration — FDA Signal → Scored Outreach Queue

A scheduled n8n workflow that turns live FDA enforcement filings into a ranked,
grounded outreach queue — with the failure handling a production GTM pipeline
actually needs.

**Two real execution records are committed here, both native n8n output:**

| File | What it is |
|---|---|
| `output/native-execution-2026-08-23.json` | n8n's own `resultData.runData` dump from a live run against openFDA — 456 KB, unedited, including the raw API payload |
| `output/error-workflow-fired-2026-08-23.json` | Proof the error workflow actually fires: a production failure and the handler execution it triggered |
| `output/sample-run.json` | An earlier human-readable **summary I wrote** from a run. Kept for readability, but it is my schema, not n8n's |

The distinction matters and used to be the weak point of this repo: `sample-run.json`
is a re-presentation, and a re-presentation can quietly drift from what actually
happened. `native-execution-2026-08-23.json` cannot — it is what the engine
emitted, pointer-interning and all.

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

  … and if the workflow itself crashes anywhere above:

      ✗ ──→ [GTM Error Handler] ──→ classify ──→ page  (code defect / auth / 4xx)
                                             └─→ log   (timeout / 429 / 5xx)
```

Two entry points: the **Schedule Trigger** is the production entry (daily 07:00);
the **Manual Trigger** exists so the workflow can be executed from CLI and CI,
which is how the committed run below was produced.

The manual trigger is there because the first CLI run failed — `n8n execute`
returned *"Missing node to start execution"*, since a Schedule Trigger alone
gives the CLI no entrypoint. Adding the manual trigger fixed it. Recording that
because "it worked first time" would be the less useful claim.

## Production markers

These are the specific things GTM Engineer postings ask for, and where each lives:

| Marker | Where |
|---|---|
| **Retry with backoff** | `retryOnFail: true`, `maxTries: 3`, `waitBetweenTries: 5000` on the HTTP node |
| **Error branch** | `onError: continueErrorOutput` routes a failed fetch to a dedicated alert node instead of killing the run |
| **Dedicated error workflow** | `settings.errorWorkflow` points at `gtm-error-handler.json`, which catches whole-workflow crashes the branch above cannot see — **verified firing**, see below |
| **Rate limiting** | `batching: { batchSize: 1, batchInterval: 1000 }` — deliberate throttling against a public API |
| **Timeout** | 30s ceiling on the fetch |
| **Fail-closed guard** | the scoring node *throws* on an empty response rather than emitting an empty queue — a silent empty queue is worse than a loud failure |
| **Observability** | `alwaysOutputData` so a zero-result fetch is still visible in the run log |

The alert node emits a structured payload — severity, stage, message, and an
explicit `action: "Queue is stale. Do not send from it until this clears."`
Wiring that to Slack or PagerDuty is scheduler config; it is **not** claimed here
as built.

## The error path, proven rather than asserted

The error *branch* above only catches the one node it is wired to. It cannot
catch a Code node throwing, a filter blowing up, or anything else nobody thought
to guard — and those are the failures that actually take a pipeline down.

So there is a second workflow, `workflows/gtm-error-handler.json`, registered as
this workflow's `errorWorkflow`. It receives the whole failed execution,
classifies it, and decides whether a human needs waking:

```
Workflow crashes ─→ On Workflow Error ─→ Classify Failure ─→ Page Or Log?
                                                              ├─ page     → Page On-Call
                                                              └─ log_only → Log And Suppress
```

Transient failures (timeouts, `ECONNRESET`, 429, 5xx) log and stay quiet — the
next scheduled run is expected to succeed and paging someone at 03:00 for a
blip is how alerting gets muted. Code defects, auth failures and 4xx contract
changes page, because every run until someone looks will fail the same way.
**Anything unrecognised pages too** — an unclassified failure assumed transient
costs a full day of pipeline.

**It was verified firing, not assumed.** `output/error-workflow-fired-2026-08-23.json`
holds both halves: a production execution that failed (`mode: webhook`), and the
handler execution n8n triggered 80 ms later (`mode: error`), which classified it
as `code_defect` → `page`. `workflows/_error-path-probe.json` is the fixture that
produces that failure on demand.

### Three things that only turned up by actually running it

**1. Error workflows do not fire for CLI executions.** Running the failure probe
via `n8n execute` produced a failed execution and *nothing else*. Error workflows
trigger on production executions — webhook, schedule — not `mode: cli`. Anyone
testing their error handling from the CLI will conclude it works when it has
never once run.

**2. The error workflow must itself be published, or n8n silently declines.**
With the handler imported but inactive, the failure produced no handler run and
no error. The only trace was a single server log line:

```
Calling Error Workflow for "...". Workflow "gtmErrorHandler01" is not active
and cannot be executed
```

Configuration looks correct, the setting is present, and the alerting does
nothing. This is the worst class of bug in an alerting path.

**3. n8n rewrites exceptions thrown inside Code nodes.** Throwing
`new TypeError('probe: simulated code defect')` arrives at the handler as
`name: "Error"` — constructor name gone — with the message rewritten to
`"simulated code defect [line 3]"` and the original prefix moved into
`error.description`. The first version of the classifier keyed on `error.name`
and therefore never matched a code defect; it fell through to `unclassified`.
It still paged, because unrecognised defaults to paging, but the specific rule
was wrong. It now keys on the `[line N]` suffix, which n8n only appends to code
it evaluated itself. The fix came from reading a real failure, not from
reasoning about one.

## Verified run

Executed on n8n **2.35.7** against the live openFDA API, 2026-08-23. Every figure
below reads directly out of `output/native-execution-2026-08-23.json`:

| Node | Status | Items out | Time |
|---|---|---|---|
| Manual Run | success | 1 | 1 ms |
| Fetch openFDA Enforcement | success | 1 | 2,331 ms |
| Roll Up + Score Firms | success | 34 | 61 ms |
| Filter ≥ 60 | success | 11 | 46 ms |
| Build Grounded Briefs | success | 10 | 16 ms |

Total wall clock 2.46 s (`startedAt` 08:05:22.452Z → `stoppedAt` 08:05:24.915Z).
An independent run two days earlier produced the same 100 → 34 → 11 → 10 funnel
with different timings, which is the expected behaviour for a live feed.

**Funnel:** 100 raw enforcement records → 34 firms after roll-up → 11 cleared the
score threshold → 10 briefs.

**The 11 → 10 step is not attrition — it is a hardcoded cap.** `Build Grounded
Briefs` does `rows.slice(0, 10)`, so the queue never emits more than 10 briefs
per run no matter how many firms clear the threshold. That is a deliberate
volume guard on a daily job, but it is a code decision and not a property of the
data, so it is stated here rather than left to look like a drop-off.

The other numbers do move between runs, because the data is live — that is the
point of a scheduled signal pipeline, and the committed `sample-run.json` is one
dated snapshot, not a fixed fixture.

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
workflows/fda-signal-to-outreach-queue.json   the pipeline (importable)
workflows/gtm-error-handler.json              error workflow: classify + route
workflows/_error-path-probe.json              test fixture that fails on purpose
output/native-execution-2026-08-23.json       n8n's own runData dump, unedited
output/error-workflow-fired-2026-08-23.json   proof the error path fires
output/sample-run.json                        earlier human-written summary
```

## Verify the error path yourself

Error workflows need a **server**, not the CLI, and both workflows must be
published:

```bash
export N8N_USER_FOLDER="$PWD/.n8n-home"
npx n8n import:workflow --input=workflows/gtm-error-handler.json
npx n8n import:workflow --input=workflows/_error-path-probe.json
npx n8n publish:workflow --id=gtmErrorHandler01
npx n8n publish:workflow --id=gtmErrorPathProbe
npx n8n start                                    # separate terminal
curl http://127.0.0.1:5678/webhook/gtm-failure-probe
```

Then check the executions: the probe fails, and a second execution appears for
`gtmErrorHandler01` with `mode: error`. If the handler execution is missing,
re-read gotcha 2 above — it is almost always the publish step.
