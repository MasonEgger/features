# Perl SDK Compliance Harness: Specification

Adds a **Perl** (`LANG=pl`, name `perl`) target to the Temporal `features`
cross-SDK compliance repo, so the Perl SDK ([`temporalio/sdk-perl`](https://github.com/temporalio/sdk-perl))
runs the same feature scenarios as the other SDKs. Resolved decisions
(D1-D7) are recorded inline; references are to the live runner code.

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
   `features/<dir>/history/history.<lang>.<ver>.json` if present: Perl ships
   none initially (the check is skipped when absent).

There is no gRPC/IPC plugin protocol: CLI args in, JSONL on a socket out, exit
code, server-observable history.

The Ruby harness (`features/harness/ruby/`) is the closest analog and
implements **only 2 of ~100 scenarios**: the precedent that a new SDK ships a
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

Summary-socket contract (verified against `cmd/run.go` `summaryServer`): the Go
side calls `l.Accept()` exactly once, then reads that single connection until
EOF. The runner MUST open one connection to `--summary-uri`, write every
feature's JSONL line to that same connection, and close it before exit (mirrors
Ruby's single `TCPSocket` + `summary_io.close`). Do not open a fresh connection
per feature: only the first connection is ever accepted, so per-feature
reconnects would drop all summaries after the first and the Go side would fall
back to "assume all passed" (`run.go` lines 388-394). The Go side blocks on the
summary channel after the runner process exits, so leaving the socket open (for
example in a child that outlives the parent) would wedge the run.

### 2.2 `lib/Temporalio/Features/Harness.pm`

Provides `register_feature(workflows => [...], activities => [...],
expect_result => $scalar, start => sub ($client, $task_queue, $feature) {...},
check_result => sub ($handle, $feature) {...}, data_converter => ...)` and the
registry keyed by feature dir (derived from the caller's filename). Defaults:
**start** the single registered workflow with id `<dir>-<uuid>`, the run task
queue, 60s execution timeout; **check_result** awaits the handle result and
compares to `expect_result`. A `SkipFeature` exception class maps to `SKIPPED`.

Naming note: the Ruby analog names this field `expect_run_result` (and also
carries an `expect_activity_error`), not `expect_result`; see
`harness/ruby/lib/harness.rb`. The Perl DSL deliberately shortens it to
`expect_result`. That rename is fine, but it is a divergence from the "mirror
Ruby" claim, so keep it intentional and consistent across `Harness.pm`,
`runner.pl`, and every `feature.pl`. See Open Questions Q1.

### 2.3 `cpanfile`

`requires 'Temporalio::SDK', '== 0.2.0';`: the `==` line is what
`BuildPerlProgram` scrapes when `--version` is absent.

Scrape mechanism: this cannot be a verbatim copy of `BuildRubyProgram`'s Gemfile
scrape. Ruby splits the line on `,` and trims quotes, which works for
`gem "temporalio", "~> 1.2"` but on `requires 'Temporalio::SDK', '== 0.2.0';`
would yield `== 0.2.0';` (leading `== ` and trailing `';`). `BuildPerlProgram`
needs its own regex that captures the version token out of the quoted
constraint and strips the `==`/whitespace/`;`. Confirm what the downstream
consumer expects: a bare `0.2.0` (like Ruby's extracted `~> 1.2`) or the full
`== 0.2.0` constraint. See Open Questions Q2.

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
  Local-path install gap: `BuildRubyProgram` handles `--version <local-path>`
  specially (it writes `gem "temporalio", path: <gemPath>` into the Gemfile and
  runs `rake compile` for the native extension; `sdkbuild/ruby.go` lines
  92-113, 148-162). A cpanfile `requires 'Temporalio::SDK'` line cannot express
  a local filesystem path, so `perl.go` needs an explicit local-path branch:
  install the SDK dist from the given directory (for example `cpanm --notest
  <path>` or `carton install` against a cpanfile that pins the dist via a local
  mirror), and build the Rust core bridge from that checkout. This matters
  because the C0 acceptance criterion is `run --lang pl --version
  <sdk-perl path>`, which is exactly the local-path case; without this branch,
  C0 cannot be met. See Open Questions Q3.
- `features/cmd/run.go` `case "pl"`: mirror the Ruby two-arg
  `sdkbuild.RubyProgramFromDir(dir, sourceDir)` shape, i.e.
  `sdkbuild.PerlProgramFromDir(filepath.Join(rootDir, DirName),
  filepath.Join(rootDir, "harness", "perl"))`. The second arg is the harness
  source dir that `NewCommand` uses to locate `runner.pl`.
- `features/cmd/run.go`: `normalizeLangName` add `perl→pl` + `pl` passthrough;
  `expandLangName` add `pl→perl`; `langFlag` usage add `pl`; the `Run` switch
  add a `case "pl"` with `sdkbuild.PerlProgramFromDir` + `RunPerlExternal`. Also
  update the `default`-branch error strings in `normalizeLangName` and
  `expandLangName` (they hardcode "must be one of: go or java or ts or py or cs
  or rb") and the `langFlag` Usage string, or the tool will advertise `pl` as
  supported in `--help` while omitting it from the error list. `latest_sdk_version.go`
  keys its map by the expanded name, so the new entry there must be `"perl"`
  (not `"pl"`), consistent with the existing `"ruby"` key.
- `features/cmd/prepare.go`: add `case "pl": p.BuildPerlProgram(ctx)`.
- `features/cmd/latest_sdk_version.go`: add a `"perl"` entry hitting the
  MetaCPAN release endpoint for the `Temporalio-SDK` dist (D4: confirm the
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
`:Defn('Name')` (per sdk-perl spec §10/§18-§19).

## 5. Scenario inventory & status

Status legend. Warning: an earlier draft of this section asserted "all sdk-perl
prerequisites have now landed: the SDK is v0.2 feature-complete." That is not
true. The sdk-perl adversarial audit
(`/home/mmegger/Code/.ai-sessions/step-45-sdk-perl-verified.md`, 2026-07-02)
confirmed 68 defects, several of which sit directly on the paths these
scenarios exercise. The labels below group scenarios by SDK area for
convenience, but they do not mean "will pass": read the SDK-Dependency Map
section before writing any `feature.pl` in an affected area. **DONE** = targets
the v0.1 surface (does not mean already green; nothing is implemented yet,
TODO shows 0 of 24 steps); **P6** = the Phase 6 area (child
workflows/updates/external handles, §18-§20); **v0.2** = other v0.2 areas
(§ in sdk-perl spec). A missing `feature.pl` simply means the scenario is not
run for Perl yet; partial coverage is the normal, supported state.

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

**Data-converter note:** the SDK's default converter encodes a plain Perl
string as `json/plain` (not `binary/plain`); raw bytes need an explicit
`Temporalio::Payload::RawBytes` wrapper to get `binary/plain`. Any
`data_converter` feature that asserts the encoding of a bare string must expect
`json/plain`.

## 6. Phasing

Sequencing correction: the phases below were originally written as if the SDK
had no remaining blockers. It does. Each phase is gated not only on harness work
but on the sdk-perl remediation IDs called out in the SDK-Dependency Map. For
every area, either land the cited fix in sdk-perl first, or land the
`feature.pl` and record it as a known-failing conformance gap (a red row in the
harness) rather than pretending it is green. Do not generate histories (Phase
C3) for any scenario that is not actually passing, because a history captured
from a buggy run pins the bug.

- **C0: harness skeleton (no features):** `harness/perl/` + `run_perl.go` +
  `sdkbuild/perl.go` + the `run.go`/`prepare.go`/`latest_sdk_version.go`
  registrations + `pl.Dockerfile`. Acceptance: `run --lang pl --version
  <local-path>` builds and the runner connects to the summary socket.
- **C1: Ruby-parity beachhead:** port the two Ruby has: 
  `activity/basic_no_workflow_timeout` and `signal/basic`.
- **C2: DONE areas:** `activity/*` → `query/*` → `data_converter/*` (adds a
  per-feature custom-data-converter surface to the harness, D6: new work,
  not just feature.pl) → the remaining DONE signal/CAN scenarios. This area is
  NOT uniformly green: only `activity/basic_no_workflow_timeout`, the
  successful/error-path `query/*`, and `data_converter/{json,json_protobuf}` are
  low-risk. `activity/{cancel_*,heartbeating,retry_on_error,running_concurrently}`,
  `data_converter/{empty,binary,failure}`, and `signal/signal_with_start` each
  hit a confirmed sdk-perl defect (see the SDK-Dependency Map). Expect roughly a
  dozen genuinely-green scenarios here, not 26, until the cited fixes land.
- **C3: generate histories** for the green set
  (`--generate-history --version <pinned>`).
- **C4: P6 areas** (gated on the SDK, not blocker-free): `child_workflow/*` →
  `update/*` → `signal/{child_workflow,external,workflow_to_workflow}`. `update/*`
  is heavily exposed (R9, L7, A1, R6, R8) and `signal/{external,child_workflow}`
  hits R4c/R10/R6. Sequence these after the corresponding sdk-perl fixes or mark
  them known-failing. See the SDK-Dependency Map.
- **C5+: v0.2 areas** (gated on the SDK, not blocker-free): port the remaining
  areas as their `feature.pl` files are written. `tracing` cannot pass at all
  until R1 lands (the interceptor is a non-functional stub); `local_activity/cancel_*`
  hit L8/L10; `search_attributes` + continue_as_new hit R11. See the
  SDK-Dependency Map.

## 7. Resolved decisions

- **D1** lang code `pl`/`perl` (no collision).
- **D2** version manifest: `cpanfile` pinned `requires`, regex-scraped.
- **D3** build tool: **Carton** (Bundler analog; lockfile; self-contained
  prepared dir for Docker).
- **D4** MetaCPAN dist name `Temporalio-SDK` (confirm on publish).
- **D5** `feature.pl` uses the real sdk-perl API (`:isa(...Definition)`,
  `:Run`/`:Signal`/`:Query`/`:Update`, `:Defn`).
- **D6** per-feature custom data converter is new harness surface: sequenced
  into C2.
- **D7** stage in this fork branch until proven (mirroring how Ruby landed
  with 2 features); upstream incrementally to temporalio/features later.

## 8. SDK-Dependency Map

Source of defect IDs: `/home/mmegger/Code/.ai-sessions/step-45-sdk-perl-verified.md`.
For each planned feature area this lists the confirmed-broken sdk-perl path, the
expected feature-test outcome, and the sequencing recommendation. "Gate on <ID>"
means sequence that `feature.pl` after the sdk-perl remediation for that finding
lands; "known-failing gap" means it is acceptable to land the `feature.pl` now
as a red conformance row, but it must not be counted green and must not have a
history generated (C3).

Activity cancellation (`activity/cancel_try_cancel`, `cancel_wait_cancel`,
`cancel_abandon`): R7 (WAIT_CANCELLATION_COMPLETED ignored for regular
activities; the future fails immediately and the eventual result is dropped),
R2 (shared `cancelled()` future is single-shot and poisoned by the wait_any
loser sweep), L15 + L13 (sync-activity cancel crosses the fork as a one-shot
boolean at dispatch, so a running forked activity never observes a later cancel;
combined, running sync activities are uncancellable). Outcome: `cancel_try_cancel`
and `cancel_wait_cancel` fail or hang; `cancel_abandon` is the least exposed.
Recommendation: gate on R7 + L15 + L13 (and R2 for the wait_any variants).

Activity heartbeat (`activity/heartbeating`): L13 (child heartbeats are relayed
only after the activity body completes; zero live heartbeats during execution).
Outcome: any assertion of mid-execution heartbeat delivery fails. Recommendation:
gate on L13.

Activity retry (`activity/retry_on_error`): L14 (cross-fork errors are
stringified and rewrapped as a generic retryable ApplicationError, so
non-retryable sync-activity errors retry forever). Outcome: a non-retryable
assertion fails or the run loops. Recommendation: gate on L14.

Activity concurrency/shutdown (`activity/running_concurrently`, `shutdown`):
L12 (the pool ADJUST publishes registry/FD/module lists into process-wide
`main::` globals; a second worker clobbers the first pre-fork, so pool A can
execute pool B's registry), L16 (pool children inherit all core/client gRPC
socket fds), L18 (`$pending` never failed/cleared at runtime shutdown; futures
awaited across shutdown hang forever). Outcome: flaky-to-failing under
concurrency; shutdown may hang. Recommendation: gate `running_concurrently` on
L12/L16 and `shutdown` on L18/L1.

Data converter (`data_converter/empty`, `binary`, `failure`, `codec`): ADJ1
(`from_payload` on an empty `json/plain` payload dies "malformed JSON string"),
L21 (wide-char UTF8 scalar in RawBytes: char-semantics length vs UTF-8 bytes at
the FFI, proto framing mismatch), R6 (the codec boundary covers only the v0.1
surface, so failure payloads and many others bypass the codec and encryption is
silently defeated), L22 (failure conversion happens outside DataConverter error
wrapping). Outcome: `empty` fails on ADJ1, `binary` fails on L21, `failure`
diverges on R6/L22; `codec` passes for plain workflow arg/result but silently
under-covers anything beyond the v0.1 surface. Recommendation: gate `empty` on
ADJ1, `binary` on L21, `failure` on R6/L22; land `codec` but note the R6 scope
gap. The existing json/plain-vs-binary/plain note in section 5 stands.

Signal with start (`signal/signal_with_start`): R12 (init signals drain only
after `:Run`'s synchronous prologue; Python runs handlers first, so
signal-with-start diverges). Outcome: divergence or failure. Recommendation:
gate on R12. `signal/prevent_close` shares the handler-drain risk; verify
against R12/R9.

Continue-as-new + search attributes (`continue_as_new/*`,
`search_attributes/{set,upsert}`): R11 (continue_as_new drops
`search_attributes` and `versioning_intent` despite the POD and proto fields),
R6 (upserts bypass the codec). Outcome: `continue_as_same` and
`signals_block_continue_as_new` likely pass (they do not carry search
attributes across the boundary); any CAN scenario that carries search
attributes, and `search_attributes/upsert` under encryption, fail. Recommendation:
gate search-attribute-carrying CAN and encrypted upsert on R11/R6.

Update (`update/*`, `continue_as_new/updates_do_not_block_continue_as_new`): R9
(workflow-failure completions clear `@commands`, dropping query/update
responses), L7 (evicting a run with an in-flight async `:Update` croaks in
`_settle_update`), A1 + ADJ2 (`workflow_failure_exception_types` and
`nondeterminism_as_workflow_fail` never reach live Runners; live/replay
divergence), R6 (update input bypasses the codec), R8 (patched/external-signal
bypass `_assert_writable` and queries never enter read-only context). Outcome:
`task_failure` and validation scenarios fail; async/self-update scenarios can
croak on eviction. Recommendation: gate `update/*` on R9 + L7 + A1; treat the
whole area as a later phase.

Child workflow and external signals (`child_workflow/*`,
`signal/{child_workflow,external,workflow_to_workflow}`): R10
(`_signal_child_workflow` lacks the on_cancel hook the external variant has; no
CancelSignalWorkflow on a race), R4c (`_apply_cancel_workflow` never sweeps
`%pending_external_signals` / `%pending_external_cancels`), R6 (child/nexus
results and signal-external args bypass the codec), L5 (start future resolved
with the owning handle creates an uncollectable cycle; a leak, not a failure).
Outcome: happy-path child start/result may pass; cancel/signal races fail or
leak. Recommendation: gate the external-signal and child-cancel scenarios on
R10/R4c; land happy-path child workflow but watch L5.

Local activity (`local_activity/*`): L8 + L10 (wait-type LA cancel parks
forever; duplicate RequestCancelLocalActivity with no at-most-once guard), A2
(no mandatory-timeout validation, so a missing timeout is an infinite-retry
hang). Note R7 says LAs do honor WAIT_CANCELLATION_COMPLETED, so LA cancel is
less broken than regular-activity cancel, but the wait-type variants still park.
Outcome: `cancel_*` and `shutdown_sends_cancellation` fail or hang; the
non_retryable_error_in_options scenario hangs on A2. Recommendation: gate
`local_activity/cancel_*` and `shutdown_sends_cancellation` on L8/L10, and the
timeout-validation scenarios on A2.

Tracing (`tracing`): R1 (= A3 = L30) (the OTel TracingInterceptor is a
non-functional stub; the POD claims spans + propagation that do not exist).
Outcome: cannot pass; there are no spans and no propagation to assert on.
Recommendation: known-failing gap until R1 is implemented. Do not attempt
`tracing` in C5 as if it were a normal port.

Telemetry metrics (`telemetry/metrics`): L28 (unbound metric records are dropped
despite the "never dropped" doc; the promised rate-limited warn is
unconditional). Outcome: passes if it only asserts counter/gauge deltas on
bound metrics; fails if it asserts an unbound metric. Recommendation: verify the
scenario's assertions against L28 before counting it green.

Nexus (`nexus/sync_success`): L20 (nexus `cancel_task` is a silent no-op) affects
cancellation only, so `sync_success` is expected to pass; L5 adds a leak on the
start path. Outcome: likely pass. Recommendation: proceed, but do not add a
nexus-cancel scenario until L20 is fixed.

Versioning (`build_id_versioning/*`, `deployment_versioning/*`): R11 also drops
`versioning_intent` on continue_as_new. Outcome: any versioning scenario that
routes through CAN fails. Recommendation: gate CAN-routed versioning scenarios
on R11.

Query (`query/*`): R8 (queries never enter the read-only context) is a
correctness gap but does not stop a basic query from returning. Outcome: the
five query scenarios are expected to pass. Recommendation: proceed; note R8 as a
latent conformance gap.

## Open Questions

- **Q1** `expect_result` vs Ruby's `expect_run_result` (plus the dropped
  `expect_activity_error`): confirm the Perl DSL rename is intentional and update
  every `feature.pl` and the default `check_result` to match. If activity-error
  assertions are needed for `activity/retry_on_error`-style scenarios, the
  harness needs an `expect_activity_error` analog too.
- **Q2** cpanfile version-scrape output form: does the downstream Carton path
  want a bare `0.2.0` or the full `== 0.2.0` constraint token? This drives the
  regex in `BuildPerlProgram`.
- **Q3** local-path SDK install for `--version <sdk-perl path>`: confirm the
  exact Carton/cpanm invocation that installs the SDK dist from a working-tree
  directory and builds the Rust core bridge, since the C0 acceptance criterion
  depends on it and cpanfile `requires` cannot name a local path.
- **Q4** D4 MetaCPAN dist name `Temporalio-SDK` is still unconfirmed until the
  dist is published; `latest-sdk-version --lang perl` will 404 until then.
- **Q5** Given the confirmed-broken paths, decide the policy for red rows: does
  this harness fork tolerate known-failing conformance gaps checked in as red,
  or must every committed `feature.pl` be green? The phasing in section 6 assumes
  the former; confirm with Mason.

## Plan Impact

These findings imply changes to `PLAN.md` / `TODO.md` that Mason should
regenerate (not edited here):

- The C0 skeleton step must add a local-path SDK install branch in
  `sdkbuild/perl.go` (Open Question Q3); the current C0.3 wording ("Carton
  build/prepare/NewCommand") does not cover it, yet C0.6 verifies exactly that
  path.
- The C0 Go step should also note updating the `normalizeLangName` /
  `expandLangName` error strings and `langFlag` usage, not only adding the
  switch cases.
- C2 is not "~26 green": split it so the confirmed-broken activity and
  data_converter scenarios (activity cancel/heartbeat/retry/concurrency,
  data_converter empty/binary/failure) are either sequenced behind the cited
  sdk-perl fixes or explicitly marked known-failing. The PLAN "Success Metrics"
  line "~26 v0.1 scenarios PASSED" should become roughly a dozen green plus a
  tracked set of red conformance gaps.
- C3 (generate histories) must run only over actually-green scenarios; add an
  explicit guard so no history is generated from a scenario that reveals a
  known SDK bug.
- C4 and C5 headers in PLAN/TODO ("SDK prerequisites landed; ready to
  implement" / "SDK is v0.2 feature-complete; ready to implement") are false and
  should be reworded to "gated on sdk-perl remediation; see SDK-Dependency Map",
  with `tracing` (R1) called out as a hard known-failing gap.
