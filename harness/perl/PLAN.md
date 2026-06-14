# Perl Compliance Harness — Build Plan

Derived from `SPEC.md`. The harness skeleton (Go-side + Perl-side) lands first,
then `feature.pl` files area-by-area. Each `feature.pl` translates the existing
`feature.go`/`feature.py` for that scenario into the real sdk-perl API and
calls `register_feature`. Acceptance per area = the Go runner reports `PASSED`
for those dirs over the summary socket.

## Phase C0 — Harness skeleton (no features)
1. `harness/perl/{cpanfile, lib/Temporalio/Features/Harness.pm, runner.pl}`
   (register_feature DSL, registry, default start/check, SkipFeature, summary
   JSONL over tcp:// + file://, async loop wrapper).
2. Go side: `cmd/run_perl.go` (BuildPerlProgram + RunPerlExternal),
   `sdkbuild/perl.go` (Carton build/prepare/NewCommand),
   `cmd/run.go` (normalize/expand/langFlag/Run switch `pl`),
   `cmd/prepare.go` (`pl` case), `cmd/latest_sdk_version.go` (`perl`/MetaCPAN),
   `dockerfiles/pl.Dockerfile`.
   Acceptance: `go run . run --lang pl --version <sdk-perl path>` builds and
   the runner connects to the summary socket (no feature.pl yet).

## Phase C1 — Ruby-parity beachhead
3. `features/activity/basic_no_workflow_timeout/feature.pl`
4. `features/signal/basic/feature.pl`

## Phase C2 — DONE areas
5. `activity/*` — heartbeating, retry_on_error, cancel_*, running_concurrently,
   shutdown.
6. `query/*` — successful_query, timeout_due_to_no_active_workers,
   unexpected_arguments/query_type_name/return_type (adds query check patterns).
7. **Harness: per-feature custom data converter** (D6) + `data_converter/*` —
   binary, binary_protobuf, codec, empty, failure, json, json_protobuf.
8. `continue_as_new/{continue_as_same,signals_block_continue_as_new}`;
   `signal/{activities,prevent_close,signal_with_start}`.

## Phase C3 — Histories
9. `run --generate-history --version <pinned>` for the green set; commit
   `features/<dir>/history/history.pl.<ver>.json`.

## Phase C4 — P6 areas (gated on sdk-perl Phase 6)
10. `child_workflow/*` (§18) — extend Harness/runner for child handles.
11. `update/*` (§19) — extend the runner for update execution/validators.
12. `signal/{child_workflow,external,workflow_to_workflow}` (§18/§20);
    `continue_as_new/updates_do_not_block_continue_as_new` (§19).

## Phase C5+ — v0.2 areas (each gated on the matching sdk-perl phase)
13. `local_activity/*` (§21) · `search_attributes/*` (§24) ·
    `schedule/*` (§25) · `nexus/sync_success` (§26) · `client/http_proxy*` +
    `reset/reset_and_delete` (§30) · `telemetry/metrics` (§28) · `tracing` +
    `interceptors` (§27) · `eager_*` (§23) · versioning (§29).

## Success Metrics
- C0: a Perl run builds and reports over the summary socket.
- C1/C2: ~26 v0.1 scenarios PASSED; histories generated (C3).
- C4+: P6 and v0.2 scenarios land as their sdk-perl phases are accepted.
- Partial coverage is the supported state (a missing feature.pl is simply not
  run for Perl).
