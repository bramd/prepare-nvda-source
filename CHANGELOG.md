# Changelog

## v2.0.0

- Add a `path` input controlling where NVDA is cloned. A relative value resolves against the
  workspace, an absolute value is used as-is. The default is `nvda`, so the clone location is
  unchanged from v1. Set `path: ../nvda` to keep the NVDA tree out of a repository that is
  checked out at the workspace root.
- **Breaking:** `nvda-path` is now an absolute path with forward slashes, instead of the
  workspace-relative literal `nvda`. This matches how other actions expose a path
  (`actions/setup-python`'s `python-path`, `actions/setup-java`'s `path`), and it stays correct
  from any working directory. It still works as a step `working-directory`, and in bash, pwsh
  and cmd (quote it in cmd); only an expression that prefixes it, such as
  `../${{ steps.prepare.outputs.nvda-path }}`, needs updating.
- **Breaking:** existing NVDA and SCons MSVC caches are not reused. `actions/cache` folds the
  cached paths into a cache entry's identity, so v1 entries are unreachable rather than
  restored to the wrong place; the first run after upgrading is a cold build.
- **Breaking:** `install-vs-components` now defaults to `false`. The GitHub-hosted Windows
  images already carry the components NVDA's `.vsconfig` declares: `windows-2022` has all
  nine, and `windows-2025` has eight, the ninth being a Windows 11 SDK that `nvaccess/nvda`
  builds without. Running `vs_installer` was a no-op that still cost several minutes per cold
  build. Set the input to `true` on a runner that genuinely lacks a component.
- Move the SCons MSVC config cache from the workspace root to `RUNNER_TEMP`. It was the one
  other file the action left in the caller's checkout.
- Clone NVDA with `git clone` instead of `actions/checkout`, which rejects any path resolving
  outside `GITHUB_WORKSPACE`. The clone is shallow with shallow submodules, matching what
  checkout did here. It clones the commit SHA already resolved by the `Get NVDA commit SHA`
  step, so a moving ref such as `master` can no longer advance between the two and store a
  tree under a cache key naming a different commit. This uses `git clone --revision`, so the
  runner needs Git 2.49 or newer; every current GitHub-hosted Windows image qualifies.
- Hash NVDA's `.vsconfig` with `sha256sum` rather than `hashFiles()` for the SCons MSVC cache
  key, since `hashFiles()` only globs inside the workspace and silently returns an empty string
  for anything outside it.

## v1.2.0

- Make `python-version` and `python-arch` optional. When either is empty, the action auto-detects it
  from NVDA's root `.python-versions` file at the target ref (`cpython-<X.Y.Z>-windows-<arch>-none`,
  first line = primary, `x86_64` → `x64`). Since NVDA's uv is `python-preference = only-system`, this
  keeps the installed interpreter in sync with the exact patch NVDA pins, instead of hard-coding a
  version that drifts. Explicit inputs still override; consumers targeting NVDA ≥ 2025.3 can now pass
  just `nvda-ref` + `github-token`.
- Auto-detection requires NVDA ≥ 2025.3 (when `.python-versions` was introduced). For older refs the
  step fails with a clear message telling you to pass `python-version`/`python-arch` explicitly.
- Backward compatible: explicit callers are unaffected, and their cache keys are unchanged (the
  resolved values echo the inputs verbatim).

## v1.1.3

- Correct the Visual Studio 2026 guidance: `windows-2025-vs2026` is a **standard, generally-available**
  GitHub-hosted image (VS 2026 Enterprise pre-installed), not an org-private runner. To build frontier
  NVDA, target `runs-on: windows-2025-vs2026` with `install-vs-components: false` — no self-hosted or
  custom runner needed. Updates README, CHANGELOG, and the input descriptions accordingly.
- Add a `master` self-test entry that builds on `windows-2025-vs2026` with
  `install-vs-components: false`, exercising the pre-baked-image path.

## v1.1.2

- Simplify the VS component install: run `vs_installer.exe` directly instead of via a `cmd.exe`
  wrapper, locate the installer via `${ProgramFiles(x86)}` rather than a hardcoded drive path, and
  express the deliberate double-run as a loop documenting why (the first pass may only self-update
  the installer).
- Remove the redundant `nvda-ref` output (it only echoed the caller's own input). The `nvda-path`
  and `cache-hit` outputs are unchanged.

## v1.1.1

- Install uv after the NVDA checkout/restore (with `cache-dependency-glob: nvda/uv.lock`) so
  `setup-uv` can key its cache on NVDA's lockfile, removing the "cache will never get invalidated"
  warning.

## v1.1.0

- Add `install-vs-components` input (default `true`) to skip the runtime VS component install on
  images that already ship the required Visual Studio toolset (e.g. the standard
  `windows-2025-vs2026` image), matching `nvaccess/nvda`'s own build.
- Add `vs-version` input to select a specific major VS version (`17` = VS 2022, `18` = VS 2026)
  when multiple are installed.
- Locate Visual Studio with `vswhere` instead of a hardcoded `2022\Enterprise` path, so the
  install works across Community/Enterprise/BuildTools and different VS versions.
- Include the VS version in the SCons MSVC config cache key.
- Document the standard-vs-custom-runner and fork tradeoffs.

## v1.0.0

Initial release of `prepare-nvda-source`, a composite action that builds a runnable NVDA source
tree for add-on CI.

- Checkout `nvaccess/nvda` at a given ref (recursive submodules), install matching Python
  (version + arch), install the VS 2022 components from NVDA's `.vsconfig`, and run `scons source`
  to produce a runnable NVDA clone with `nvda/.venv` and a compiled `source/` tree.
- Cache the built `nvda/` tree keyed by `<os>-nvda-<python-version>-<python-arch>-<nvda-sha>`.
- Cache SCons's detected MSVC environment (`SCONS_CACHE_MSVC_CONFIG`) on cold builds, keyed by
  runner OS, image version, and a hash of NVDA's `.vsconfig`, to skip repeated MSVC toolchain
  detection.
- Automatically install a secondary 32-bit (x86) Python for `x64` builds (NVDA's x64 builds
  contain 32-bit components); `x86` builds already have a 32-bit primary interpreter.
- Modeled on `nvaccess/nvda`'s current `testAndPublish.yml` build job: `actions/checkout@v7`,
  `actions/cache@v5`, dual x64/x86 Python, `astral-sh/setup-uv@v7`, MSVC config cache, and a
  `scons-args` passthrough input (default `-j2`).
