# Abla Test Runner

`abla-test` is a bounded parallel test runner written in Abla. It launches
tests as isolated child processes, gives every step its own combined output
log, enforces timeouts, and keeps command arguments structured instead of
turning them into shell text.

The hosted process adapter uses POSIX facilities available on Linux and macOS.
The manifest and runner code are platform-neutral; Windows will need a hosted
process adapter implementing the same small Abla ABI.

## Why

Large Abla suites have traditionally accumulated shell scripts for compiling
fixtures, checking exit codes, managing parallel jobs, and printing failures.
That makes orchestration difficult to reuse and leaves most suites sequential.
`abla-test` moves those common mechanics into one tested runner:

- up to 64 concurrent test processes, defaulting to the online CPU count;
- sequential steps inside each test and parallel execution across tests;
- expected exit statuses, per-step timeouts, and process-tree termination;
- isolated stdout/stderr logs with failure output surfaced automatically;
- suite and step environment overrides;
- working directories without global `cd` state;
- validated test dependencies with transitive filtered selection;
- exclusive barriers for memory-heavy or stateful tests;
- platform selection, name filtering, listing, and fail-fast mode; and
- JSON manifests that can be generated and consumed on any host.

Commands are executed directly. Globbing, pipes, redirects, `$()` and other
shell syntax are never interpreted accidentally. A test that deliberately
needs a shell can declare `sh` and `-c` explicitly.

## Build

```sh
make build
./build/abla-test --help
```

`make check` builds the runner entirely through Abla and exercises manifest
validation, parallel execution, multiple steps, non-zero expected statuses,
exclusive tests, filtering, failure logs, and timeout termination.

## Manifest

Create `abla-tests.json` in the project root:

```json
{
  "version": 1,
  "jobs": 8,
  "timeoutMs": 300000,
  "logDirectory": "build/test-results",
  "environment": {
    "ABLA_MAX_MEMORY_MB": "4096"
  },
  "tests": [
    {
      "name": "crypto",
      "steps": [
        {
          "command": "${root}/../ablac/build/ablac",
          "args": [
            "build", "tests/crypto.ab",
            "-o", "build/crypto-test", "--fast", "--no-cache"
          ]
        },
        {
          "command": "${root}/build/crypto-test",
          "expect": 42,
          "timeoutMs": 20000
        }
      ]
    },
    {
      "name": "crypto-integration",
      "dependsOn": ["crypto"],
      "steps": [{
        "command": "${root}/build/crypto-test",
        "expect": 42
      }]
    },
    {
      "name": "database-integration",
      "exclusive": true,
      "platform": "linux",
      "steps": [{
        "command": "docker",
        "args": ["compose", "up", "--abort-on-container-exit"]
      }]
    }
  ]
}
```

A step can also check what it left behind, once its exit status matched: the step's
log, which holds stdout and stderr together, and files it was to write.

```json
{
  "command": "${root}/build/tool",
  "outputEquals": "exact log text\n",
  "outputContains": ["needle", "another"],
  "outputExcludes": ["unwanted"],
  "outputLineContainsInOrder": [["error[", "]:", "ticket"]],
  "outputLineMatches": ["^error\\[E_[A-Z]+\\]: .*ticket$"],
  "files": [
    {"path": "${log}/result.txt", "equals": "exact file text\n"},
    {"path": "${root}/build/report.txt", "contains": "ok"},
    {"path": "${root}/build/refused.out", "absent": true}
  ]
}
```

The log is the step's stdout and stderr in the order the process wrote them; a step that
writes to both streams through buffers can interleave them differently from run to run, so
prefer `outputContains` for such a step. Needles must not be empty strings (the manifest is
rejected).

`outputLineMatches` is a list of regular expressions (`abla/regex` syntax, RE2-like); each must match somewhere in at least one
single line of the log (`grep -E`'s test of a line: `^` and `$` are the line's ends and `.` does not match a newline), and a
manifest with a pattern that does not compile is rejected, with the compile error's message and offset. It needs a compiler
whose standard library has `abla/regex`.

`outputLineContainsInOrder` is a list of needle lists; each list must be found, in order, on one single
line of the log (what `grep 'a.*b'` checks), so needles on different lines don't match.

A missing file, a differing text or a needle that is absent (or, for `outputExcludes`,
present) fails the test with the reason, and its later steps don't run.

Supported substitutions in commands, arguments, directories, file paths and environment
values are `${root}`, `${test}`, and `${log}`. Named `@name@` substitutions can
be supplied with repeatable `--set name=value` options. `platform` may be
`any`, `linux`, or `macos`.

`dependsOn` accepts test names declared anywhere in the manifest. Dependencies
must pass before a dependent can start. Unknown names, duplicates, self
dependencies, and cycles are manifest errors. When a prerequisite fails or is
skipped, its dependents are skipped without being launched. Filtering a test
automatically selects its complete transitive dependency closure, so focused
runs remain self-contained:

```sh
abla-test --filter crypto-integration
```

The command above runs both `crypto` and `crypto-integration`, in dependency
order. `--list` applies the same closure and prints every test that would be
needed by the run.

Run it with:

```sh
abla-test
abla-test --jobs 16
abla-test --filter crypto
abla-test --set compiler=/opt/abla/bin/ablac
abla-test --list
abla-test --fail-fast
abla-test --root /path/to/project --file tests/native.json
```

The exit status is `0` when selected tests pass, `1` for test failures, and `2`
for command-line or manifest errors.

## Library use

The scheduling and manifest APIs are also importable:

```abla
import github("AndreBaltazar8/abla-testrunner")

fun main: int = runTestCommand(["--file", "tests/service.json"])
```

## Migration strategy

Start by expressing each existing script as a test with direct command steps.
This immediately centralizes parallelism, timeouts, exit-code checks, and logs.
Then move shell-only setup into explicit Abla fixtures or reusable package APIs
and connect consumers with `dependsOn`. Keep genuinely platform-specific
integration tests marked with `platform` rather than branching throughout the
suite.

Licensed under the MIT License.
