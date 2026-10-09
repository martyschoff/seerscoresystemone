# SeerScore SystemOne Lab Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a public, local-first SystemOne evaluation lab with a reusable Python core, CLI, local labeling UI, private data handling, and reproducible gating metrics.

**Architecture:** One package owns configuration, typed internal schemas, SystemOne transport, import, SQLite storage, experiment orchestration, labeling, and evaluation. CLI and loopback-only web UI call the same services. Real SeerScore data stays outside Git; the application never writes to SeerScore.

**Tech Stack:** Python >=3.12, `httpx`, FastAPI, Jinja2, SQLite, pytest, and a standard-library CLI entry point. Pin exact dependency versions in `pyproject.toml` when scaffolding.

**Implementation Model:** Cursor Agent, Claude Sonnet 5.5 Medium (the available Sonnet variant verified for this account).

**Spec:** `docs/superpowers/specs/2026-10-08-seerscoresystemone-design.md`

## Global Constraints

- Public Git contains application code, docs, schemas, tests, and synthetic examples only.
- Real claims, transcript excerpts, IDs, source URLs, labels, model outputs, logs, and endpoints stay in a configurable local data directory excluded from Git.
- Import a local snapshot read-only, record its hash, and retain the baseline label as a heuristic annotation.
- Send claim text only after explicit real-data opt-in; default to synthetic fixtures; do not send transcript context by default.
- No production write-back, automatic exclusion/deletion, or truth-verdict evaluation.
- The app binds to loopback by default; only the SystemOne adapter performs network calls.
- Treat timeout, malformed response, and service error as explicit failures/abstentions, never as an implicit label.
- Routing is deterministic application code; every model result is advisory and uncertainty routes to human review.

## Review Focus

1. Empty or malformed SystemOne responses, changed response fields, and unexpected HTTP statuses must become visible typed failures, never guessed labels.
2. A real-data run without explicit opt-in—or to an unexpected destination—must make zero outbound calls.
3. The importer must reject malformed records and leave the source snapshot byte-for-byte unchanged, including on interrupted import.
4. Abstentions, missing labels, and classes absent from a sample must not inflate accuracy/F1 or crash metrics.
5. Local UI requests from non-loopback origins or unexpected Host headers must not start model runs or reveal private data.

---

## File Structure

```text
pyproject.toml                         package/dependency/test configuration
.gitignore                             deny-by-default for local data/config/results
src/s1lab/__init__.py                  package version
src/s1lab/config.py                    environment/config parsing and privacy guards
src/s1lab/schemas.py                   typed internal task, response, dataset, label, run types
src/s1lab/systemone/client.py          sole SystemOne HTTP adapter
src/s1lab/systemone/contract.py        verified endpoint request/response mapping
src/s1lab/systemone/fake.py            deterministic fake transport for failure tests
src/s1lab/store/sqlite.py              SQLite connection/schema and CRUD
src/s1lab/importers/seerscore.py       read-only snapshot loader and baseline labels
src/s1lab/experiments/runner.py        explicit-opt-in run orchestration and audit event
src/s1lab/labeling/queue.py            fixed-seed sample and blind labeling operations
src/s1lab/evaluation/metrics.py        per-class metrics, abstention/coverage, bootstrap CI
src/s1lab/cli.py                        smoke/import/sample/run/eval/config commands
src/s1lab/web/app.py                   loopback-only FastAPI routes
src/s1lab/web/templates/               dashboard, labeling, run results
fixtures/synthetic/                    synthetic-only public cases and gold labels
 tests/                                 offline unit, contract, and web tests
```

## Task 1: Package scaffold and fail-safe configuration

**Files:**
- Create: `pyproject.toml`, `.gitignore`, `src/s1lab/__init__.py`, `src/s1lab/config.py`, `src/s1lab/schemas.py`
- Create: `tests/test_config.py`, `tests/test_schemas.py`, `tests/test_privacy_guards.py`

**Interfaces:**
- `AppConfig.load(env: Mapping[str, str] | None = None) -> AppConfig`
- `AppConfig`: `base_url: str | None`, `model: str`, `data_dir: Path`, `allow_real_data: bool`, `timeout_s: float`
- `authorize_run(config: AppConfig, dataset_kind: Literal["synthetic", "real"], dataset_hash: str, destination: str, fields: Sequence[str], confirmed: bool = False) -> RealDataConsent`
- `RealDataConsent`: `dataset_hash: str`, `destination_id: str`, `fields: tuple[str, ...]`, `confirmed_at: datetime`, `mode: Literal["synthetic", "environment_opt_in", "ui_confirmation"]`
- Internal `TaskRequest`, `TaskResult`, `RunManifest`, `ClaimRecord`, `HumanLabel`; do not mirror assumed SystemOne transport JSON here.

- [ ] **Step 1: Write failing config/privacy tests**
  - `test_defaults_do_not_allow_real_data`: empty env gives `allow_real_data is False` and no endpoint.
  - `test_opt_in_requires_explicit_true`: only `S1LAB_ALLOW_REAL_DATA=1` enables real-data mode.
  - `test_real_dataset_without_opt_in_is_rejected`: raises `PermissionError` before any client can be constructed.
  - `test_synthetic_dataset_needs_no_real_data_opt_in`: synthetic path is permitted.
  - `test_data_dir_inside_repo_is_rejected`: path under checkout raises `ValueError`.
  - `test_config_repr_redacts_secret_values`: a fake API key never appears in repr or redacted output.
  - Schema tests reject out-of-range confidence, empty labels, and invalid task values; abstention is represented distinctly from a label.

- [ ] **Step 2: Run tests and verify RED**
  - Run: `pytest tests/test_config.py tests/test_schemas.py tests/test_privacy_guards.py -q`
  - Expected: FAIL because package/config/schema do not exist.

- [ ] **Step 3: Implement minimal package, config, and schemas**
  - Use standard library dataclasses or Pydantic consistently; reject unknown/missing configuration rather than defaulting to a real endpoint.
  - Add deny-by-default ignores for `.env*`, local config, `data/`, `*.db`, `*.jsonl`, `logs/`, `exports/`, and local results.
  - The privacy guard must run before network construction; it must not log claim text, credentials, or endpoint secrets.

- [ ] **Step 4: Run tests and verify GREEN**
  - Run: `pytest tests/test_config.py tests/test_schemas.py tests/test_privacy_guards.py -q`
  - Expected: all tests pass; `python -m compileall src/s1lab` exits 0.

- [ ] **Step 5: Commit**
  - `git add pyproject.toml .gitignore src/s1lab tests && git commit -m "feat: add safe SystemOne lab foundation"`

## Task 2: SystemOne contract probe and typed client

**Files:**
- Create: `src/s1lab/systemone/__init__.py`, `client.py`, `contract.py`, `fake.py`
- Create: `tests/systemone/test_client.py`, `tests/systemone/test_contract.py`, `tests/systemone/test_fake.py`
- Modify: `pyproject.toml` for `httpx` and test extras; `src/s1lab/schemas.py` only if the verified contract requires an internal field.

**Interfaces:**
- `SystemOneClient(base_url: str, model: str, timeout_s: float, transport: httpx.BaseTransport | None = None)`
- `SystemOneClient.probe() -> SystemOneHealth`
- `SystemOneClient.ask(request: TaskRequest) -> TaskResult`
- `map_systemone_request(request: TaskRequest, model: str) -> dict[str, object]`
- `parse_systemone_response(payload: object, task: TaskRequest) -> TaskResult`

- [ ] **Step 1: Run T0 contract probe before coding**
  - Use only a synthetic sentence with an explicitly configured endpoint and model.
  - Record observed request/response keys and types locally; verify that the typed answer and probability fields are understood. Do not record a private endpoint, credentials, or real claim text.
  - If the live contract does not support the proposed task/labels, stop and revise the adapter design before implementation.

- [ ] **Step 2: Add sanitized synthetic contract fixture and write failing tests**
  - Save only a scrubbed response for the synthetic prompt to `tests/fixtures/systemone/synthetic_response.json` after verifying it contains no account, host, or private data.
  - `test_maps_smoke_question_to_verified_systemone_request`: exact request mapping matches the verified contract.
  - `test_parses_valid_typed_response`: verifies label, probability (when actually supplied), and abstention/status handling.
  - `test_malformed_or_empty_response_is_schema_error`: returns explicit invalid result, never a default label.
  - `test_http_4xx_is_not_retried` and `test_transient_5xx_retries_with_cap`: attempts never exceed configured cap.
  - `test_timeout_returns_explicit_failure`: timeout does not fall through to another model.
  - `test_request_uses_only_allowed_fields`: no transcript excerpt is sent by default.
  - `test_client_does_not_log_key_or_claim_text`: captured logs contain neither.

- [ ] **Step 3: Run tests and verify RED**
  - Run: `pytest tests/systemone -q`
  - Expected: FAIL because the client/contract adapter does not exist.

- [ ] **Step 4: Implement adapter and fake transport**
  - Map internal `TaskRequest` to the verified request shape; validate raw response strictly and normalize to `TaskResult`.
  - Make retries bounded and limited to documented transient conditions. Concurrency starts at one; no hidden provider fallback.
  - The fake transport must script success, timeout, malformed body, 4xx, and 5xx without network access.

- [ ] **Step 5: Run tests and verify GREEN**
  - Run: `pytest tests/systemone -q`
  - Expected: all offline tests pass; explicit real-endpoint smoke test reports model, elapsed time, schema status, and redacted destination; no real-data opt-in is needed for synthetic content.

- [ ] **Step 6: Commit**
  - `git add pyproject.toml src/s1lab/systemone src/s1lab/schemas.py tests/systemone tests/fixtures/systemone && git commit -m "feat: add verified SystemOne client"`

## Task 3: Private SQLite store, immutable run records, and snapshot importer

**Files:**
- Create: `src/s1lab/store/__init__.py`, `store/sqlite.py`, `src/s1lab/importers/__init__.py`, `importers/seerscore.py`, `importers/measurement_baseline.py`, `src/s1lab/experiments/__init__.py`, `experiments/runner.py`
- Create: `tests/store/test_sqlite.py`, `tests/importers/test_seerscore.py`, `tests/importers/test_measurement_baseline.py`, `tests/experiments/test_runner.py`, `fixtures/synthetic/seerscore_shape.json`
- Modify: `.gitignore`, `src/s1lab/config.py` if required for private paths.

**Interfaces:**
- `LabStore.open(path: Path) -> LabStore`; tables for `datasets`, `claims`, `runs`, `responses`, `labels`, and `audit_events`.
- `import_snapshot(path: Path, store: LabStore, individual_name: str | None = None) -> ImportSummary`
- `replay_measurement_heuristic(claim: str) -> Literal["quantitative", "subjective"]`
- `ImportSummary`: record count, snapshot SHA-256, counts by heuristic label, rejected-record count.
- `create_run(store: LabStore, manifest: RunManifest) -> str`
- `RunManifest`: dataset hash, task/schema version, model identifier, config hash, seed, created timestamp.

- [ ] **Step 1: Write failing store/import/privacy tests**
  - `test_import_filters_requested_individual_and_preserves_baseline_as_heuristic` uses synthetic data only and asserts exact imported count/labels.
  - `test_import_records_source_hash_and_does_not_modify_input` compares SHA-256 and mtime before/after.
  - `test_private_import_is_read_only_and_never_uses_network` proves import only reads local filesystem; no model/client calls occur.
  - `test_replays_each_legacy_measurement_pattern` covers four-digit years, dollar signs, percentages, amount words, election/signing verbs, and date-window phrases; a date lacking a four-digit year remains subjective under the old rule.
  - `test_malformed_records_are_counted_not_silently_dropped` reports rejected count and reason class without logging record text.
  - `test_interrupted_import_is_atomic` leaves no partial dataset/run.
  - `test_run_manifest_is_immutable_and_reproducible` stores exact hashes and versions.
  - `test_audit_event_contains_metadata_not_claim_text_or_secret` checks destination identifier, field names, confirmation, and absence of raw text/key.
  - CI fixture includes no real claim text, ID, source URL, or person-specific content.

- [ ] **Step 2: Run tests and verify RED**
  - Run: `pytest tests/store tests/importers tests/experiments -q`
  - Expected: FAIL because store/importer/runner are not implemented.

- [ ] **Step 3: Implement private store and snapshot importer**
  - Store DB and logs only under configured data directory outside repository.
  - Use SQLite transactions and idempotent import keys; never write to the imported file.
  - Preserve current measurement label with explicit source marker `heuristic`; do not call it gold truth.

- [ ] **Step 4: Implement guarded run creation/audit**
  - Before any real-data read or model request, require explicit opt-in, display endpoint destination and fields, and persist operator confirmation in private audit storage.
  - Synthetic runs proceed by default; real data with missing consent fails without opening the network.

- [ ] **Step 5: Run tests and verify GREEN**
  - Run: `pytest tests/store tests/importers tests/experiments -q`
  - Expected: all pass; working tree contains no imported dataset or SQLite artifacts.

- [ ] **Step 6: Commit**
  - `git add .gitignore src/s1lab/store src/s1lab/importers src/s1lab/experiments tests/store tests/importers tests/experiments fixtures/synthetic && git commit -m "feat: add private snapshot import and run store"`

## Task 4: Blind labeling UI and CLI workflows

**Files:**
- Create: `src/s1lab/labeling/queue.py`, `src/s1lab/cli.py`, `src/s1lab/web/__init__.py`, `web/app.py`, `web/templates/base.html`, `dashboard.html`, `label.html`, `run.html`
- Create: `tests/labeling/test_queue.py`, `tests/web/test_app.py`, `tests/test_cli.py`
- Modify: `pyproject.toml` for FastAPI/Jinja2/test client and console entry point.

**Interfaces:**
- `build_sample(claim_ids: Sequence[str], strata: Mapping[str, str], seed: int, size: int) -> list[str]`
- `submit_label(store: LabStore, claim_id: str, task: str, label: str | None, unsure: bool, guideline_version: str) -> None`
- FastAPI `create_app(config: AppConfig, store: LabStore) -> FastAPI`
- CLI commands: `s1lab config show`, `s1lab smoke`, `s1lab import`, `s1lab sample`, `s1lab label`, `s1lab eval`.

- [ ] **Step 1: Write failing sample/label/web tests**
  - `test_sample_is_stratified_deterministic_and_without_replacement` returns identical IDs for the same seed.
  - `test_label_submission_records_guideline_version_and_unsure` stores blind human labels.
  - `test_label_page_hides_systemone_and_heuristic_outputs_until_submit` verifies blinding.
  - `test_web_binds_loopback_by_default_and_rejects_untrusted_host` blocks non-loopback host requests and mutating cross-origin requests.
  - `test_dashboard_contains_status_and_no_private_text_in_summary` checks public-safe summary behavior.
  - `test_cli_import_and_smoke_help_are_available` checks command parsing without making network calls.

- [ ] **Step 2: Run tests and verify RED**
  - Run: `pytest tests/labeling tests/web tests/test_cli.py -q`
  - Expected: FAIL before routes, templates, queue, or CLI exist.

- [ ] **Step 3: Implement blind sample queue and label persistence**
  - Keep model/heuristic answers out of labeling response until the human submission is stored.
  - Support label, unsure, skip, and flag-for-adjudication actions with keyboard-accessible forms.

- [ ] **Step 4: Implement loopback web UI and CLI**
  - Routes use shared service functions; server defaults to `127.0.0.1`; settings redact secrets; real-data actions display the destination and fields and require opt-in.
  - CLI smoke/import/sample commands reuse the same config/privacy guards.

- [ ] **Step 5: Run tests and verify GREEN**
  - Run: `pytest tests/labeling tests/web tests/test_cli.py -q`
  - Expected: all pass; Playwright/HTTP smoke through synthetic label flow performs no external requests.

- [ ] **Step 6: Commit**
  - `git add pyproject.toml src/s1lab/labeling src/s1lab/cli.py src/s1lab/web tests/labeling tests/web tests/test_cli.py && git commit -m "feat: add local blind labeling UI"`

## Task 5: Shadow runner, evaluation metrics, and comparison views

**Files:**
- Create: `src/s1lab/evaluation/__init__.py`, `evaluation/metrics.py`, `evaluation/sampling.py`, `experiments/shadow.py`, `web/templates/compare.html`
- Create: `tests/evaluation/test_metrics.py`, `tests/evaluation/test_sampling.py`, `tests/experiments/test_shadow.py`
- Modify: `src/s1lab/cli.py`, `src/s1lab/web/app.py`, `tests/web/test_app.py`.

**Interfaces:**
- `evaluate(gold: Sequence[HumanLabel], predicted: Sequence[TaskResult], seed: int, bootstrap_samples: int = 1000) -> EvaluationReport`
- `EvaluationReport`: per-class precision/recall/F1, macro-F1, confusion matrix, abstention rate, coverage, invalid rate, latency summary, and bootstrap confidence intervals.
- `run_shadow(store: LabStore, dataset_id: str, task: str, sample_ids: Sequence[str], client: SystemOneClient, consent: RealDataConsent) -> str`
- Shadow output is proposal-only; no writes to SeerScore and no automatic destructive action.

- [ ] **Step 1: Write failing metrics/sampling/shadow tests**
  - `test_hand_computed_confusion_precision_recall_macro_f1` asserts exact values on a synthetic fixture.
  - `test_abstentions_count_as_uncovered_not_correct` ensures selective metrics do not inflate accuracy.
  - `test_missing_class_does_not_crash_metrics` returns defined zero/undefined treatment per documented rule.
  - `test_bootstrap_intervals_repeat_for_fixed_seed` returns identical bounds for identical inputs.
  - `test_shadow_run_requires_real_data_consent` makes zero fake/real client calls when consent is missing.
  - `test_public_safe_export_contains_no_claim_text_ids_urls_or_hosts` scans serialized export.

- [ ] **Step 2: Run tests and verify RED**
  - Run: `pytest tests/evaluation tests/experiments/test_shadow.py -q`
  - Expected: FAIL because metrics/shadow APIs do not exist.

- [ ] **Step 3: Implement deterministic metrics and shadow orchestration**
  - Count abstentions and invalid responses separately; compute coverage and answer-only accuracy; use seeded bootstrap resampling for intervals.
  - Store each result and request metadata privately; export only aggregate safe data unless the user explicitly selects a private local export.

- [ ] **Step 4: Implement compare view and CLI evaluation**
  - Show confusion matrix, per-class metrics, coverage/abstention, errors/latency, and disagreements with both human labels and heuristic baseline.

- [ ] **Step 5: Run the full verification suite**
  - Run: `pytest -q`
  - Expected: all offline tests pass; no tests contact a real endpoint; secret scan and synthetic-fixture allowlist pass.
  - Manual T0: `s1lab smoke` against configured endpoint with a synthetic request; verify observed contract, redacted destination, model, latency, and no data opt-in requirement.
  - Manual T3/T5: only after explicit confirmation, import private snapshot and run a small fixed-seed sample; confirm source hash unchanged and all output remains outside Git.

- [ ] **Step 6: Commit**
  - `git add src/s1lab/evaluation src/s1lab/experiments src/s1lab/web src/s1lab/cli.py tests && git commit -m "feat: add shadow evaluation and metrics"`

---

## Self-Review

- **Spec coverage:** privacy boundary and opt-in are covered by Tasks 1 and 3; live SystemOne contract and synthetic smoke by Task 2; private import/baseline by Task 3; UI/blind labeling by Task 4; shadow evaluation/metrics by Task 5. Calibration/model comparison and production write-back are intentionally deferred per spec.
- **Step scan:** each task has explicit failing tests, RED run, implementation, GREEN run, and commit; each interface is defined before downstream use.
- **Type consistency:** `TaskRequest`/`TaskResult` flow from schemas to `SystemOneClient.ask`; `RunManifest` and `LabStore` are defined in Task 1/3 before runner/UI use; `HumanLabel` is used by labeling and evaluation.
- **Review Focus tests:** malformed API payload/HTTP status in Task 2; opt-in and destination/host guard in Tasks 1/3/4; malformed/interrupted imports and unchanged source in Task 3; abstention/missing-class behavior in Task 5; Host/CORS/loopback checks in Task 4.
- **Proportion:** five cohesive tasks yield an offline-tested core and UI, with all real-model/data behavior isolated behind explicit, manually verified gates.
