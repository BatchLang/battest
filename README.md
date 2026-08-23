# battest

Runtime test runner for Windows batch files (`.bat` / `.cmd`). battest launches
real `cmd.exe` and asserts on exit code, stdout, stderr, environment, filesystem
side effects, and calls to mocked external commands.

[![PyPI](https://img.shields.io/pypi/v/battest.svg)](https://pypi.org/project/battest/)
[![Python versions](https://img.shields.io/pypi/pyversions/battest.svg)](https://pypi.org/project/battest/)
[![CI](https://github.com/tboy1337/battest/actions/workflows/CI.yml/badge.svg)](https://github.com/tboy1337/battest/actions/workflows/CI.yml)
[![License](https://img.shields.io/pypi/l/battest.svg)](COPYING)

It is a **trusted-fixture runner**, not a sandbox. Destructive scripts can still
harm the host. Use `--safe-defaults` (or the GitHub Action, which enables it)
and a disposable VM or CI runner for untrusted suites. Details:
[Safety](docs/safety.md).

battest is a sibling of [Blinter](https://github.com/tboy1337/Blinter) (static
analysis). It does not depend on Blinter.

**Requirements:** Python 3.11+ and Windows for `battest run`.

## Features

- Real `cmd.exe` in an isolated temp workdir per case (Job Object, kill-on-close)
- Assertions: exit code, stdout/stderr, environment, and files
- PATH mocks for external commands (`ipconfig`, `reg`, …) with call recording
- Param overlays: one YAML document, many variants
- `setup` / `teardown`, stdin, env, and copy-in fixtures
- Parallel `--jobs`, JUnit XML, and a Windows GitHub Action
- Optional `--safe-defaults` PATH stubs for common destructive utilities

cmd.exe internals (`del`, `copy`, `rd`, …) cannot be shadowed via `PATH`. See
[Mocking](docs/mocking.md).

## Quick start

```text
pip install battest
```

Create `hello.cmd`:

```bat
@echo off
echo hello
exit /b 0
```

Create `hello.battest.yaml` next to it:

```yaml
description: hello prints hello
script: hello.cmd
expect:
  exit_code: 0
  stdout:
    contains: hello
```

Run:

```text
battest run hello.battest.yaml
```

`python -m battest` is the same as `battest`. A passing case prints `PASS`. A
failing case prints a diff and exits `1`. Invalid YAML or usage exits `2`.

Case-directory form is equivalent:

```text
tests/hello/input.cmd
tests/hello/expect.yaml
```

Then `battest run tests`. From this repository, `battest run examples` runs the
bundled fixtures, including a mocked `ipconfig /flushdns` script with param
overlays.

CLI `--safe-defaults` is **off**. The GitHub Action turns it **on**. That flag
PATH-stubs common destructive externals (`format`, `shutdown`, `reg`, and
others); it does not isolate the filesystem. See [CLI](docs/cli.md) and
[Mocking](docs/mocking.md).

### Mocking externals

PATH stubs replace named executables for the case. This fixture asserts
`ipconfig /flushdns` is invoked, then overlays a non-admin variant:

```yaml
description: flush DNS when admin
script: flush_dns.cmd
timeout_seconds: 15
mocks:
  net:
    exit_code: 0
  ipconfig:
    exit_code: 0
    expect_calls:
      - args_contains: "/flushdns"
  timeout:
    exit_code: 0
expect:
  exit_code: 0
  stdout:
    contains: Flushing DNS cache
params:
  - id: not-admin
    mocks:
      net:
        exit_code: 2
      ipconfig:
        expect_calls:
          - not_called: true
      timeout:
        exit_code: 0
    expect:
      exit_code: 1
      stdout:
        contains: administrator
```

Full field list: [Fixture format](docs/fixture-format.md). Bundled example:
[`examples/windowsrescue/`](examples/windowsrescue/).

## CLI

```text
battest [--version] run [path] [--junit-xml FILE] [--jobs N]
        [--timeout SECONDS] [--max-diff N] [--safe-defaults]
        [--no-safe-defaults] [-v]
```

| Flag | Meaning |
|---|---|
| `path` | Fixture file or directory. Default: `./tests` when it contains battest fixtures, otherwise the current directory |
| `--jobs` | Parallel case execution (each case has its own temp dir). 1–256 |
| `--timeout` | Default timeout when a case omits `timeout_seconds`. Default: `30` |
| `--junit-xml` | Write xunit2 JUnit XML |
| `--safe-defaults` | PATH-stub common destructive externals unless mocked or listed in `allow` |
| `-v` | Debug logging to stderr |

Exit codes: `0` all passed, `1` one or more FAIL/ERROR/TIMEOUT, `2` usage or
schema error. Full flag list: [CLI](docs/cli.md).

## GitHub Action

Requires a Windows runner. Use the moving major tag (`@v1`), not a commit SHA.

```yaml
jobs:
  test-batch:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v7
      - id: battest
        uses: tboy1337/battest@v1
        with:
          path: tests
          safe-defaults: "true"
      - uses: actions/upload-artifact@v7
        if: always()
        with:
          name: battest-junit
          path: ${{ steps.battest.outputs.junit-xml }}
```

Inputs, outputs, and `--` before `path` are documented in
[GitHub Action](docs/github-action.md).

## Installation

### pip (recommended)

```text
pip install battest
```

### Standalone executable (no Python)

Run this from **cmd.exe** (not PowerShell). It downloads the bootstrap script,
installs the latest `battest.exe` to `%LOCALAPPDATA%\Programs\battest\bin`,
adds that directory to your user `PATH`, and returns the installer exit code
after deleting the downloaded `.cmd`:

```text
curl -L https://raw.githubusercontent.com/tboy1337/battest/main/scripts/install_battest.cmd -o install_battest.cmd && call install_battest.cmd & set "BATTEST_INSTALL_EXIT=%ERRORLEVEL%" & del install_battest.cmd & exit /b %BATTEST_INSTALL_EXIT%
```

The installer always fetches the latest GitHub release and verifies the zip
SHA-256 digest before extract. Download URLs must be `https` on `github.com`,
`objects.githubusercontent.com`, or `release-assets.githubusercontent.com`.
The bootstrap `.cmd` itself is not digest-pinned; the exe payload is. Pinning
the curl URL to a release tag (instead of `main`) is stricter if you want a
known installer script. Restart the terminal or IDE after install so `PATH`
updates are visible.

**Manual zip:** download `Battest-vX.Y.Z.zip` from
[GitHub Releases](https://github.com/tboy1337/battest/releases) and run
`Battest-vX.Y.Z\battest.exe`. Some antivirus products flag PyInstaller unpacking
as a false positive. The source is public; pip avoids that class of heuristic.

### Uninstall

Standalone install (cmd.exe):

```text
curl -L https://raw.githubusercontent.com/tboy1337/battest/main/scripts/uninstall_battest.cmd -o uninstall_battest.cmd && call uninstall_battest.cmd & set "BATTEST_UNINSTALL_EXIT=%ERRORLEVEL%" & del uninstall_battest.cmd & exit /b %BATTEST_UNINSTALL_EXIT%
```

pip:

```text
pip uninstall battest
```

## Python API

```python
from battest import load_case, run_case, run_cases

cases = load_case("hello.battest.yaml")
result = run_case(cases[0], safe_defaults=False)
results = run_cases(cases, jobs=1, safe_defaults=False)
```

`run_case` / `run_cases` require Windows `cmd.exe`. `safe_defaults` defaults to
off, matching the CLI. Full notes: [CLI](docs/cli.md).

## Documentation

Getting started:

- [Fixture format](docs/fixture-format.md)
- [CLI](docs/cli.md)
- [GitHub Action](docs/github-action.md)

Behavior:

- [Mocking external commands](docs/mocking.md)
- [PATH mock stub crate](docs/stub.md)
- [Encoding](docs/encoding.md)
- [Safety](docs/safety.md)
- [Security](docs/SECURITY.md)
- [Changelog](docs/CHANGELOG.md)

## License

AGPL-3.0-or-later ([COPYING](COPYING)).
