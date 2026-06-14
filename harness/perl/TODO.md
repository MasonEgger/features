# Perl Compliance Harness — TODO

## Phase C0 — Skeleton
- [ ] C0.1 harness/perl: cpanfile + lib/Temporalio/Features/Harness.pm + runner.pl
- [ ] C0.2 Go: cmd/run_perl.go (BuildPerlProgram + RunPerlExternal)
- [ ] C0.3 Go: sdkbuild/perl.go (Carton build/prepare/NewCommand)
- [ ] C0.4 Go: cmd/run.go (normalize/expand/langFlag/Run `pl`) + cmd/prepare.go + cmd/latest_sdk_version.go (perl/MetaCPAN)
- [ ] C0.5 dockerfiles/pl.Dockerfile
- [ ] C0.6 Verify: `run --lang pl --version <sdk-perl path>` builds + runner connects to summary socket

## Phase C1 — Ruby-parity beachhead
- [ ] C1.1 activity/basic_no_workflow_timeout/feature.pl
- [ ] C1.2 signal/basic/feature.pl

## Phase C2 — DONE areas
- [ ] C2.1 activity/* (heartbeating, retry_on_error, cancel_*, running_concurrently, shutdown)
- [ ] C2.2 query/* (5 scenarios)
- [ ] C2.3 Harness per-feature custom data converter + data_converter/* (7 scenarios)
- [ ] C2.4 continue_as_new/{continue_as_same,signals_block_continue_as_new} + signal/{activities,prevent_close,signal_with_start}

## Phase C3 — Histories
- [ ] C3.1 generate + commit history.pl.<ver>.json for the green set

## Phase C4 — P6 (gated on sdk-perl Phase 6)
- [ ] C4.1 child_workflow/* (§18)
- [ ] C4.2 update/* (§19)
- [ ] C4.3 signal/{child_workflow,external,workflow_to_workflow} + continue_as_new/updates_do_not_block (§18/§19/§20)

## Phase C5+ — v0.2 areas (each gated on the matching sdk-perl phase)
- [ ] C5.1 local_activity/* (§21)
- [ ] C5.2 search_attributes/* (§24)
- [ ] C5.3 schedule/* (§25)
- [ ] C5.4 nexus/sync_success (§26)
- [ ] C5.5 client/http_proxy* + reset/reset_and_delete (§30)
- [ ] C5.6 telemetry/metrics (§28)
- [ ] C5.7 tracing + interceptors (§27)
- [ ] C5.8 eager_* (§23) + build_id/deployment versioning (§29)
