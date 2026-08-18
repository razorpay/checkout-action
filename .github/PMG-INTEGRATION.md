# PMG (SafeDep Package Manager Guard) — CI Integration Report

Reference: https://docs.safedep.io/package-security/pmg/github-actions

Scope: every job in every workflow under `.github/workflows/`, regardless of whether
that job installs packages. A job is only skipped where PMG **cannot** run without
breaking it.

## Summary

| Workflow | Jobs | PMG integrated | Skipped |
|---|---|---|---|
| `test.yml` | 4 | 4 | 0 (1 partial — see `test`) |
| `licensed.yml` | 1 | 1 | 0 |
| `pmg-test.yml` | 3 | 3 | 0 |
| `genesis.yml` | 1 | 0 | 1 (structurally impossible) |
| **Total** | **9** | **8** | **1** |

## The applied pattern

```yaml
permissions:
  contents: read
# ...
    - uses: actions/checkout@v2
    - name: Setup PMG proxy            # after toolchain setup, before installs
      uses: safedep/pmg@v1
      with:
        server-mode: true
        api-key: ${{ secrets.PMG_PUBLIC_REPOS_TOKEN }}
        tenant-id: ${{ secrets.PMG_TENANT_ID }}
    # ... build / test steps ...
    - name: Enforce PMG policy         # must be the LAST step in the job
      if: always()
      run: pmg proxy stop --fail-on-violation
```

Invariants held across all integrated jobs (machine-verified):

- The setup step comes **after** toolchain setup (`setup-node`) so the Node download
  is not routed through the proxy.
- The enforce step is **the last step** in every job. `pmg proxy stop` leaves
  `HTTP_PROXY` set, so any network step ordered after it would point at a dead proxy.
- The enforce step carries `if: always()`, so violations surface even when a build
  step fails.

---

## Per-workflow / per-job detail

### `test.yml` — Build and Test (`pull_request`, `push` to main/releases)

#### `build` — ✅ integrated, full coverage
The only job in the repo that genuinely installs packages (`npm ci`). PMG sits after
`actions/setup-node` and `actions/checkout`, before `npm ci`; enforce is last, after
`verify-no-unstaged-changes.sh`. This is the job PMG actually protects.

#### `test` — ✅ integrated on Linux, ⚠️ skipped on macOS/Windows legs
Matrix job across `ubuntu-latest`, `macos-latest`, `windows-latest`.

- **Guarded by `if: runner.os == 'Linux'`.** The `safedep/pmg@v1` composite action
  performs an explicit OS check and **hard-fails on any non-Linux runner**
  (cross-platform support is tracked in safedep/pmg#248). Leaving it unguarded would
  fail 2 of the 3 matrix legs outright. This is the "PMG can break it" exemption.
- The job installs **no packages**, so nothing is left unguarded by that skip — PMG is
  wired in on Linux for uniform coverage only.
- **`NO_PROXY` exempts git/LFS/GitHub hosts.** Ubuntu's `git` links against
  `libcurl3-gnutls`, and GnuTLS ignores `SSL_CERT_FILE`, which is how PMG advertises
  trust for its MITM proxy. A proxied clone can therefore fail certificate
  verification even where npm and curl succeed. This job does ~12 clones plus LFS and
  recursive submodule fetches, so those hosts are exempted. `localhost`/`127.0.0.1`
  are exempted so `pmg proxy stop` can reach its own daemon.
- No package traffic is waived by that exemption — there is none in this job.
- Enforce step: `if: always() && runner.os == 'Linux'`.

#### `test-proxy` — ✅ integrated, best-effort
Runs inside the `alpine/git:latest` container (musl, no bash) behind a squid service.

- The PMG action is a **composite action built from bash steps**, which may not
  install or execute in this image — so the setup step is `continue-on-error: true`.
  PMG is attempted; the job proceeds unchanged whether or not it comes up.
- The job's whole purpose is proving checkout works *through the squid proxy*, so
  squid must remain the effective proxy. PMG exports its own proxy vars via
  `GITHUB_ENV`, and precedence between `GITHUB_ENV` and the job-level `env:` block is
  not documented — so **both spellings** (`HTTPS_PROXY`/`https_proxy`) are pinned back
  to squid explicitly. Correct under either precedence rule.
- Teardown is tolerant: it checks `command -v pmg` before calling
  `pmg proxy stop --fail-on-violation`, since pmg may never have installed.
- No packages are installed here, so there is no violation to enforce.

#### `test-bypass-proxy` — ✅ integrated
Asserts checkout still works when `https_proxy` points at a nonexistent proxy and
`no_proxy` exempts GitHub.

- PMG exports working proxy vars, which would quietly make the "broken" proxy
  reachable and void the assertion — so the deliberately-broken values are pinned back
  in both spellings after PMG setup. Only `localhost`/`127.0.0.1` are added to
  `NO_PROXY`, so `pmg proxy stop` can reach its daemon; this does not weaken the
  GitHub bypass under test.
- Installs no packages; PMG is present for uniform coverage.

---

### `licensed.yml` — Licensed (`push`/`pull_request` on main)

#### `test` (Check licenses) — ✅ integrated, full coverage
Runs `npm ci`, so PMG genuinely guards this job. PMG is placed after checkout and
before `npm ci`. Note the `licensed` binary is fetched via `curl` from GitHub
releases — that is a tarball download, not a package-manager install, and is outside
PMG's interception scope either way. Enforce is last, after `licensed status`.

---

### `pmg-test.yml` — PMG Proxy Test (`workflow_dispatch`, manual only)

Purpose-built verification workflow; not part of PR CI. Run it from the Actions tab
after changing PMG config or rotating the `PMG_*` secrets.

#### `test-pmg-allows-clean-install` — ✅ integrated
Installs `lodash` under the proxy and expects success. Guards against a
false-positive/over-blocking regression.

#### `test-pmg-blocks-malicious-package` — ✅ integrated
Installs `safedep-test-pkg@0.1.3` (a known-flagged package) with
`continue-on-error: true`, then **inverts the assertion**: the job is green only when
`pmg proxy stop --fail-on-violation` exits non-zero. A clean exit means PMG failed to
enforce and the step raises `::error::`.

#### `test-pmg-git-compatibility` — ✅ integrated
Compatibility probe backing the `test` job's `NO_PROXY` decision above. Runs five
`continue-on-error` probes under the proxy — plain `git clone`, checkout basic, LFS,
recursive submodules, and the REST API fallback path — prints the git HTTPS backend
and cert environment, then fails with an explicit verdict if any probe regressed.
This job is how you re-test whether the git exemptions in `test` can be dropped.

---

### `genesis.yml` — Quality Checks (nightly `schedule`)

#### `Analysis` — ❌ NOT integrated (structurally impossible)

```yaml
jobs:
  Analysis:
    uses: razorpay/genesis/.github/workflows/quality-checks.yml@master
    secrets: inherit
```

This is a **reusable-workflow call**, not a `steps:` job. The GitHub Actions workflow
schema forbids `steps:` on a job that has `uses:` — adding the PMG steps here would
make the workflow file invalid and the job would not run at all. There is no
caller-side hook (no pre/post step, no way to inject env into the callee's runner).

**This is the one hard skip, and it is a "PMG would break it" case in the strongest
sense: the file would stop parsing.**

To cover it, PMG must be added inside
`razorpay/genesis/.github/workflows/quality-checks.yml` on the callee side — every job
in that workflow that installs packages. That is a change in the `razorpay/genesis`
repo, out of scope here. Once added there, every repo calling it (including this one)
inherits the coverage.

`permissions:` was deliberately **not** added to `genesis.yml`: the callee uses
`secrets: inherit` and its required token scopes are unknown, so restricting them
from the caller risks breaking the nightly quality checks.

---

## Changes made in this pass

- Added `permissions: contents: read` to `test.yml`, `licensed.yml`, and
  `pmg-test.yml`, completing the documented PMG pattern. All jobs in these workflows
  only read repository contents (checkout, npm install, license check), so this is
  safe.
- All four workflow files re-parsed and the PMG invariants (setup placement, enforce
  step is last, `if: always()`) verified programmatically.

## Secrets required

| Secret | Used by |
|---|---|
| `PMG_PUBLIC_REPOS_TOKEN` | all 8 integrated jobs (`api-key`) |
| `PMG_TENANT_ID` | all 8 integrated jobs (`tenant-id`) |

Both are optional per SafeDep's docs (PMG still blocks locally without them) but are
required to sync events to SafeDep Cloud before the runner is destroyed.

## Known limitations

- **Non-Linux runners are unsupported by the action** — macOS and Windows matrix legs
  cannot carry PMG until safedep/pmg#248 lands.
- **Container jobs are best-effort** — the composite action needs bash; minimal images
  like `alpine/git` may not run it.
- **git traffic vs. the MITM proxy** — GnuTLS-linked `git` does not honour
  `SSL_CERT_FILE`. Where a job does heavy git work and installs nothing, exempting
  GitHub hosts via `NO_PROXY` costs no coverage. Re-check with
  `test-pmg-git-compatibility` before tightening.
- **Reusable-workflow calls cannot be instrumented from the caller** — coverage must
  be added in the called repo.
