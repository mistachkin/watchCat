# watchCat

**Out‑of‑process log watcher for Eagle test runs.**
Automatically detects hangs, logs progress locally and remotely, and (when needed) terminates stuck test processes. Includes a small cron helper to roll up batch results.

> **What it is**
>
> * A tiny set of Eagle/Tcl scripts:
>
>   * `packages/watchCat.eagle` — library of integration helpers (`Eagle.Test` procedure hooks, automatic `watchCat.tool.eagle` execution, etc).
>   * `packages/watchCat.library.eagle` — library of tool helpers (file change detection, “is log complete?” checks, kill helpers, remote/local logging, etc).
>   * `tools/watchCat.tool.eagle` — the log monitor you run **beside** your test process. It watches a single log file and a target PID; if the log stops changing too long, it declares the run hung and kills the process. It can also periodically “ping” your remote server via Eagle’s test logging.
>   * `tools/watchCron.eagle` — a simple post‑run/cron summarizer: counts `OVERALL RESULT` lines in a log, finds unique test‑run IDs, and sends a single “TOTALS” record to your remote logger.
> * Ship as Eagle packages: **`Eagle.WatchCat 1.0`** and **`WatchCat.Library 1.0`** (pkgIndex provided).

---

## Why you might want this

* **Catches deadlocks & infinite waits** by watching the log file’s mtime. If nothing’s written for > *limit* seconds, watchCat treats the run as hung.
* **Terminates hangs with authority** (Windows: uses `isActiveProcess`/`kill`; POSIX: uses system `kill`, optionally the whole process group).
* **Writes clear breadcrumbs** both **locally** (into your test log via `tlog`) and **remotely** (via `logRemoteMessage`), including “PING”, “HUNG”, “KILL”, “CRASH”, and “DONE”.
* **Zero changes to your tests** when you already use Eagle’s test harness (`package require Eagle.Test`).

---

## Repository layout

```
watchCat/
├─ packages/
│  ├─ pkgIndex_*.eagle            # Declares Eagle.WatchCat & WatchCat.Library
│  ├─ watchCat.eagle              # Test Package Integration Procedures (wiring, hooks)
│  └─ watchCat.library.eagle      # Tool Library Procedures (internal API)
└─ tools/
   ├─ watchCat.tool.eagle         # Primary CLI tool: watch one log + PID
   └─ watchCron.eagle             # Secondary CLI tool: cron batch roll-up
```

Packages are recognized as `Eagle.WatchCat 1.0` and `WatchCat.Library 1.0` via the pkgIndex; `WatchCat.Library` is loaded with `sourceWithInfo` depending on Eagle patch level.

---

## Prerequisites

* **Eagle** (the Extensible Adaptable Generalized Logic Engine) with the **Eagle.Test** package available at runtime. watchCat explicitly requires `Eagle.Test`.
* **POSIX**: access to the `kill` tool (used for liveness checks and termination). **Windows**: the harness provides `isActiveProcess`.
* Your test suite produces a standard Eagle test log containing lines like `OVERALL RESULT: ...` and remote logging summaries; watchCat parses these to decide if the run “completed”.

---

## Installing

You have three straightforward options. Pick one:

### Option A — Submodule or copy

Add `watchCat` to your repository under, say, `externals/watchCat`, and point Eagle’s `auto_path` at `watchCat/packages`.

### Option B — Use the environment discovery baked into the tools

`watchCat.tool.eagle` will add itself to `auto_path` from either of these variables, if present:

* `XDG_WATCHCAT_HOME`
* `XDG_STARTUP_HOME/LoadOnStartup/Public/WatchCat`

i.e., you can drop the repository under `${XDG_STARTUP_HOME}/LoadOnStartup/Public/WatchCat` and it will be found automatically.

### Option C — Manual `auto_path`

Before sourcing, add the packages path:

```tcl
lappend auto_path [file join $::env(PROJECT_ROOT) externals watchCat packages]
```

---

## Quick start

### 1) Start your test run and capture the PID

**Projects using Eagle test-suite infrastructure:**

```tcl
# The test-suites that need WatchCat should do the following (at some point):

set ::env(SCRATCH_ROOT) /full/path/to/watchCat/packages
set ::env(XDG_WATCHCAT_HOME) /full/path/to/watchCat/tools

lappend ::auto_path $::env(SCRATCH_ROOT)

package require Eagle.WatchCat
hookGetTestLogForWatchCat true true

# After this, just run the test-suite as usual, e.g. via "all.eagle".
```

Under the hood watchCat will:

* `package require WatchCat.Library` and `Eagle.Test` (so it can write to your test log and talk to your remote logger).
* Treat the first arg as the **log file** and the second as the **target PID** (both validated).
* Loop forever while the process is alive:

  * If a **tagged “kill” file** appears (`<log>.killProcess` or `<log>.killProcess<PID>`), it will log the event and kill the process immediately.
  * If the log **hasn’t changed** and the **elapsed time** exceeds `limit`, it logs “HUNG” locally & remotely and kills the process (optionally the whole process group on POSIX).
  * If `ping > 0`, it periodically **PINGs** your remote logger with elapsed status.
* On exit, it verifies the log “looks complete” (i.e., contains a valid `OVERALL RESULT` and a successful remote logging line) and logs **DONE** or **CRASH** accordingly.

---

## Configuration & tuning

Set these **Eagle variables *in the watchCat interpreter*** before it starts looping (for example via a wrapper script that sets variables then `source`s the tool):

| Variable     | Default            | Meaning                                                                                                                    |
| ------------ | ------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| `limit`      | 3600 seconds (1h)  | “Consider it hung” threshold based on log file mtime. (Author has a long “overnight” variant on some hosts.) |
| `ping`       | 1200 seconds (20m) | Interval for remote **PING** logging; `0` disables.                                                          |
| `aggressive` | `false`            | If `true`, attempt an extra kill after success to mop up edge cases (e.g., Mono lingering on exit).          |
| `verbose`    | `false`            | Enables additional tracing (ties into `enableTracing` if available).                                         |
| `quiet`      | `true`             | Suppresses non‑error output from the tool itself.                                                            |
| `retries`    | `3`                | Overrides web request retries for remote logging (via `Utility.SetWebMaximumRetries`).                       |

**Wrapper example** (recommended):

```tcl
# watchCat.wrap.eagle
# Configure first, then source the tool:
set limit   1800          ;# 30 minutes
set ping    600           ;# 10 minutes
set verbose true
set quiet   false

# Find watchCat (use XDG_WATCHCAT_HOME if you installed via Option B)
set here [file dirname [info script]]
lappend auto_path [file normalize [file join $here .. packages]]

source [file join $here watchCat.tool.eagle]
```

---

## Safety “tags” (control via files)

watchCat checks for special files next to your log (or script) to alter behavior at runtime. Create an **empty file** with the appropriate suffix to trigger:

| Tag file                        | Effect                                                                    |
| ------------------------------- | ------------------------------------------------------------------------- |
| `<log>.killProcess`             | Kill the watched process now (logs event first).            |
| `<log>.killProcess<PID>`        | Same, but only for a matching PID (useful if PIDs recycle). |
| `<log>.noKillProcess`           | Disable **all** kill attempts (safety interlock).           |
| `<script>.noKillHungProcess`    | Disable “hung” kill path (diagnostics).                     |
| `<script>.noKillProcessAndSelf` | Disable the rare “kill both target and self” path.          |

> These files are checked with a simple `file exists` + tag regex match and are intended as emergency brakes while a run is in progress.

---

## “Is my log complete?” — how watchCat decides

The library considers a log **complete** only when both are true:

1. There’s an `OVERALL RESULT: (SUCCESS|FAILURE|STOP-ON-FAILURE|STOP-ON-LEAK|NONE)` line (SUCCESS/FAILURE are “terminal”; others are flagged).
2. There’s a line confirming **remote result logging** (OK/ERROR/FAILURE) in the expected format.

It also extracts and reports **skipped counts**/names if present. All of this is implemented with explicit regex patterns.

---

## Cron / batch roll‑ups (`tools/watchCron.eagle`)

Use **after** a batch of runs (e.g., nightly CI) to send an aggregate “TOTALS” message to your remote logger.

**Usage**

```bash
dotnet exec EagleShell.dll -file tools/watchCron.eagle path/to/nightly.log
```

What it does:

* Scans the log for all `OVERALL RESULT:` entries, tallies counts for `SUCCESS`, `FAILURE`, `STOP-ON-FAILURE`, `STOP-ON-LEAK`, `NONE`, plus `UNKNOWN`.
* Extracts unique test‑run IDs and reports `RUNS=<count>`.
* Uses `env(BATCH_ID)` if set; otherwise tries to extract one from the log (pattern like `BATCH-ID <64-hex>`).
* Emits a single remote `logRemoteMessage`:
  `BATCH BATCH-ID <bid> TOTALS TEST-RUN: host <host> counts <...>`.

---

## Integration details with **Eagle.Test**

watchCat assumes the standard Eagle test harness:

* It requires `Eagle.Test` so it can use `tlog`, `getTestLog`, `logRemoteMessage`, and friends. It sets `::test_log` to the watched file and attempts to set up the `TEST-RUN` tag if your harness exposes `setupTestRunTag`.
* It can **trace** using `dtrace` or `debug trace` when available (`verbose` mode).
* On exit, it hooks `PreInterpreterDisposed` to ensure a final cleanup attempt can be made (e.g., logging a late HUNG and trying to kill).

> If your remote logging requires specific API keys or server configuration, keep those in your existing Eagle.Test setup; watchCat simply **piggy‑backs** calls like `logRemoteMessage` and will ignore failures via `catch`.

---

## Examples

### (POSIX) Kill hung test groups

If your test launcher puts the run in its own **process group**, watchCat’s POSIX path can kill the **entire group** on a HUNG (uses `kill -s KILL` or a harness‑provided `maybeKillProcessGroup`).

```bash
# In your launcher, create a process group first, then record the shell PID
# ...
eagle tools/watchCat.tool.eagle "$LOG" $TEST_PID &
```

### (Windows) Normal kill path

On Windows, liveness is checked via `isActiveProcess`, and watchCat uses Eagle’s `kill -force` command when it must terminate.

---

## Troubleshooting

* **“Can’t find package WatchCat.Library”**
  Ensure `auto_path` includes `watchCat/packages`, or install under `XDG_WATCHCAT_HOME`/`XDG_STARTUP_HOME/LoadOnStartup/Public/WatchCat`.

* **“unknown command tlog / logRemoteMessage”**
  Your Eagle.Test package isn’t available. Add `package require Eagle.Test` to the runner environment; watchCat itself requires it.

* **“It won’t kill my process”**
  Check for safety tags like `.noKillProcess`. On POSIX, verify the `kill` tool exists and that the watchCat process has permission to signal the target.

* **“It thinks the run is incomplete”**
  Make sure your harness prints a valid `OVERALL RESULT:` line and that the remote logging completion line is present; that’s how completeness is determined.

* **Increase verbosity**
  Set `verbose true` and `quiet false`, and (if present) call `enableTracing` from your harness to get richer traces.

---

## Public API (library highlights)

You most likely won’t call these directly, but for completeness:

* `hasFileChanged file ?varName?` — track mtime per file.
* `isProcessHung limit file ?varName? ?quiet?` — stale‐log detector.
* `isLogFileComplete file ?stageVar?` — log completeness heuristic (regexes).
* `killProcess pid ?deadVar? ?force? ?group?` and `killHungProcess file pid ?deadVar? stage` — termination helpers.
* `logPing|logDone|logCrash|logKillProcess|logHungProcess` — local + remote logging shims around your Eagle.Test integration.

---

## Contributing

* The repository is brand new (initial import + minor follow‑up commit for syntax highlighting). PRs welcome for docs, new heuristics, and portability improvements.

---

## License

Each source file contains a standard header referencing `license.terms`. If that file is not present in your checkout, consult the author’s licensing terms as indicated in the headers.

---

## Acknowledgements

Authored by **Joe Mistachkin** (Mistachkin Systems), also the author of **Eagle**.

---

### Appendix: Exact CLI usage

**watchCat**

```text
dotnet exec EagleShell.dll -file tools/watchCat.tool.eagle <fileName> <pid>
```

* Validates `<fileName>` exists and `<pid>` is an integer.

**watchCron**

```text
dotnet exec EagleShell.dll -file tools/watchCron.eagle <fileName>
```

* Validates `<fileName>` exists; aggregates results; logs one “BATCH TOTALS” record using `env(BATCH_ID)` or a parsed value.
