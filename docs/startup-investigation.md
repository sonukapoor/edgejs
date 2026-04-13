# Startup Investigation

## Scope

This document maps the Edge.js startup path from process entry to first useful JS execution, compares it conceptually with Deno's startup architecture, records measured hotspots, and ranks realistic optimization paths.

Reference points used during this investigation:

- Edge entry and runtime bootstrap:
  - `src/edge_main.cc`
  - `src/edge_cli.cc`
  - `src/edge_runtime.cc`
  - `src/edge_option_helpers.h`
  - `lib/internal/bootstrap/node.js`
  - `lib/internal/bootstrap/switches/is_main_thread.js`
  - `lib/internal/process/pre_execution.js`
- Deno reference architecture:
  - `/Users/sonukapoor/Projects/deno/runtime/worker.rs`
  - `/Users/sonukapoor/Projects/deno/runtime/snapshot_info.rs`
  - `/Users/sonukapoor/Projects/deno/cli/snapshot/build.rs`
  - `/Users/sonukapoor/Projects/deno/cli/snapshot/README.md`

## What Changed In This Investigation

Two practical changes landed in this branch as part of the investigation:

1. `EDGE_STARTUP_TRACE=1` now includes CLI and runtime-environment setup phases, not only JS bootstrap phases.
2. `BuildEffectiveCliState()` now skips `cwd` resolution work when there are no `--env-file*` or `--experimental-config-file*` flags.

The second change is a low-risk startup optimization for common paths like `edge -e ""` and `edge script.js`.

## Current Startup Path

### 1. Process and CLI entry

`src/edge_main.cc` calls `uv_setup_args()`, initializes CLI process state, then dispatches into `EdgeRunCli()`.

`src/edge_cli.cc` is responsible for:

- compat dispatch checks
- safe-mode argument scan
- CLI argument parsing and mode selection
- building effective option state via `BuildEffectiveCliState()`
- applying V8 flags and env-file/config-file derived env updates
- dispatching to one of:
  - eval
  - repl/stdin
  - file entry
  - test runner
  - watch mode
  - `--run` package script mode

### 2. Runtime environment setup

`RunWithFreshEnv()` in `src/edge_cli.cc` performs:

- OpenSSL initialization
- N-API embedder hook installation
- N-API environment creation
- runtime attachment
- runtime platform hook installation

This happens before `RunScriptWithGlobals()` starts the existing native startup trace in `src/edge_runtime.cc`.

### 3. Native runtime/bootstrap setup

`RunScriptWithGlobals()` in `src/edge_runtime.cc` performs:

- stdio and signal setup
- runtime platform hooks
- timers host initialization
- flag parsing from source / secure heap config
- process object installation
- module loader installation
- per-context bootstrap builtins
- realm bootstrap
- `internal/bootstrap/node`
- web-exposed bootstrap builtins
- thread/process-state switch bootstrap
- dispatch to the selected built-in main or user script

### 4. JS bootstrap and main-thread setup

The heaviest JS-side startup work is currently in:

- `lib/internal/bootstrap/node.js`
- `lib/internal/bootstrap/switches/is_main_thread.js`
- `lib/internal/process/pre_execution.js`

Important eager work on the main-thread path includes:

- `internal/timers`
- `internal/source_map/source_map_cache`
- `buffer`
- `internal/process/execution`
- `internal/modules/*`
- `internal/process/pre_execution`
- `internal/dns/utils`
- `internal/modules/run_main`

`lib/internal/bootstrap/switches/is_main_thread.js` also eagerly requires several modules specifically to prepare main-thread execution:

- `fs`
- `util`
- `url`
- `internal/modules/cjs/loader`
- `internal/modules/esm/utils`
- `internal/perf/utils`
- `internal/modules/run_main`
- `internal/dns/utils`
- `internal/process/pre_execution`

That is an important Deno comparison point: Deno explicitly snapshots or lazy-initializes many equivalent runtime extensions instead of paying for them on every launch.

## Measured Startup Breakdown

All numbers below are from local repeated runs on this checkout using `EDGE_STARTUP_TRACE=1`.

### Pre-runtime host phases now visible

Representative medians for `edge -e ""`:

- `cli.build-effective-state`: ~`0.03ms`
- `cli.apply-v8-flags`: ~`0.30ms`
- `cli.env.openssl-init`: ~`0.70ms`
- `cli.env.create-napi-env`: ~`1.23ms`

Representative medians for `edge benchmarks/workloads/empty-startup.js`:

- `cli.build-effective-state`: ~`0.06ms`
- `cli.apply-v8-flags`: ~`0.36ms`
- `cli.env.openssl-init`: ~`0.74ms`
- `cli.env.create-napi-env`: ~`1.38ms`

### Runtime / bootstrap hotspots

Representative medians for `empty-startup.js`:

- `bootstrap.realm`: ~`3.0ms`
- `bootstrap.node.top-level.require-internal-timers`: ~`2.9ms`
- `bootstrap.per_context.primordials`: ~`2.6ms`
- `bootstrap.node.source-map-support`: ~`2.2ms`
- `bootstrap.switch.thread`: ~`2.1ms`
- `bootstrap.node.fatal-exception-hooks`: ~`1.4ms`
- `bootstrap.web.exposed-wildcard`: ~`1.3ms`
- `bootstrap.node.setup-buffer.require-buffer`: ~`0.8ms`

Important note:

- Several earlier JS lazy-load experiments reduced one trace bucket while simply shifting cost into another phase.
- This repo now has enough startup tracing to catch that effect, which is why several prior candidates were reverted.

## Biggest Startup Suspects

### 1. Runtime environment creation before JS runs

The newly visible pre-runtime phases show that some startup time is spent before `RunScriptWithGlobals()` begins:

- OpenSSL initialization
- N-API environment creation
- runtime attachment/hook installation

This is not a Deno-style JS bootstrap problem. It is an embedder/host startup problem.

### 2. `internal/bootstrap/node` still does expensive top-level work

The largest repeated JS-side costs remain:

- `internal/timers`
- source-map setup
- fatal exception hooks
- process methods / buffer setup

We already tested several lazy-load variants here. Most reduced trace buckets but did not produce a stable wall-clock win.

### 3. Main-thread bootstrap eagerly prepares more than the immediate command needs

`lib/internal/bootstrap/switches/is_main_thread.js` eagerly prepares module-loader, DNS, perf, and pre-execution wiring before command dispatch reaches most user code.

This is conceptually the closest place to Deno's "snapshot more, execute less at launch" strategy.

### 4. `internal/process/pre_execution.js` contains a large amount of mode-agnostic setup

It performs a wide range of initialization:

- trace/inspector hooks
- network inspection
- navigator/web APIs
- warning handling
- SQLite/QUIC/web storage/websocket/eventsource setup
- reporting / permission setup
- source maps / deprecations / config file support
- DNS initialization
- module loader initialization
- proxy setup

A Deno-style takeaway is not "rewrite this in Rust", but rather:

- split what truly must happen before first useful execution
- snapshot or defer everything else

## Deno Comparison: What Transfers

### What Deno is doing conceptually

Deno's runtime path makes three ideas very explicit:

1. The worker can start from a prebuilt startup snapshot.
   - `runtime/worker.rs` exposes `WorkerOptions.startup_snapshot`.
2. The CLI snapshot is built ahead of time.
   - `cli/snapshot/build.rs` creates the runtime snapshot during build.
3. A large set of runtime extensions are organized around lazy initialization.
   - `runtime/worker.rs` uses many `lazy_init()` extensions.
   - `runtime/snapshot_info.rs` lists what is baked into the snapshot.

This means Deno tries to move work from:

- launch time

into:

- build time
- snapshot time
- first-use time

### Transferable ideas for Edge

These ideas are directly transferable in principle:

1. Build-time snapshot pipeline
- Edge already inherits Node-style startup snapshot APIs in `lib/internal/v8/startup_snapshot.js`.
- What is missing is a Deno-like, first-class Edge build pipeline that intentionally snapshots a stable subset of bootstrap work.

2. Stricter split between snapshotted work and lazy work
- Deno's extension list makes it obvious which runtime parts are eager, snapshotted, or lazy.
- Edge currently has a lot of main-thread/bootstrap work that is still eager and not cleanly partitioned that way.

3. Minimize work before first useful execution
- Deno treats pre-user-code work as a cost center and moves eligible setup into snapshot creation.
- Edge can apply the same principle even if the implementation remains C++ + JS instead of Rust + extensions.

4. Lazy-init non-critical subsystems
- Deno uses `lazy_init()` repeatedly for subsystems that do not need to be fully hot before first execution.
- Edge should keep applying that idea, but only where phase tracing plus end-to-end benchmarking agree.

## Deno Comparison: What Does Not Transfer Cleanly

These pieces are not directly portable:

1. Tokio / Rust async orchestration
- Edge is not structured as a Rust runtime with extension registration.
- Reproducing Deno's worker/runtime layering directly would be a rewrite, not an optimization.

2. Deno extension graph and op registration model
- Deno's startup can be reshaped extension-by-extension.
- Edge's Node-compat bootstrap and built-in module graph are much more intertwined.

3. Full snapshot-first CLI behavior as a short-term patch
- Deno already has a dedicated snapshot build pipeline.
- Edge has snapshot-related primitives, but not yet the same operational path for building, validating, and shipping an Edge-specific startup snapshot.

## Low-Risk Startup Win Implemented Here

### Optimization

File:

- `src/edge_option_helpers.h`

Change:

- `CollectEnvFileSpecs()` now returns early unless the raw argv actually contains an `--env-file*` option.
- `CollectConfigFileSpecs()` now returns early unless the raw argv actually contains an `--experimental-config-file*` option.

Why this helps:

- Before this change, simple startup paths still resolved the current working directory even when no env-file or config-file feature was in use.
- That work happened before runtime setup and before JS bootstrap, so it was a small but real tax on every common launch path.

### Measurement support added

Files:

- `src/edge_cli.cc`
- `src/edge_process.h`
- `src/edge_process.cc`

Change:

- `EDGE_STARTUP_TRACE=1` now emits CLI/env setup phases as well as the existing bootstrap phases.

Examples of new phases:

- `cli.parse-argv`
- `cli.build-effective-state`
- `cli.apply-v8-flags`
- `cli.env.openssl-init`
- `cli.env.openssl.configure.begin`
- `cli.env.openssl.configure.end`
- `cli.env.openssl.csprng-check.begin`
- `cli.env.openssl.csprng-check.end`
- `cli.env.create-napi-env`
- `cli.env.attach-runtime.environment`
- `cli.env.attach-runtime.process-exit-handler`
- `cli.env.attach-runtime.cleanup-stages`

This closes an important observability gap.

## Host-side Follow-up

This pass focused on the host/runtime setup path before JS bootstrap:

- `cli.env.openssl-init`
- `cli.env.create-napi-env`
- runtime attachment and hook installation

It added finer-grained trace points and separated OpenSSL configuration from the explicit CSPRNG validation step.

### New host-side trace phases

For eval and script-entry paths, `EDGE_STARTUP_TRACE=1` now exposes:

- `cli.env.openssl.use-precomputed-effective-state`
- `cli.env.openssl.build-effective-state.begin`
- `cli.env.openssl.build-effective-state.end`
- `cli.env.openssl.configure.begin`
- `cli.env.openssl.configure.end`
- `cli.env.openssl.csprng-check.begin`
- `cli.env.openssl.csprng-check.end`
- `cli.env.install-napi-hooks`
- `cli.env.create-napi-env`
- `cli.env.attach-runtime.environment`
- `cli.env.attach-runtime.process-exit-handler`
- `cli.env.attach-runtime.cleanup-stages`
- `cli.env.install-runtime-hooks`

Representative single-run trace for `edge -e ""` on this branch:

- `cli.env.openssl.configure`: about `0.30ms`
- `cli.env.openssl.csprng-check`: about `0.46ms`
- `cli.env.create-napi-env`: about `1.47ms`
- `cli.env.attach-runtime.*`: each sub-step is small, but they are now visible separately

Representative single-run trace for `edge --help` on this branch:

- `cli.env.openssl.configure`: about `0.55ms`
- `cli.env.openssl.csprng-check`: about `0.68ms`
- `cli.env.create-napi-env`: about `4.66ms`

The absolute values vary from run to run, but the phase ordering is stable.

### What OpenSSL initialization is doing today

The OpenSSL hot path currently comes from `RunWithFreshEnv()` in `src/edge_cli.cc`, which calls:

- `EdgeInitializeOpenSslForExecArgv()` in `src/edge_runtime.cc`
- then `EdgeValidateOpenSslCsprng()`

The work is:

1. inspect effective exec args and process environment for OpenSSL-related configuration
   - `--openssl-config`
   - `--openssl-shared-config`
   - `OPENSSL_CONF`
2. call `OPENSSL_init_crypto(OPENSSL_INIT_LOAD_CONFIG, ...)`
3. explicitly verify that a usable CSPRNG exists via `ncrypto::CSPRNG(nullptr, 0)`

That means the current startup cost is not just "crypto might be needed later"; it also preserves fail-fast behavior when the configured OpenSSL provider/config leaves the runtime without a usable RNG.

### What depends on OpenSSL

Direct or likely dependencies include:

- `crypto`
- TLS / HTTPS / secure sockets
- any subsystem depending on OpenSSL provider/config state
- any path that relies on a working secure random source through Node-compatible crypto assumptions

What this host-side pass did not find:

- evidence that `--version` intrinsically needs OpenSSL
- evidence that help text generation intrinsically needs OpenSSL

### Do trivial paths truly require OpenSSL on the hot path?

#### `edge --version`

No.

`--version` already has a native fast path in `EdgeRunCli()` and returns before OpenSSL initialization, N-API environment creation, or JS bootstrap.

Observed trace:

- `cli.enter`
- `cli.initialize-process`
- `cli.scan-safe-mode`
- `cli.dispatch.version`

#### `edge --help`

Not semantically, but currently yes in the shipped control flow.

The observed current path still goes through runtime/bootstrap rather than using a native help renderer, so it ends up paying:

- OpenSSL setup
- N-API environment creation
- runtime attachment

I prototyped a `--help` path that skipped OpenSSL while still using the JS help renderer. It was behaviorally safe, but repeated benchmarks did not show a win, so it was reverted.

Measured A/B for the reverted `--help` fast path:

- baseline: `45.4 ms ± 1.4`
- prototype: `48.4 ms ± 0.6`
- result: regression, not kept

#### `edge -e ""`

On the current architecture, yes.

Even a trivial eval still requires:

- runtime/bootstrap
- N-API environment creation
- runtime attachment

OpenSSL is currently also initialized before eval runs.

Could OpenSSL be deferred here?

- maybe, but not yet safely proven
- only if no OpenSSL-related flags or env are active, and if fail-fast semantics around provider/config/CSPRNG are preserved or intentionally changed

At the moment, this remains a riskier optimization than it first appears.

### Do trivial paths truly require N-API environment creation on the hot path?

#### `edge --version`

No.

#### `edge --help`

Under the current implementation, yes.

The help path still runs through the JS/runtime layer rather than a native formatter, so N-API environment creation remains on the hot path.

#### `edge -e ""`

Yes.

Eval requires the JS runtime, so N-API environment creation and runtime attachment are fundamental unless the execution model changes more substantially.

### Runtime attachment / hook installation findings

`EdgeAttachEnvironmentForRuntime()` in `src/edge_environment_runtime.cc` currently performs:

- environment attachment
- process exit handler setup
- cleanup-stage registration for workers, runtime platform, and c-ares

Each of those sub-steps is small individually, but tracing them separately is useful because:

- it confirms they are not the primary host-side bottleneck
- it isolates the larger host-side buckets more clearly:
  - OpenSSL configuration + RNG validation
  - N-API environment creation

### Safe fast-path optimization outcome

What landed:

- no additional host-side fast path beyond the earlier `BuildEffectiveCliState()` optimization
- finer-grained host-side tracing
- reuse of precomputed effective CLI state so OpenSSL setup does not rebuild it unnecessarily

What did not land:

- a `--help` OpenSSL-skip fast path

Why it was not kept:

- it did not improve startup in repeated A/B runs
- it would have added another special-case path without a measurable payoff

### Risk analysis for OpenSSL deferral

Deferring OpenSSL for trivial execution is attractive, but the risks are real:

- correctness:
  - OpenSSL config/provider failures would no longer fail before runtime entry on affected paths
- compatibility:
  - users relying on `OPENSSL_CONF` or OpenSSL CLI flags could see different failure timing
- security expectations:
  - the current eager CSPRNG validation is a deliberate fail-fast check

Conclusion:

- `--version` is already in the right shape
- `--help` would need a real native help path to remove runtime/N-API cost cleanly
- `-e ""` is not a safe candidate for blanket OpenSSL deferral without a more careful contract change

## Measured Effect Of The Implemented Change

Benchmark method:

- `hyperfine --warmup 10 --runs 80`
- baseline binary copied before the patch
- optimized binary copied after the patch

### `edge -e ""`

- before: `41.6 ms ± 1.3`
- after: `40.9 ms ± 1.0`
- effect: about `1.02x` faster (`~1.7%` improvement)

### `edge benchmarks/workloads/empty-startup.js`

- before: `40.4 ms ± 1.2`
- after: `40.2 ms ± 0.8`
- effect: about `1.01x` faster (`~0.5%` improvement, small but favorable)

Interpretation:

- This is a real but modest startup win.
- It improves common launch paths without changing runtime semantics.
- It does not solve the larger cold-start gap by itself.

## Recommendation: Are Startup Snapshots Realistic For Edge?

Short answer:

- yes as a medium/longer-term project
- no as a quick tactical patch

Why "yes":

- Edge already carries Node-style startup snapshot awareness in JS bootstrap code.
- The Deno comparison shows that moving bootstrap work into snapshot time is one of the most credible ways to attack launch overhead without endless micro-deferrals.

Why "not yet":

- There is no obvious Edge-first snapshot build pipeline comparable to Deno's `cli/snapshot/build.rs`.
- The current bootstrap graph mixes truly required startup work with command-specific and process-state-specific work.
- Without a clean partition, snapshotting more code is likely to be fragile.

Practical recommendation:

1. Use the expanded startup trace to define a snapshot candidate set.
2. Identify which bootstrap modules are deterministic and snapshot-safe.
3. Prototype a minimal Edge snapshot build path only after the host/runtime initialization cost is better understood.

## Recommendation: Would A Warm Daemon/Process Mode Help?

Short answer:

- yes for repeated CLI workflows
- no as a replacement for cold-start work

Where it would help:

- repeated `edge run ...` package-script workflows
- test/watch/dev-server style command loops
- commands that repeatedly rebuild environment/module state

Where it would not help enough:

- one-shot CLI runs used in baseline cold-start benchmarks
- fair runtime-comparison benchmarks where a persistent process would change the execution model

Recommendation:

- treat warm-daemon mode as a product/workflow optimization
- keep cold-start work separate so the benchmark story stays honest

## Ranked Opportunities

### 1. Host/runtime initialization before JS bootstrap

Effort:

- medium to high

Potential payoff:

- medium to high

Why:

- `cli.env.openssl-init` and `cli.env.create-napi-env` are now clearly measurable before JS bootstrap even begins.
- This is one of the few areas where we have not yet done serious optimization work.

### 2. Edge-specific startup snapshot pipeline

Effort:

- high

Potential payoff:

- high

Why:

- This is the most Deno-like strategic path.
- It attacks repeated launch-time JS/bootstrap work at the right conceptual layer.

### 3. Split `pre_execution` by command/path criticality

Effort:

- medium

Potential payoff:

- medium

Why:

- `lib/internal/process/pre_execution.js` still bundles a lot of work together.
- The likely win is not one lazy import, but a cleaner separation between:
  - required before any user code
  - required before file entry only
  - required before network/process-heavy modes only

### 4. Revisit eager main-thread bootstrap imports with the new trace

Effort:

- medium

Potential payoff:

- low to medium

Why:

- Prior lazy-load attempts often only shifted time.
- The new CLI trace plus existing JS bootstrap trace make these experiments safer, but expectations should stay modest.

### 5. Optimize `--run` and package-script mode separately

Effort:

- low to medium

Potential payoff:

- mode-specific medium

Why:

- `FindCliPackageJson()` walks parent directories, builds PATH prefixes, reads/parses `package.json`, and prepares a shell-spawn environment.
- That is fine for `--run`, but it should stay isolated and should not leak into the default file/eval path.

## What Did Not Change

This investigation did not change:

- JS bootstrap semantics in `lib/internal/bootstrap/node.js`
- the main-thread bootstrap module graph
- `pre_execution` behavior
- timers/bootstrap/module-loader internals
- snapshot generation/build plumbing

## Recommended Next Step

The next best experiment is:

1. keep the new CLI trace and host-side sub-phase breakdown
2. move to the Bun delta comparison next
3. revisit host-side deferral only if Bun comparison or future profiling points to a clear target with a measurable upside
4. only after that, evaluate whether an Edge-specific startup snapshot pipeline is worth prototyping

That sequencing is important.

Right now, the evidence suggests:

- JS micro-deferrals alone are unlikely to yield a large cold-start win
- host-side tracing is now good enough to evaluate future deferral ideas honestly
- the remaining high-value work is split between:
  - host/embedder startup
  - build-time snapshot/preinit strategy
