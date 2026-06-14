# Perl SDK Compliance Harness — Specification

Adds a **Perl** (`LANG=pl`, name `perl`) target to the Temporal `features`
cross-SDK compliance repo, so the Perl SDK ([`temporalio/sdk-perl`](https://github.com/temporalio/sdk-perl))
runs the same feature scenarios as the other SDKs. Resolved decisions
(D1–D7) are recorded inline; references are to the live runner code.

## 1. How the cross-SDK runner works

The Go program (`main.go` → `cmd.Execute`) is language-agnostic about feature
execution: it prepares a per-language project, starts a dev server (+ Nexus
endpoint / HTTP-CONNECT proxy as needed), then **shells out to a per-language
runner process**, passing the features to run as CLI args and collecting
pass/fail over a TCP socket; afterward the Go side independently fetches and
diff-checks workflow history.

**Two reporting channels a harness MUST satisfy:**
1. **Summary JSONL over TCP** (authoritative): the runner connects to
   `--summary-uri tcp://host:port` and writes one
   `{"name":"<dir>","outcome":"PASSED|FAILED|SKIPPED","message":""}` per
   feature; `name` MUST equal the feature `dir`. On `SKIPPED` the Go side
   skips the history check.
2. **Process exit code**: non-zero if any feature failed.
3. **History diff** (Go-side, no harness involvement) against
   `features/<dir>/history/history.<lang>.<ver>.json` if present — Perl ships
   none initially (the check is skipped when absent).

There is no gRPC/IPC plugin protocol: CLI args in, JSONL on a socket out, exit
code, server-observable history.

The Ruby harness (`features/harness/ruby/`) is the closest analog and
implements **only 2 of ~100 scenarios** — the precedent that a new SDK ships a
small, growing subset.

## 2. `features/harness/perl/`

Mirror the Ruby layout, Perlified (Perl 5.38 floor, `feature 'class'`).

```
features/harness/perl/
├── cpanfile               # requires 'Temporalio::SDK', '== X.Y.Z'; (version scrape source, D2)
├── cpanfile.snapshot      # Carton lockfile (D3)
├── lib/Temporalio/Features/Harness.pm   # register_feature DSL + registry
└── runner.pl              # entry point invoked by the Go runner
```

### 2.1 `runner.pl` contract

Invoked as (mirrors `RunRubyExternal`):
`perl runner.pl --server HOST:PORT --namespace NS [--client-cert-path P
--client-key-path P --ca-cert-path P --tls-server-name N --http-proxy-url URL]
--summary-uri tcp://127.0.0.1:PORT <dir:taskqueue[:nexusEndpoint]> ...`

Responsibilities: parse args (`Getopt::Long`); open the summary sink
(`tcp://` and `file://`, via `IO::Socket::INET`); for each feature `require`
`features/<dir>/feature.pl`, look it up by `dir`, connect a
`Temporalio::Client` (with TLS + http-connect-proxy options), build a
`Temporalio::Worker`, run it, start the workflow (custom `start` or default),
run `check_result`; catch `SkipFeature` → `SKIPPED`, any other exception →
`FAILED` (+ message); write the JSONL entry and flush; exit non-zero if any
failed. Because sdk-perl is async over `IO::Async`/`Future::AsyncAwait`, the
worker-run/start/check sequence runs inside `my $loop = IO::Async::Loop->new;
$loop->await(...)`.

### 2.2 `lib/Temporalio/Features/Harness.pm`

Provides `register_feature(workflows => [...], activities => [...],
expect_result => $scalar, start => sub ($client, $task_queue, $feature) {...},
check_result => sub ($handle, $feature) {...}, data_converter => ...)` and the
registry keyed by feature dir (derived from the caller's filename). Defaults:
**start** the single registered workflow with id `<dir>-<uuid>`, the run task
queue, 60s execution timeout; **check_result** awaits the handle result and
compares to `expect_result`. A `SkipFeature` exception class maps to `SKIPPED`.

### 2.3 `cpanfile`

`requires 'Temporalio::SDK', '== 0.1.0';` — the `==` line is what
`BuildPerlProgram` scrapes when `--version` is absent.

## 3. Go-side additions

- `features/cmd/run_perl.go` (new): `(*Preparer).BuildPerlProgram` (scrape the
  cpanfile version when `--version` is absent; delegate to
  `sdkbuild.BuildPerlProgram`) and `(*Runner).RunPerlExternal` (same
  arg-assembly as `RunRubyExternal`, then `NewCommand(...).Run()`).
- `features/sdkbuild/perl.go` (new): the `sdkbuild.Program` impl using
  **Carton** (D3): generate a `cpanfile`, `carton install --path=<dir>/local`,
  `NewCommand` runs `carton exec perl runner.pl <args>`. Skip-if-installed by
  stat-ing `<dir>/local`. (sdk-perl pulls in an Alien Rust-core-bridge build,
  so the build image needs Rust + a C toolchain, like `rb.Dockerfile`.)
- `features/cmd/run.go`: `normalizeLangName` add `perl→pl` + `pl` passthrough;
  `expandLangName` add `pl→perl`; `langFlag` usage add `pl`; the `Run` switch
  add a `case "pl"` with `sdkbuild.PerlProgramFromDir` + `RunPerlExternal`.
- `features/cmd/prepare.go`: add `case "pl": p.BuildPerlProgram(ctx)`.
- `features/cmd/latest_sdk_version.go`: add a `"perl"` entry hitting the
  MetaCPAN release endpoint for the `Temporalio-SDK` dist (D4 — confirm the
  dist name on publish).
- `features/dockerfiles/pl.Dockerfile` (new): copy `rb.Dockerfile`, swap the
  base to `perl:5.40-bookworm`/`-slim`, keep the Rust install; entrypoint
  `run --lang pl --prepared-dir prepared`.

## 4. `feature.pl` convention

`features/<area>/<scenario>/feature.pl` declares workflow/activity classes
using the **real sdk-perl API** and calls `register_feature`. Translation of
`signal/basic`:

```perl
use v5.38; use warnings; use feature 'class'; no warnings 'experimental::class';
use Future::AsyncAwait;
use Temporalio::Workflow;
use Temporalio::Workflow::Definition;
use Temporalio::Features::Harness;

class BasicSignalWorkflow :isa(Temporalio::Workflow::Definition) {
    field $value = undef;
    async method run :Run('BasicSignalWorkflow') () {
        await Temporalio::Workflow::wait_condition(sub { defined $value });
        return $value;
    }
    method my_signal :Signal('mySignal') ($arg) { $value = $arg; return }
}

register_feature(
    workflows     => ['BasicSignalWorkflow'],
    expect_result => 'arg',
    start => sub ($client, $task_queue, $feature) {
        my $handle = await $client->start_workflow('BasicSignalWorkflow', [],
            id => 'signal-basic-' . uuid(), task_queue => $task_queue,
            execution_timeout => 60);
        await $handle->signal('mySignal', ['arg']);
        return $handle;
    },
);
```

The workflow base is `Temporalio::Workflow::Definition`; the entry point is
`:Run('Name')`; handlers are `:Signal('name')`/`:Query('name')`/(v0.2)
`:Update('name')`; activities are `:isa(Temporalio::Activity::Definition)` with
`:Defn('Name')` (per sdk-perl spec §10/§18–§19).

## 5. Scenario inventory & status

Status legend: **DONE** = v0.1 surface; **P6** = needs sdk-perl Phase 6
(child workflows/updates/external handles); **v0.2** = needs another v0.2
phase (§ in sdk-perl spec). A missing `feature.pl` simply means the scenario
isn't run for Perl — partial coverage is the normal, supported state.

- **DONE (~26):** `activity/*` (8), `query/*` (5), `data_converter/*` (7),
  `continue_as_new/{continue_as_same,signals_block_continue_as_new}` (2),
  `signal/{basic,activities,prevent_close,signal_with_start}` (4).
- **P6 (~29):** all `child_workflow/*` (13, §18); all `update/*` (12, §19);
  `signal/{child_workflow,external,workflow_to_workflow}` (3, §18/§20);
  `continue_as_new/updates_do_not_block_continue_as_new` (1, §19).
- **v0.2 other:** `local_activity/*` (15, §21); `schedule/*` (6, §25);
  `search_attributes/{set,upsert}` (2, §24); `client/http_proxy*` (4, §30);
  `nexus/sync_success` (§26); `reset/reset_and_delete` (§30);
  `telemetry/metrics` (§28); `tracing` (§27); `eager_*` (2, §23);
  `build_id_versioning/*` + `deployment_versioning/*` (§29).
- **N/A for Perl:** `bugs/go/*`, `snippets/*`.

## 6. Phasing

- **C0 — harness skeleton (no features):** `harness/perl/` + `run_perl.go` +
  `sdkbuild/perl.go` + the `run.go`/`prepare.go`/`latest_sdk_version.go`
  registrations + `pl.Dockerfile`. Acceptance: `run --lang pl --version
  <local-path>` builds and the runner connects to the summary socket.
- **C1 — Ruby-parity beachhead:** port the two Ruby has —
  `activity/basic_no_workflow_timeout` and `signal/basic`.
- **C2 — DONE areas:** `activity/*` → `query/*` → `data_converter/*` (adds a
  per-feature custom-data-converter surface to the harness, D6 — new work,
  not just feature.pl) → the remaining DONE signal/CAN scenarios. ~26 green.
- **C3 — generate histories** for the green set
  (`--generate-history --version <pinned>`).
- **C4 — P6 areas** (gated on sdk-perl Phase 6): `child_workflow/*` →
  `update/*` → `signal/{child_workflow,external,workflow_to_workflow}`.
- **C5+ — v0.2 areas**, each gated on the matching sdk-perl phase landing.

## 7. Resolved decisions

- **D1** lang code `pl`/`perl` (no collision).
- **D2** version manifest: `cpanfile` pinned `requires`, regex-scraped.
- **D3** build tool: **Carton** (Bundler analog; lockfile; self-contained
  prepared dir for Docker).
- **D4** MetaCPAN dist name `Temporalio-SDK` (confirm on publish).
- **D5** `feature.pl` uses the real sdk-perl API (`:isa(...Definition)`,
  `:Run`/`:Signal`/`:Query`/`:Update`, `:Defn`).
- **D6** per-feature custom data converter is new harness surface — sequenced
  into C2.
- **D7** stage in this fork branch until proven (mirroring how Ruby landed
  with 2 features); upstream incrementally to temporalio/features later.
