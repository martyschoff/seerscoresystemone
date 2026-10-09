# SeerScore SystemOne Lab — design spec

## Purpose

Create a public, reusable test lab for evaluating SystemOne-style typed decisions against SeerScore claim data before any production integration. Start with claim-type and measurability gating; keep all real SeerScore data local. The framework should make experiments reproducible, make model abstention/failure visible, and provide a practical UI for human labeling and result review.

## Confirmed context

- The current SeerScore data contains 779 Peter Zeihan prediction records: 401 are marked `quantitative` and 378 `subjective` by the existing heuristic. This is a noisy baseline, not ground truth.
- Nimble is installed on the user's Ollama host, is currently loaded, and `/v1/systemone` has already returned a typed verdict in a smoke test.
- The public project will be named `seerscoresystemone`.
- Implementation should use a Sonnet model; use the Sonnet variant available in Cursor Agent at implementation time.

## Chosen architecture

A **local-first Python application** with one shared core and two interfaces:

1. **Core library:** versioned task schemas, SystemOne adapter, deterministic decision/policy layer, run orchestration, import/export boundaries, evaluation metrics.
2. **CLI:** reproducible scripted smoke tests, batch runs, imports, labeling/evaluation export.
3. **Local web UI:** keyboard-first labeling queue, experiment setup/progress, disagreements, metrics, and run comparisons.
4. **Storage:** SQLite for runs, labels, tasks, responses, and metrics; private data directory outside the Git checkout.

Start with FastAPI + server-rendered templates (HTMX optional) rather than a separate SPA. Keep the model adapter replaceable and provider-specific behavior isolated. The app binds to loopback by default and never writes to SeerScore.

## Data and public-repo boundary

- Public Git contains application code, docs, schemas, tests, and synthetic examples only.
- Real claims, transcript excerpts, IDs, source URLs, labels, model outputs, logs, and endpoints stay in a configurable local data directory excluded from Git.
- Import a local snapshot read-only, record its hash, and retain the baseline label as a heuristic annotation. Do not silently treat it as truth.
- Send claim text to the configured SystemOne endpoint only after an explicit real-data opt-in (`S1LAB_ALLOW_REAL_DATA=1` or an equivalent UI confirmation). Show the destination and fields before each real-data run; default to synthetic fixtures. Record an audit entry containing dataset hash, destination identifier, field names, model/task, timestamp, and operator confirmation (never store credentials). Do not send transcript context by default.
- No production write-back, no automatic exclusion/deletion, and no truth-verdict evaluation.

## Versioned test applications

**T0 — connectivity/contract smoke test.** Verify endpoint reachability and the actual request/response contract against the installed Ollama SystemOne endpoint; validate parsed outputs and report model/version/latency. Do not guess undocumented response fields.

**T1 — synthetic single-item gate.** Exercise a task schema on synthetic claims and explicit abstentions. Persist raw response privately, validated result, schema version, model and timing.

**T2 — resilience/fake endpoint.** Test timeout, malformed output, rate/server errors, retry limits, cancellation, cache behavior and deterministic fallbacks without contacting a real model.

**T3 — private SeerScore import and baseline.** Load a user-selected local snapshot read-only; reproduce the current heuristic and report mismatches. Begin with the 779 Zeihan records; do not include them or real examples in public tests.

**T4 — human labeling UI.** Blind labels from existing model/heuristic outputs; support unsure/abstain, skip, and adjudication. Start with a fixed-seed stratified sample and retain a locked evaluation split.

**T5 — shadow evaluation.** Run a chosen task (first candidate: measurability; claim type can be a sibling task) on a labeled sample. Report per-class precision/recall/F1, confusion matrix, abstention/coverage, latency, invalid-output rate, and confidence intervals. Compare with the heuristic diagnostically, not as gold truth.

Later increments may add calibration, model comparisons, and proposal-only shadow reports. No integration back into production SeerScore is included without a separate decision.

## UI concept

- **Dashboard:** endpoint status; dataset/run identifiers (private UI only); progress through T0–T5; latest metrics and abstention rate.
- **Dataset/import:** select local snapshot, view hash/counts, baseline mismatch report.
- **Label queue:** one claim at a time, model/baseline answers hidden until submission, keyboard shortcuts, unsure/flag controls.
- **Run setup/monitor:** task/schema/model/config, sample selection, progress, retry/errors, cancel/pause.
- **Compare:** confusion matrix, coverage/accuracy, disagreement list, run-to-run comparisons.
- **Settings:** data directory and endpoint configuration shown with secrets redacted. Clearly disclose outbound fields before a real-data run.

## Reliability and policy

- A typed result includes task/schema version, label, confidence if actually provided, abstention/status, and optional bounded structured fields. Keep the SystemOne transport schema separate from the app's internal schema.
- Validate responses strictly. Treat timeouts, malformed output, and service errors as explicit failures/abstentions, never as an implicit label.
- Retry only documented transient failures with bounded backoff; cap concurrency; expose service health and failures.
- Deterministic application code decides routing. For the test lab, every result is advisory; uncertainty routes to human review. No destructive action based on model confidence.
- Store immutable run manifests (dataset hash, task/schema/prompt versions, model identifier, endpoint config hash, seed, timestamp). Cache only with complete input/config keys; disclose when a cached response is used.

## Suggested module boundaries

- `s1lab/config` — local config and redacted display.
- `s1lab/schemas` — internal task/result/run/label models.
- `s1lab/systemone` — transport adapter and contract validation.
- `s1lab/tasks` — claim type and measurability tasks.
- `s1lab/importers` — local snapshot and generic CSV/JSONL import.
- `s1lab/store` — SQLite schema/migrations.
- `s1lab/experiments` — runner, manifests, cache, fake endpoint.
- `s1lab/labeling` — queue, blind review, adjudication.
- `s1lab/evaluation` — metrics, sampling, confidence intervals.
- `s1lab/cli` and `s1lab/web` — interfaces over shared services.

## Milestones and acceptance criteria

1. Public repo and privacy guardrails; clean clone runs offline tests with synthetic-only fixtures.
2. T0/T1 run against the actual endpoint; contract verified from live response; no API assumptions baked in.
3. T2 resilience suite passes against a fake endpoint.
4. T3 imports the local snapshot without modifying it and reports count/hash/baseline mismatches.
5. T4 blind labeling works by keyboard and stores versioned human labels.
6. The app makes no real-data request without explicit opt-in; synthetic tests prove the guard, destination/field preview, and audit record.
7. T5 generates reproducible metrics with abstentions and invalid responses counted explicitly.

Before publishing, scan all staged files for real claims, IDs, URLs, endpoint addresses, credentials, and private configuration. Verify public repo contents by reading them back from GitHub.

## Assumptions to confirm

- Option B (local Python core + CLI + local web UI + SQLite) is the desired balance for the requested reusable framework.
- The first task is measurability gating, with claim-type gating next; both can use the same experiment framework.
- Real data remains on the user's machines, and only aggregate results approved by the user may be published.
- The exact SystemOne contract will be verified in T0 against the installed Nimble endpoint before the client is implemented.

## Implementation handoff

After this design is approved, create `martyschoff/seerscoresystemone` as public, publish this spec, write a detailed implementation plan, then implement one test application at a time with the available Cursor Agent Sonnet model and TDD. No implementation begins before the plan is approved.
