# Developing

## Prerequisites

- Rust toolchain from `rust-toolchain.toml`; CLI/daemon MSRV 1.89, tray MSRV 1.90
- Python 3.12 or newer for release tooling and reports; Bun for browser fixtures
- `cargo-cyclonedx` for workspace SBOM generation
- `cargo-nextest` for tests and `cargo-deny` for dependency checks
- `tauri-cli` and the platform's Tauri prerequisites for the tray app: Xcode command-line tools on macOS, WebKitGTK/GTK on Linux, MSVC on Windows
- On Fedora/RHEL, `qt5-qttools` provides `qtpaths`, which the Linux AppImage bundler needs

## Build

From the repository root:

```sh
cargo build --release -p testd   # CLI and daemon
cargo tauri build                # tray app
```

The workspace includes `testd-gui`. On a machine without GTK/WebKit, add `--exclude testd-gui` to workspace test and clippy commands. CI installs the Tauri system prerequisites and checks the full workspace.

## Icons

`src-tauri/icons/` is generated; its sources live in `assets/brand/`. Regenerate the bundle and tray icons with:

```sh
python3 scripts/build-icons.py
```

The script uses only the Python standard library. `testd-app.png` supplies sizes below 128px, `testd-app-large.png` supplies 128px and above, and the two 32px masks supply the tray variants. `testd-app-large.png` is a 1024px render of `testd-slim.svg`; refresh it when the SVG changes.

## Website

The product site lives under `sites/`; the documentation Markdown lives under `docs/` at the repository root. The site runs in a container, so no local `bun install` is needed to develop, build, or check it.

```sh
./sites/bin/site.sh dev      # product at http://127.0.0.1:5173/, docs at /docs/
./sites/bin/site.sh build    # dist/, with the docs under dist/docs
./sites/bin/site.sh preview  # serves dist/ at http://127.0.0.1:4173/
./sites/bin/site.sh check
./sites/bin/site.sh links
./sites/bin/site.sh shell
./sites/bin/site.sh clean
# or
bun run dev
```

The script runs the official Bun image (`oven/bun:1.4.4`) with `docs/`, `sites/`, and `scripts/` mounted, installs dependencies into the `testd-site-node-modules` volume, and runs the stock `vitepress` and `vite` CLI entrypoints with Bun. It prefers Podman and falls back to Docker; set `TESTD_SITE_RUNTIME` to pin one. The product server is the public origin and proxies `/docs/` to VitePress on loopback.

`--host 0.0.0.0` (or `TESTD_SITE_HOST`) serves the site to the network. `--port <port>` overrides the published port for `dev` or `preview`; `TESTD_SITE_PORT` and `TESTD_SITE_PREVIEW_PORT` set the defaults, and `TESTD_SITE_IMAGE` or `TESTD_SITE_VOLUME` overrides the image or dependency volume. Remove the volumes with `./sites/bin/site.sh clean`.

The build and check scripts remain the local path used by CI; they need `bun install` first and run outside the container:

```sh
bun install
bun run build     # dist/, with the docs under dist/docs
bun run preview   # serves the built tree at http://localhost:4173/
bun run check
bun run check:links
```

`bun run build` writes the product site first and the docs second; deploy `dist/` as one static tree. `bun run preview -- --host 0.0.0.0` exposes the built tree on every interface.

`master` publishes `dist/` to <https://testd.dev> through the Cloudflare Pages Git integration. The project settings and the Workers alternative are in [sites/README.md](sites/README.md#publish).

## Test

Use hosted CI for the repeatable platform matrix. Use the matching platform script to reproduce a failure locally. Run local commands from the repository root; GitHub commands below explicitly select `LapilliWorks/testd` and also work from another directory.

| Run | Scope |
|---|---|
| Push to `master` or open a pull request | Change-filtered automated checks; documentation-only changes may skip builds |
| Manual `ci.yml` workflow run | Full platform matrix, minimum-runtime fixtures, website checks, native package builds, and workspace SBOMs; no release publishing |
| Local `ci` suite | The same 20 unprivileged stages on the current platform, including GUI compilation/tests, ignored contracts, doctests, fixtures, audit, website, and CLI/daemon checks |
| Local `all` suite | Automated checks plus separately prepared and explicitly enabled system, artifact, desktop, browser, and manual stages; missing evidence remains blocked |

### Run the full matrix on GitHub

No local Rust, Python, Bun, or VM setup is needed for hosted CI. The workflow provisions its tools and build libraries. With the [GitHub CLI](https://cli.github.com/) installed, check authentication with `gh auth status`; the account must be allowed to dispatch this repository's workflows. A personal access token also needs `workflow` scope when pushing changes to `.github/workflows/`.

Commit and push the revision you want tested before dispatching. `--ref master` uses GitHub's copy of `master`, not local or uncommitted changes; replace `master` with a pushed branch name to test that branch. Do not create a version tag just to run tests: version tags can trigger release publishing.

```sh
gh workflow run ci.yml --repo LapilliWorks/testd --ref master
gh run list --repo LapilliWorks/testd --workflow ci.yml --event workflow_dispatch --limit 5
```

Alternatively, open the repository's **Actions → ci → Run workflow**, select the branch, and start the run. Manual runs include package and SBOM builds only after the test gates pass. Their unsigned artifacts are retained by GitHub Actions; the release-publishing job stays skipped.

Replace `RUN_ID` in the following commands with the numeric ID from the new run. Confirm its source SHA and URL before interpreting results:

```sh
gh run view RUN_ID --repo LapilliWorks/testd --json headSha,status,conclusion,url
gh run watch RUN_ID --repo LapilliWorks/testd --interval 45 --exit-status
gh run view RUN_ID --repo LapilliWorks/testd --web
```

`watch --exit-status` returns nonzero for an unsuccessful run. A transport error or a watch timeout is not itself a CI failure: inspect `run view` and resume watching the same run. The five platform summaries must each contain 20 passing stages. For a full manual run, also require the website, minimum-runtime fixture, four bundle, and SBOM jobs to pass. An intentionally skipped release-publishing job is not missing test evidence.

### Inspect failures and repeat a run

Wait for the whole run to complete before requesting its combined failure log:

```sh
gh run view RUN_ID --repo LapilliWorks/testd --log-failed
gh run rerun RUN_ID --repo LapilliWorks/testd --failed
gh run watch RUN_ID --repo LapilliWorks/testd --interval 45 --exit-status
```

Use `rerun --failed` for a transient infrastructure failure. A rerun uses the original commit and workflow, not newly pushed fixes. After fixing code, commit and push it, dispatch a new run, and use the new run ID. For a complete repeat of the original run, omit `--failed`.

### Retain reports and built artifacts

Use a new, private, persisted destination for each run or rerun attempt. Replace `OUTPUT_DIR` below with that destination; do not overwrite an earlier attempt's evidence. The download commands work in both POSIX shells and PowerShell.

```sh
gh run download RUN_ID --repo LapilliWorks/testd --pattern 'ci-summary-*' --dir 'OUTPUT_DIR/summaries'
gh run download RUN_ID --repo LapilliWorks/testd --pattern 'bundles-*' --dir 'OUTPUT_DIR/bundles'
gh run download RUN_ID --repo LapilliWorks/testd --name sbom --dir 'OUTPUT_DIR/sbom'
gh run view RUN_ID --repo LapilliWorks/testd --log > 'OUTPUT_DIR/run.log'
```

Each `ci-summary-*` artifact contains `ci-summary.json`, with source/OS/Python metadata and stage statuses. Package and SBOM artifacts are available from full manual or tag runs, not ordinary push/PR runs. Full runner reports, fixture homes, PKI, browser profiles, and source patches are not uploaded. GitHub retains command output in the job logs; review logs before sharing because fixture output can contain sensitive data. Built artifacts are unsigned and are not installation or signing evidence.

### Reproduce the unprivileged suite locally

Install the [prerequisites](#prerequisites), including the native Tauri build libraries/compiler even if you only want to run `ci`. Python 3.12 or newer is supported; hosted CI currently uses 3.14, while its minimum-runtime fixtures use 3.12. Ensure the rustup shims and Bun are on `PATH`; a distro-provided `cargo` cannot interpret `+stable`. On Linux/macOS, load `source "$HOME/.cargo/env"` in the current shell if needed; on Windows, reopen PowerShell after installation. Prepare the shared tools and checkout dependencies once:

```sh
rustup toolchain install stable 1.89.0 1.90.0 --profile minimal
cargo +stable --version
rustup component add --toolchain stable rustfmt clippy
cargo +stable install --locked cargo-nextest cargo-deny
bun install --frozen-lockfile
```

The runner installs nothing. Run it from a normal user terminal with no existing testd daemon occupying the fixture ports. Do not initialize testd or install system integration first: the host checks create their own isolated home. Unset `CARGO_BUILD_TARGET` for automatic native builds, or pass `--bin` with a separately verified candidate CLI. When working in a Git worktree, point `CARGO_TARGET_DIR` at the main checkout's `target/` to share the build cache.

On Linux, including Fedora, use:

```sh
export CARGO_INCREMENTAL=0 CARGO_PROFILE_DEV_DEBUG=0 CARGO_PROFILE_TEST_DEBUG=0
python3 scripts/test-linux.py ci --plan
python3 scripts/test-linux.py ci --report "$HOME/testd-reports/linux-ci-$(date -u +%Y%m%dT%H%M%SZ)"
```

On macOS, use the same environment settings and timestamped report pattern with `scripts/test-macos.py`:

```sh
export CARGO_INCREMENTAL=0 CARGO_PROFILE_DEV_DEBUG=0 CARGO_PROFILE_TEST_DEBUG=0
python3 scripts/test-macos.py ci --plan
python3 scripts/test-macos.py ci --report "$HOME/testd-reports/macos-ci-$(date -u +%Y%m%dT%H%M%SZ)"
```

On Windows, use PowerShell, not WSL; no elevation is needed for `ci`:

```powershell
$env:CARGO_INCREMENTAL = '0'
$env:CARGO_PROFILE_DEV_DEBUG = '0'
$env:CARGO_PROFILE_TEST_DEBUG = '0'
$stamp = (Get-Date).ToUniversalTime().ToString('yyyyMMddTHHmmssZ')
$report = Join-Path $HOME "testd-reports\windows-ci-$stamp"
python scripts\test-windows.py ci --plan
python scripts\test-windows.py ci --report $report
```

The report path must not exist and must be outside temporary storage. To repeat a run, generate a new timestamp rather than deleting the previous report. Windows also requires a free UDP port 53. Other occupied fixture ports are reported by the runner; do not stop unrelated services merely to make a test pass.

### Read local outcomes and remaining coverage

Inspect `report.json`, `run.log`, `boundaries.log`, and individual stage reports/logs. The `ci` suite additionally writes the publishable status-only `ci-summary.json`.

| Exit code | Meaning |
|---|---|
| `0` | Applicable checks completed without failures |
| `1` | At least one failure |
| `2` | Incomplete coverage without failures |

Missing prerequisites and unexpected skips are not passes. The `ci` suite excludes actual tray/login observations and all privileged, installer, real-browser, and manual stages; `all` retains missing release evidence as blocked. Therefore, an unprepared `all` run may exit `2` even when its automated checks pass.

A Fedora container run establishes userspace behavior on the Ubuntu runner's kernel, not native Fedora system integration. Hosted Windows Server is not desktop Windows 11 evidence. Resolver/trust changes, service supervision, real browser trust, desktop setup, installation/uninstall, and signing/notarization remain separate checks. For those operations use the [release-validation commands](#release-validation) on a clean disposable host with a persisted full-system backup, retain reports outside the guest before recovery, and restore the baseline between sensitive stages. macOS and Windows product system/desktop checks remain blocked; do not bypass the guards or append `--system` to `ci`.

See [tests/README.md](tests/README.md) for individual suite commands, detailed contracts, browser preparation, and recovery requirements, and [CI coverage](#ci-coverage) for the hosted platform matrix. A green CI run is not release approval. Runner images, stable Rust, and installed tool versions can change between runs even when the source commit and lockfiles are unchanged.

### Targeted checks

For quick individual checks without a runner report:

```sh
cargo nextest run --workspace
cargo clippy --workspace --all-targets -- -D warnings
cargo fmt --check
cargo deny check
```

Ordinary nextest runs skip ignored contracts; use the `ci` or `contracts` suite for the ignored release gate. Exclude `testd-gui` from workspace commands if the native GUI build prerequisites are absent.

## Release validation

This is the runbook for checks **outside hosted CI**: actual OS resolver/trust changes, service supervision, real browsers, interactive desktop/login behavior, and installation/uninstall. Use individual suites on separately restored VMs, not one unattended `all --system --interactive` run. Incomplete artifact or system cleanup can block later stages of `all`.

### What can currently run

| Check | Linux desktop VM | macOS VM | Windows 11 VM |
|---|---|---|---|
| Resolver/trust installation and owned cleanup | `host --system`; shared trust effects still require independent comparison | Blocked by ownership safeguards | Blocked by ownership safeguards |
| Actual service supervision | `service` with a real systemd user session | Launchd check blocked by ownership safeguards | Not applicable: no CLI service shipped |
| Real browser | Fixture-owned setup with `browser --system` | Preconfigured mode only; separately proven DNS/trust setup required | Preconfigured mode only; separately proven DNS/trust setup required |
| Tray, setup, login, GUI workflow | `desktop`, plus separate logout/login evidence | Desktop runner blocked | Desktop runner blocked |
| Archive/package smoke | `artifacts`; real uninstall remains incomplete | `artifacts`; not complete installed-app evidence | `artifacts`; not complete installed-app evidence |
| Full installation/upgrade/uninstall | Manual checklist with owned integration and full-system comparison | Resolver/trust cleanup coverage blocked | Resolver/trust cleanup coverage blocked |
| Signing/notarization | Record packages as unsigned | Separate credentialed signing and verification | Separate credentialed signing and verification |

A VM does not bypass a blocked runner. Do not remove its guards, force system checks through direct scripts, or mark blocked platform coverage passed. macOS 15 or another explicitly claimed supported macOS version can be tested; upgrading the host to Tahoe is not a requirement of this runbook.

### Prepare and save the VM baseline

1. Create a disposable VM with a real desktop and a fresh local test user. Use native Ubuntu 24.04 and Fedora 44 Workstation VMs for claimed Linux coverage, the claimed macOS versions/architectures on Apple hardware, and Windows 11 for desktop Windows evidence. Containers, WSL, and Windows Server do not substitute for these OS checks.
2. Before changing the guest, persist a complete restorable backup of its disks and VM configuration outside the guest and outside temporary storage. Check that your VM manager can restore it. Do not rely solely on an in-guest snapshot or `system-before/` report.
3. Install Python 3.12 or newer, Git, and the suite-specific tools below. Install Bun for browser fixtures and the selected real browsers with clean profiles and no accounts. Install native GUI runtime dependencies for the candidate package; source builds additionally need the [build prerequisites](#prerequisites). Use only synthetic projects and credentials.
4. Copy the exact candidate assets and a checkout of their source revision into the VM. Record the run URL, source SHA, OS image/version/architecture, tool/browser versions, and artifact SHA-256 digests. Do not execute candidate binaries on your everyday host merely to check their version.
5. Save a second full-system backup as the **prepared baseline**, still without testd user data, resolver/trust integration, running daemon, or login entry. Preserve installed tools/browsers in this baseline so repeated runs use the same environment.
6. Before each sensitive suite, restore that prepared baseline, verify the guest is clean, and choose a new persisted report path. Block external network egress for browser tests after installing dependencies; keep loopback working. Disable unnecessary shared host folders and real providers. Use a dedicated private evidence-transfer location only.
7. After each suite, stop candidate apps, daemons, browser fixtures, and providers. Retain the report and before/after evidence outside the VM before restoring it. Restore the baseline before the next suite, browser, package, upgrade, or seeded-conflict case. A report left only on a reverted guest is lost evidence.

On Linux, the resolver and service suites need a running systemd-resolved and systemd user manager. Provision tools **only inside the backed-up disposable guest**:

```sh
# Ubuntu 24.04
sudo apt-get update
sudo apt-get install -y python3 git curl openssl systemd-resolved ca-certificates libnss3-tools libcap2-bin

# Fedora 44: use this instead of the Ubuntu commands
sudo dnf install -y python3 git curl openssl systemd-resolved ca-certificates nss-tools libcap
```

From a terminal in the guest's logged-in desktop session, check the prerequisites without changing them:

```sh
python3 --version
systemctl is-active systemd-resolved
resolvectl status
systemctl --user show-environment
```

If the resolver or user manager is unavailable, prepare a supported clean guest first. On Fedora, record `getenforce` and retain SELinux enforcement; do not switch to permissive to obtain a pass. Do not stop unrelated listeners or overwrite conflicting DNS configuration to obtain a pass. Run suite commands directly in an interactive terminal as the test user, not under `sudo python`, through a pipe, or with redirected stdin. Cache sudo only where shown below. On Windows use `python` or `py -3` instead of `python3`.

### Prepare the candidate and per-run paths

Download only the matching `bundles-TARGET` artifact from the successful manual CI run. Targets are `x86_64-unknown-linux-gnu`, `aarch64-apple-darwin`, `x86_64-apple-darwin`, and `x86_64-pc-windows-msvc`. Keep each target's files in its own candidate directory. Manual builds do not provide the release job's `SHA256SUMS`; generate a local manifest from the downloaded bytes before transferring them:

```sh
python3 scripts/release.py checksums /absolute/path/to/new-candidate-directory
```

This manifest detects later changes; it is not a published release checksum, signature, or proof of source provenance. Preserve it with the run metadata. Do not regenerate an existing manifest to conceal changed files. For published releases, use their original manifest instead.

Inside the VM, extract the CLI archive into a dedicated candidate directory. Set `CLI` to that executable and, for desktop testing, `APP` to the actual executable from the selected package or verified source build. The app and CLI must match the intended source revision. These POSIX-shell variables are recreated after each baseline restore:

```sh
CLI='/absolute/path/to/candidate/testd'
APP='/absolute/path/to/candidate/testd-gui'
ASSETS='/absolute/path/to/candidate-assets'
REPORT="$HOME/testd-reports/$(date -u +%Y%m%dT%H%M%SZ)"
```

Replace the paths before use. GitHub artifact downloads do not preserve Unix executable permissions: inside the backed-up Linux guest, use `chmod u+x /absolute/path/to/candidate.AppImage` before its runtime smoke check if needed. Preserve the downloaded original; record permission changes separately from content hashes. The runner creates the report directory; do not create it yourself or reuse a previous path. For multiple commands on the same restored guest, choose distinct report suffixes. Candidate hashes, command output, OS metadata, and source revision are retained by the runner.

### Run Linux resolver and trust checks

Restore the prepared baseline, set the candidate/report variables, then run:

```sh
sudo -v
python3 tests/release/run.py host --system --bin "$CLI" --report "$REPORT-system"
```

Do not run `testd init` first. This suite creates an isolated fixture home, installs its test-only route and public root, tests OS resolution and trusted HTTPS, stops its fixture, and invokes ownership-checked cleanup. It refuses existing testd integration. It can change the systemd-resolved drop-in, the distro CA anchor, and generated shared trust-store output/hooks.

Inspect `report.json`, command logs, and `system-before/`. Independently compare resolver state, source anchors, effective trust, generated bundles, and hook effects with the persisted full-system baseline. Scoped cleanup alone leaves this stage incomplete. Record the comparison result separately; neither restoring the VM nor editing report JSON converts missing preservation evidence into a pass.

Complete [SYS-01 through SYS-16](tests/manual/system-integration.md), including interruption/recovery and synthetic unowned/conflicting-state cases. Each seeded conflict needs its own restored snapshot and before/after identity/content evidence; do not ask `host --system` to overwrite it. SYS-15 and SYS-16 remain blocked until macOS/Windows ownership safeguards exist.

### Run Linux service supervision

Restore the prepared baseline and run from the test user's real desktop session:

```sh
systemctl --user show-environment
python3 tests/release/run.py service --bin "$CLI" --report "$REPORT-service"
```

Do not add `--system` or run this as root. The suite uses a randomly named user unit, isolated home, fake provider, high ports, and actual manager-directed start, crash/restart, and stop. Verify the stopped listener and ownership-checked unit cleanup in the report; `service-unit.service` retains the generated unit. Keep macOS launchd marked blocked and Windows CLI service marked not applicable.

### Run real Linux browsers

Before saving the prepared baseline, install Playwright where Bun can resolve it from the checkout, and install the engines you intend to test. Do not add a project dependency or modify tracked manifests just for a VM run:

```sh
bun install --frozen-lockfile
bun add --no-save playwright
bunx playwright --version
# Ubuntu: installs browser dependencies as well as the selected engines
bunx playwright install --with-deps chromium firefox webkit
```

On Fedora, use `bunx playwright install chromium firefox webkit` and supply the engine's Fedora runtime dependencies separately; do not force Ubuntu's dependency installer onto Fedora. Unsupported or missing engines remain blocked. Linux Chromium/Firefox require NSS `certutil`, provided by the packages above. Record Playwright/browser versions and retain them in the prepared baseline.

For an automatic real-engine run, restore the baseline, block guest external egress, then run:

```sh
sudo -v
python3 tests/release/run.py browser --system --bin "$CLI" --browser-engine chromium \
  --edge-port 8443 --report "$REPORT-chromium"
```

Repeat from separately restored baselines for `firefox` and `webkit`. No prior `testd init` or daemon is required: `--system` creates the fixture home, configures its candidate resolver/root, starts its own host, drives the browser, checks OS TLS before/after a renewed leaf, and performs scoped cleanup. Shared trust effects still require the independent comparison described above. Playwright WebKit is not installed Safari evidence; Chromium is not installed Chrome or Edge evidence.

For the actual installed Chrome, Firefox, or Edge and their certificate viewers, run a separate manual browser session:

```sh
sudo -v
python3 tests/release/run.py browser --system --bin "$CLI" --edge-port 8443 --report "$REPORT-browser-manual"
```

Open the printed `https://app.releasecheck.test:8443/` URL in a clean profile. If that browser uses its own trust store, import only the fixture's public root from `<report>/browser-home/.config/testd/pki/root.crt` into that disposable profile. Do not use certificate-warning bypasses, `--insecure`, browser-only DNS overrides, or a real account. Complete [BROWSER-01 through BROWSER-08](tests/manual/browser.md): certificate/SAN/chain/lifetime, WebSocket idle, SSE/upload, cookie scope, dashboard/dead upstream/unknown name, and renewal. Save the page's JSON report, stop with Ctrl+C, close the test browsers, and answer shutdown confirmations truthfully.

On macOS/Windows, `browser` without `--system` supports a separately preconfigured candidate DNS/trust setup, but does not establish that setup's ownership or cleanup. The host must be stopped before running the suite; it starts its own candidate daemon. Do not improvise a fresh-VM root/NRPT/resolver installation to bypass the current platform blockers. Record Safari/Windows browser coverage as blocked until a separately proven, restorable setup is available.

### Run Linux desktop, login, and installed-package checks

Restore the baseline and install the actual selected `.deb` or prepare the verified AppImage/source-build executable and required GUI runtime libraries. Do not launch or initialize testd before the desktop runner: the test user must still have no testd configuration. On Ubuntu, install the verified package and inspect its paths:

```sh
DEB='/absolute/path/to/candidate.deb'
sudo apt-get install "$DEB"
dpkg-deb --contents "$DEB"
```

Set `APP` to the installed GUI executable shown by the package, not the `.deb` file. Fedora does not install a Debian package; use its verified AppImage or a native source build and record the actual packaging format tested.

The real GUI executable must be able to bind port 443. On the backed-up guest only, grant the capability to the actual ELF executable if needed and record the change:

```sh
sudo setcap 'cap_net_bind_service=+ep' "$APP"
getcap "$APP"
python3 tests/release/run.py desktop --app "$APP" --bin "$CLI" --report "$REPORT-desktop"
```

Do not grant a capability to an AppImage launcher and assume its extracted child inherits it; identify the actual GUI executable. Do not run the entire desktop as root. The runner launches the app and prompts for [GUI-01 through GUI-28](tests/host/gui-checklist.md). Perform each action before answering `p`, `f`, `n`, or `q`; record evidence/reasons and never include the operator secret. Only confirm shutdown after all app/daemon/provider processes and fixture listeners have stopped. Deferred cleanup requires full-system recovery.

GUI-24 needs a **separate logout/login run**: retain evidence outside the guest, enable the candidate's login item, log out/in, and verify one actual tray starts with the expected state. Record the login artifact and process observations; an existing plist/desktop/registry entry alone is not proof. Do not expect an interactive runner session to survive logout.

For archive smoke on each supported OS/architecture, on a separately restored VM:

```sh
python3 tests/release/run.py artifacts --artifacts "$ASSETS" --require-checksums --report "$REPORT-artifacts"
```

Windows equivalent, with a new report path on every attempt:

```powershell
python tests\release\run.py artifacts --artifacts 'C:\candidate-assets' --require-checksums `
  --report "${HOME}\testd-reports\artifacts-$((Get-Date).ToUniversalTime().ToString('yyyyMMddTHHmmssZ'))"
```

Artifact smoke validates checksums, CLI extraction/version/init/re-init, and applicable package metadata/runtime; it deliberately reports real uninstall/purge as blocked. Expected exit `2` is **incomplete**, not a pass for the installed workflow. `--previous-bin /path/to/real-previous-testd` adds CLI configuration/root preservation checks, not actual package upgrade evidence.

Complete [INSTALL-01 through INSTALL-10](tests/manual/installation.md) separately for each claimed package: actual installation and first setup, trusted browser workflow, stopped registration, upgrade from a real prior release, interrupted-upgrade rollback, disabled login item, owned integration removal, package removal, and purge/reinstall. On Linux, stop candidate processes and retain ownership/root evidence before `testd uninstall`; purge only after system cleanup succeeds. Compare unowned and shared state against the full-system baseline. Do not manually remove conflicting state. On macOS/Windows, complete cleanup/trusted first-run evidence remains blocked; archive smoke does not resolve it.

### Verify signing and conditional network claims

Manual CI artifacts are unsigned. Record that fact rather than claiming signing/notarization passed. If signed desktop delivery is claimed, follow [Signing and notarization](#signing-and-notarization) and verify the final signed bytes on the target OS. Do not publish a release merely to test signing.

On macOS, verify the actual signed app and its notarization ticket:

```sh
codesign --verify --deep --strict --verbose=2 /path/to/Testd.app
codesign --display --verbose=4 /path/to/Testd.app
spctl --assess --type execute -vv /path/to/Testd.app
xcrun stapler validate /path/to/Testd.app
```

On Windows, inspect the installer and installed executables:

```powershell
Get-AuthenticodeSignature 'C:\candidate\testd-setup.exe' | Format-List Status,StatusMessage,SignerCertificate
```

Require the expected signer, not merely any valid or ad-hoc signature. SmartScreen reputation and actual OS prompts need separate observations. Missing signing credentials leave signed-delivery claims unverified, not passed.

If a release claims real-provider sharing or non-loopback network behavior, use the separately authorized two-machine procedure in [network and sharing contracts](tests/manual/network-and-sharing.md). Fake providers do not establish real exposure or crash cleanup; unimplemented network controls remain blocked. Do not enable external exposure or a real provider without explicit approval.

### Retain evidence and recover before repeating

Keep `report.json`, `run.log`, command logs, scoped snapshots, and browser/desktop evidence private. Outside the guest, retain a ledger with one row per SYS/BROWSER/GUI/INSTALL/NET/SHARE check: candidate SHA/digest, OS/browser/package, full-system baseline ID/location, result (`pass`, `fail`, `blocked`, or justified `not applicable`), evidence paths, and independent cleanup comparison. Untested, interrupted, and unanswered checks never become passes.

If interrupted, stop every candidate/fixture process and inspect the retained report. An interactive scoped recovery may be attempted on the same guest:

```sh
python3 tests/release/run.py --restore-report /path/to/retained-report
```

This is not a full-system restore. Conflicts stay untouched, and Linux/macOS shared trust effects may still produce exit `2`. Preserve recovery logs and restore the full prepared VM baseline before any further sensitive test. Copy evidence out **before** reverting. A restored guest proves recoverability, not that the candidate's uninstall preserved all unowned state.

## Contents

- [Quickstart](docs/quickstart.md) for using the installed tool
- [Test runbook](#test) for repeated CI runs, local reproduction, and reports
- [Non-CI validation runbook](#release-validation) for backed-up VMs, browsers, services, desktop, and installers
- [Host testing](#host-testing) for checks that need real machines
- [Browser validation](#browser-validation) for manual checks in real browsers
- [Release](#release) for the release checklist and artifacts
- [Homebrew](#homebrew) for the formula and cask
- [Signing and notarization](#signing-and-notarization) for macOS notarization and Windows Authenticode

## Host testing

The container suite covers the protocol, the edge, registration ownership, DNS answers, CLI behavior, sharing, and the standalone operator secret. The checks below cover OS resolver integration, trust stores, privileged ports, packaging, code signing, and the tray.

Run them on real machines, not inside the dev container. A macOS, Linux, and Windows host are needed for full coverage. Use a dedicated test machine or a VM: the optional system checks change the OS resolver and trust store.

### What the scripts cover

| Script | Checks |
|---|---|
| `tests/host/linux.sh` | resolver route, CA trust, edge TLS, loopback dashboard, DNS A records, token requests, issued tokens, local registration, sharing, quick launches, URL actions, doctor state checks, launch-at-login entry, systemd unit |
| `tests/host/macos.sh` | Unprivileged edge, dashboard, DNS, registration, sharing, quick launches, doctor state, and launch-at-login checks; resolver and trust checks are blocked |
| `tests/host/windows.ps1` | Unprivileged edge, dashboard, DNS, registration, sharing, quick launches, and launch-at-login checks; NRPT and root trust checks are blocked |
| `tests/host/gui-checklist.md` | setup dialog, tray, dashboard window, click-to-open, single instance, quick launches, certificate controls, URL actions, packaging/signing |

The Linux and macOS scripts share their core checks in `tests/host/common.sh` and their helpers in `tests/host/lib.sh`; the platform scripts add the resolver, trust, and init-system sections.

The sharing checks start the daemon with `TESTD_TUNNEL` pointing at a fake provider, so they need no provider installed: start, list, refuse a second share, stop, refuse an unknown provider, and refuse a name that is no longer registered. The provider shapes themselves (cloudflared, ngrok, tailscale) are covered by `cargo test -p testd-daemon --test share`, which the Windows script cannot do because a PowerShell script is not a spawnable provider binary.

Local registration uses the operator secret that `testd init` writes into `[host] secret`; the standalone host has no accounts or roster. The scripts run the client on the node itself, where the default loopback profile uses that secret automatically; no `TESTD_TOKEN` is exported. A separate named profile with no credential requests a token, which the script approves and then revokes through `testd requests`, `testd approve`, and `testd tokens`.

The quick launch checks need `python3` (or `python` on Windows) to start a local HTTP server. They save a launch through the loopback dashboard, start it, check the name through the edge and DNS, stop it, and delete it.

Each script uses:

- a throwaway `HOME`/`%APPDATA%` retained under the persisted report directory,
- a test-only throwaway TLD (`testdcheck` by default) so it does not collide with a real `.test` setup,
- a high HTTPS port (`8443`) so it does not need root for the edge,
- dedicated upstream ports for registration (`18080`) and quick launches (`18081`).

The scripts do **not** call sudo themselves. On Linux and macOS they use `--cacert` for HTTPS; on Windows they use `--insecure` for edge and proxy checks because Schannel does not treat `--cacert` like OpenSSL. Only the Linux runner currently permits optional OS resolver and trust checks, with a cached sudo credential, a disposable-host confirmation, and a persisted full-system snapshot. macOS and Windows system checks are blocked.

### Linux

Prerequisites: `bash`, `curl`, `python3`, `resolvectl` (for the optional system
check), `systemd-analyze` (optional). `sudo` must be available if
`TESTD_HOST_TEST_SYSTEM=1`.

```sh
cd testd
cargo build --release -p testd

# Core checks, no system changes
tests/host/linux.sh

# Optional system checks on a backed-up disposable VM, with a new persisted report
sudo -v
python3 tests/release/run.py host --system --bin target/release/testd \
  --report ~/testd-reports/linux-system-01
```

The Linux system check can write `/etc/systemd/resolved.conf.d/90-testd.conf` and the distro's CA anchor: `/usr/local/share/ca-certificates/testd-root.crt` on Debian and Ubuntu or `/etc/pki/ca-trust/source/anchors/testd-root.crt` on Fedora and RHEL. Cleanup uses the candidate's ownership checks and reports conflicts without replacing or deleting unowned state. Trust-store regeneration and hooks can affect additional shared files, which the scoped snapshot does not verify.

### macOS

Prerequisites: `bash`, `curl`, `python3`, `plutil`, and `dscacheutil`. Xcode command-line tools are needed to build.

```sh
cd testd
cargo build --release -p testd
tests/host/macos.sh
```

The unprivileged host checks do not install a resolver or trust root. The product system checks, desktop runner, launchd service check, and resolver-specific probe are blocked until their ownership and cleanup can be established. The scoped self-test removes only a recorded exact fixture certificate and cannot verify all keychain trust settings or shared effects.

### Windows

Prerequisites: PowerShell 5.1 or later, `curl.exe` (included with Windows 10
1803+), and a free port 53. The daemon uses port 53 for NRPT; if another
service owns it, the script fails at the DNS check.

The unprivileged script prints the OS caption, version, build, and architecture at the start of every run. Compilation, PowerShell fixtures, and CLI checks do not validate NRPT, machine trust, or a desktop Windows release.

CI's `windows-latest` runner is Windows Server, not a desktop Windows 11 host.
It is useful for compilation and the CLI tests, but it does not replace the
NRPT and trust-store checks above.

```powershell
cd testd
cargo build --release -p testd

# Core checks: edge and proxy without system trust
powershell -NoProfile -ExecutionPolicy Bypass -File tests\host\windows.ps1
```

The unprivileged run intentionally uses `curl --insecure --ssl-no-revoke` for edge and proxy checks. That validates routing and TLS minting without requiring a trust-store change. The loopback dashboard is proxied through the edge and returns 200; these checks do not establish OS trust.

`testd init` and `testd doctor --fix` no longer need the whole process elevated
on Windows. Privileged actions are delegated to
`scripts/testd-elevated.ps1`, a closed-action helper that validates the TLD,
IPv4 address, certificate thumbprint, and CA constraints. If testd is not
elevated, Windows shows one UAC prompt for that helper only.

The Windows snapshot helper verifies NRPT rules and machine roots without deleting or recreating them by name. Real product system and desktop checks remain blocked because the helper cannot prove ownership of every change; use the persisted full-system snapshot for recovery. Windows NRPT rules cannot specify a port, so the daemon binds `:53`. If it is occupied, use a different disposable host rather than stopping an unrelated service.

### Launch at login

The tray enables launch at login once, on its first start, through
`tauri-plugin-autostart`, and then exposes a **Launch at login** checkbox. The
checkbox asks the plugin for the current state after every click, so a failed
enable or disable corrects itself on the rebuild rather than drifting from the
OS.

| OS | Artifact | Check |
|---|---|---|
| macOS | `~/Library/LaunchAgents/dev.testd.host.plist` | `plutil -lint`, and `launchctl print gui/$(id -u)/dev.testd.host` |
| Linux | `~/.config/autostart/dev.testd.host.desktop` | has an `Exec=` line pointing at the tray binary |
| Windows | `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`, value `dev.testd.host` | Read the entry without treating it as candidate evidence |

The CLI-only host scripts cannot establish that an existing login entry belongs to the candidate GUI. Windows records an existing entry as unverified, not as a pass or failure. The GUI checklist covers the actual run and toggle.

### Browser validation

The [Browser validation](#browser-validation) section holds the per-browser script: trust stores, padlock
and certificate expectations, WebSocket and SSE behaviour, cookie scope across
`app.example.test` and `api.example.test`. The host
scripts stop at the HTTP layer on purpose; they cannot judge a padlock.

### GUI

The tray is a public source build and an optional free release artifact. On a backed-up disposable Linux desktop, run `python3 tests/release/run.py desktop --app <candidate> --bin <candidate-cli> --report <new-report>` to record `tests/host/gui-checklist.md`. The app and CLI must come from the intended source. macOS and Windows desktop runs remain blocked until system ownership is proven. Attach/detach of existing processes is deferred and does not gate this release.

### Packaging and signing

These require credentials and hardware that do not belong in this repository:

- **macOS:** Developer ID certificate, notarization credentials, `codesign`,
  `spctl`, `xcrun stapler`.
- **Windows:** Authenticode certificate and SmartScreen reputation; verify with
  `Get-AuthenticodeSignature`.
- **Linux:** unsigned `.deb`, AppImage, and tarball; verify with `dpkg-deb -I`
  and an AppImage smoke run.

See [Release](#release) for the release checklist and `tests/host/gui-checklist.md`
for the per-artifact commands.

### Environment variables

| Variable | Default | Meaning |
|---|---|---|
| `TESTD_BIN` | `target/release/testd` | Binary to test |
| `TESTD_HOST_TLD` | `testdcheck` | Test-only throwaway TLD; must be 1 to 10 lowercase alphanumeric chars |
| `TESTD_HOST_EDGE_PORT` | `8443` | HTTPS edge port |
| `TESTD_HOST_APP_PORT` | `18080` | Test upstream port |
| `TESTD_HOST_LAUNCH_PORT` | `18081` | Quick-launch upstream port |
| `TESTD_HOST_TEST_SYSTEM` | `0` | Allow resolver and trust changes |
| `TESTD_HOST_SNAPSHOT` | unset | Snapshot directory written by `run.py --system` |

### System integration

Before permitted system checks, `run.py` records selected integration under `<report>/system-before/`. The scoped snapshot is not a full-system backup and does not capture every artifact, trust setting, generated store, or hook affected by the product.

| Platform | Selected snapshot | Cleanup policy |
|---|---|---|
| Linux | The systemd-resolved drop-in, the distro's CA anchor, and their ownership markers | Candidate-owned integration is removed through the candidate's ownership checks; other changes are left untouched |
| macOS | The test TLD resolver file and named testd roots in the System keychain | Product checks are blocked; the self-test removes only its exact recorded fixture certificate |
| Windows | NRPT rules for the test TLD and named testd roots in the machine store | Verify only; never remove or recreate rules or roots by name |

File fixtures use exclusive creation and record their identity and content before cleanup. A replaced, changed, deleted, or unowned artifact is reported, not overwritten or removed. Pre-existing keychain and machine-root trust settings are not captured and cannot be verified from this snapshot. System stages remain incomplete when shared trust-store output or hooks have not been compared with the persisted full-system snapshot.

After an interrupted run, stop all candidate processes and inspect the persisted report and full-system snapshot. An interactive scoped recovery can be attempted with the relevant report or `self-test-state` directory:

```sh
python3 tests/release/run.py --restore-report ~/testd-reports/linux-system-01
```

Recovery requires the same platform and any necessary sudo or elevation. It leaves conflicts untouched. Exit `2` means scoped state matched but shared trust effects remain unverified; restore or compare the full-system snapshot before reusing the machine. Direct host scripts cannot run privileged checks without the runner's safeguards.

### CI coverage

`.github/workflows/ci.yml` provisions and runs the `ci` suite on Ubuntu 24.04, macOS 15 Apple Silicon and Intel, and Windows Server 2025. It installs Python 3.14, stable and both MSRV Rust toolchains, `cargo-nextest`, `cargo-deny`, Bun, and website dependencies; Linux build libraries come from the local dependency action. Xcode and MSVC/Windows SDK prerequisites come from the hosted runner images. Rust build caches are isolated by platform, and only master runs save them.

A separate Fedora 44 container job installs Fedora build dependencies, uses its system Python, and runs the same suite as a normal user. It establishes Fedora userspace compilation and CLI/daemon behavior on the Ubuntu runner's kernel, not native Fedora resolver, trust, systemd-session, SELinux, or desktop integration. Hosted Windows Server is not desktop Windows 11 evidence.

Every applicable job runs ignored contracts and executable doctests, rather than reserving them for stable tags. Missing prerequisites, unexpected skips, and incomplete checks make the CI command nonzero. The workflow uploads only `ci-summary.json`; fixture state and full report files are not uploaded. Separate Python 3.12 system-state fixture jobs retain minimum-runtime coverage without changing the runner's resolver or trust store. Site-only changes retain their lightweight website job.

Real system, service, browser-trust, installer, and desktop checks still require an explicitly confirmed disposable host with a persisted full-system snapshot; hosted CI does not supply that snapshot and does not run them. Tags and manual workflow dispatches build and package artifacts and generate SBOMs after the platform, minimum-runtime fixture, and website gates pass, without executing the interactive artifact smoke runner. Manual dispatches use the workspace version and do not publish a release; publishing remains restricted to version tags. Source builds and package creation do not replace installation, signing, or uninstall evidence. A green workflow is not release approval. Required-check settings must use the current platform job names rather than the removed standalone MSRV/audit jobs.

### System checks need a running daemon

The Linux resolver and trust checks query the fixture daemon. The runner verifies that its fixture ports are free before cleanup and refuses to stop or replace unrelated listeners. A missing daemon or unresolved cleanup is reported rather than counted as a pass.

### Interpreting failures

- `init` fails: check the printed log. The script uses a temporary home, so a
  pre-existing lock in your real home is not involved.
- `DNS listener` fails: another process owns the port. On Windows this is
  usually `:53`; on Linux/macOS it is `:7753`.
- `HTTPS listener` fails: the configured `edge_addr` is in use, or the binary
  cannot bind it.
- Dashboard TLS fails but `curl --insecure` works: the root is not trusted. On
  Windows, Schannel reports `CERT_TRUST_REVOCATION_STATUS_UNKNOWN` for a private
  root and does not honor `--cacert` the way OpenSSL does; use
  `--ssl-no-revoke` and either `--insecure` before trust or the machine root
  store after `testd doctor --fix`.
- System resolver check fails after `doctor --fix`: inspect
  `resolvectl status` (the route is a global `DNS=`/`Domains=` pair) or
  `/etc/resolver/testdcheck` (macOS), and make sure the daemon is still running.
  On Linux, `doctor --fix` refuses to write the drop-in when systemd-resolved is
  inactive or already has global `DNS=`/`Domains=` configured, and names the
  setting it would have replaced.

## Browser validation

Manual checks that a name works in a real browser. Use a disposable host with a persisted full-system snapshot. A separate device or non-loopback exposure needs separate approval and synthetic data.

### Preconditions

For the fixture-driven browser suite, follow [Run real Linux browsers](#run-real-linux-browsers); do not start a daemon beforehand. The following applies instead to a separately observed installed-app workflow on a backed-up disposable host. macOS/Windows product-managed DNS/trust setup remains blocked.

- Initialize the intended candidate, establish owned Linux DNS/trust integration, and start its host; `testd doctor` must exit `0`
- Start a synthetic local service and register `testd run app.example:3000 -- <your dev server>`
- Trust only the candidate's public root using the proven OS setup or a clean disposable browser profile; record its fingerprint and the import method

### Trust stores

| Browser | Store | Import |
|---|---|---|
| Chrome, Edge (Linux) | Browser certificate manager or platform integration; varies by package | Import `root.crt` as a trusted authority in the browser certificate manager and verify the actual package's trust behavior |
| Chrome, Edge (macOS), Safari | system keychain | `sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain root.crt` |
| Chrome, Edge (Windows) | machine store | `certutil -addstore -f Root root.crt` |
| Firefox (all) | Profile NSS store, or system roots when enabled | Settings → Privacy & Security → Certificates → View Certificates → Authorities → Import; trust the root for websites |

For NSS command-line imports, use `certutil -d sql:<actual browser profile directory> -A -t 'C,,' -n testd-root -i root.crt`. Do not assume `~/.pki/nssdb` is the Firefox profile. Firefox's `security.enterprise_roots.enabled` can use supported OS root stores; validate it on the actual platform/package rather than assuming system installation is sufficient.

### Checks

Run each in every browser you support: Chrome, Firefox, Safari, Edge.

1. `https://app.example.test` loads with no certificate interstitial or bypass. The leaf is issued by `testd intermediate`, chains to `testd root`, carries `app.example.test` in its SAN, and expires within 24 hours.
2. Wait for the leaf to be re-minted, or restart the daemon: a reload still shows a valid certificate with a newer validity window. When the browser suite started the host, use the page's restart button.
3. The tray's `testd.test → dashboard` row opens the dashboard window, or `testd dashboard` prints the URL. The page lists the live name. `testd.test` is served by the edge for loopback clients only.
4. WebSocket: a dev server using HMR websockets connects over `wss` and survives 60 seconds idle.
5. SSE and a 100 MB upload complete without a truncated response.
6. Cookies: two names in one project (`app.example.test`, `api.example.test`) share a cookie set on `.example.test`, exactly as `app.example.com` and `api.example.com` would.
7. Stop the upstream, reload: the edge returns `502`. An unregistered name does not reach an application; DNS or TLS may reject it before any HTTP response.
8. Single-label warning: `testd run app:3000` prints the cross-site warning, and `app.test` still resolves.

### Record

For each browser: name and version, OS, pass or fail, and the leaf's SHA-256 fingerprint. Keep the result in the release notes or the host test log.

## Release

A free release is a version tag, CLI archives, public tray bundles, workspace SBOMs, generated Homebrew definitions, and `SHA256SUMS`. The tag workflow attaches them to a draft GitHub release only after the test, dependency-policy, and MSRV jobs pass. No commercial service or updater participates.

### Before tagging

- [ ] `cargo fmt --check`
- [ ] `cargo clippy --workspace --all-targets -- -D warnings`
- [ ] `cargo nextest run --workspace`
- [ ] `cargo deny check`
- [ ] `cargo +1.89.0 check --workspace --exclude testd-gui --locked`
- [ ] `cargo +1.90.0 check -p testd-gui --locked`
- [ ] `python3 scripts/release.py verify-version`
- [ ] Enable GitHub private vulnerability reporting for the public repository
- [ ] Run the release-validation suites above and retain reports for every claimed platform/browser
- [ ] `scripts/release-sbom.sh` and attach the SBOMs to the release
- [ ] Finalize the dated changelog entry; synchronize all package and Tauri bundle versions, and refresh `Cargo.lock` when dependencies change
- [ ] Run `testd doctor --json` against a clean VM and keep the output
- [ ] Verify the documented fresh-install path reaches a trusted HTTPS name, including Linux port privileges

### Build

Linux and macOS CLI:

```sh
cargo build --release --bin testd
```

Tray app (per platform):

```sh
cargo tauri build
```

Artifacts: CLI `.tar.gz` (Linux/macOS) or `.zip` (Windows), `.dmg` containing `.app` (macOS), `.exe` (Windows NSIS), `.AppImage`/`.deb` (Linux). CI builds macOS Apple Silicon and Intel, Linux x86-64, and Windows x86-64.

Windows ships the NSIS installer only. It installs per user into `%LOCALAPPDATA%\Testd` and asks for no elevation. Tauri's MSI is always per machine (`Program Files`, HKLM uninstall key, UAC prompt) with no option to make it per user, so it is not produced; `src-tauri/tauri.windows.conf.json` narrows the Windows target list. An enterprise deployment that needs a machine-wide install can build one with `cargo tauri build --bundles msi` and install that `.msi` by hand.

Removing the app does not remove the machine trust root or the DNS route. CLI uninstall is blocked on macOS and Windows until system ownership can be verified; do not remove unowned state. On Linux, run `testd uninstall` before removing the app; add `--purge` to delete the local config directory after system cleanup succeeds.

### Sign

Notarization, Authenticode, and the Homebrew tap have their own runbooks:
[Signing and notarization](#signing-and-notarization) and [Homebrew](#homebrew). Browser checks are in
[Browser validation](#browser-validation).

- macOS: `codesign` with the Developer ID Application certificate, then
  `xcrun notarytool submit --wait` and `xcrun stapler staple`. Ship the privacy
  manifest from `packaging/macos/PrivacyInfo.xcprivacy`.
- Windows: Authenticode sign the NSIS installer and the binaries; submit to SmartScreen.

This repository does not ship an updater. Releases are installed by the user: the archive, `brew upgrade`, or the system package manager. The binaries never check for a new version.

A Linux container can build and test the CLI and run `cargo tauri build`, but
cannot produce notarized or Authenticode-signed artifacts.

### Publish

- [ ] Tag `vX.Y.Z` and push the tag after the pre-tag checks
- [ ] Let CI build CLI archives and tray bundles, generate SBOMs, Homebrew definitions and checksums, and attach them to a draft GitHub release
- [ ] Download the draft assets and run artifact, host, browser, and desktop validation on the real supported platforms
- [ ] Apply signing/notarization separately if shipping signed desktop downloads; regenerate definitions and checksums from the final signed bytes
- [ ] Copy the generated `testd.rb` and `testd-gui.rb` into the Homebrew tap, then audit and install them locally
- [ ] Publish the draft with the changelog entry and validation results

The public workflow produces unsigned tray bundles and does not replace existing release assets on rerun. If an upload is already present, retain a backup before explicitly replacing it. Do not move an existing release tag to different source.

### Rollback

- GitHub: mark the broken version as withdrawn and publish a fixed version. Keep the previous good version and its checksums available. Existing installs keep working until the user chooses to replace them.
- Homebrew: revert the formula commit and `brew update`.
- macOS and Windows bundles are versioned per release, so a broken version can be
  withdrawn without affecting the previous one.

## Homebrew

The formula ships the CLI and daemon; the cask ships the tray app.

### Tap layout

Homebrew wants a repository named `homebrew-<tap>`. For `brew tap LapilliWorks/testd`:

```
homebrew-testd/
  Formula/testd.rb
  Casks/testd-gui.rb
```

`packaging/homebrew/*.rb.in` are the source templates. The release generator fills a version and real source/DMG digests and attaches installable `.rb` definitions to the public release. The cask selects Apple Silicon or Intel DMGs explicitly.

### Local test before publishing

```sh
# Definitions downloaded from the release (or generated locally from its final assets)
brew install --formula ./dist/testd.rb
brew install --cask ./dist/testd-gui.rb
brew test testd
brew audit --strict --online ./dist/testd.rb
brew uninstall --formula testd
brew uninstall --cask testd-gui
```

Verify Gatekeeper behavior on a clean Mac. An unsigned or unnotarized app may require a user override; do not claim a normal signed-install experience until signing and notarization have been verified.

### Digests

After the version tag exists and both final DMGs are in `dist/`:

```sh
python3 scripts/release.py homebrew --version 0.1.0 --artifacts dist --output dist
python3 scripts/release.py checksums dist
```

The source archive is fetched once and cached as `dist/testd-0.1.0-source.tar.gz`. The generator computes digests from that archive and both architecture-specific DMGs. It refuses to overwrite existing output definitions or checksums. Use a new staging directory after signing or changing assets; retain the previous bytes.

### Formula notes

- The formula builds from the source tarball with `cargo install` and declares `depends_on "rust" => :build`.
- `service do` runs `testd host` as a user agent with `keep_alive true`. `brew services start testd` is the supported path; the tray app is the alternative.
- The formula's `test do` runs `testd --help` and `testd doctor --json`. `doctor` exits non-zero without a config, so the test tolerates that exit code.

### Cask notes

- `depends_on macos: ">= :ventura"`.
- `app "Testd.app"`.
- `zap trash: "~/.config/testd"` only after the root and the DNS route are removed; `testd uninstall` does both. Document that order in the release notes, not as a cask stanza that silently deletes state.

### Bumping a release

1. Synchronize package and bundle versions and finalize the changelog entry
2. Tag the release and generate definitions from its final source archive and DMGs
3. Run `brew audit --strict --online`, `brew test testd`, and fresh formula/cask installs
4. Publish the matching definitions to the tap

## Signing and notarization

Optional signing of the public desktop artifacts: macOS notarization and Windows Authenticode need operator-supplied credentials. The public CI workflow does not sign or notarize. Source publication and CLI source builds do not require these credentials.

### macOS

#### Prerequisites

- Apple Developer Program membership (99 USD/year).
- A `Developer ID Application` certificate in the login keychain, with its private key. `security find-identity -v -p codesigning` lists it.
- An app-specific password for the Apple ID, or an App Store Connect API key.
- Xcode command line tools: `codesign`, `xcrun notarytool`, `stapler`, `spctl`.

Store the notary credentials once:

```sh
xcrun notarytool store-credentials testd-notary \
  --apple-id you@example.com \
  --team-id TEAMID1234 \
  --password "$APP_SPECIFIC_PASSWORD"
```

#### Build and sign

```sh
cargo tauri build --bundles app,dmg
```

Sign inside out: nested binaries and frameworks first, then the bundle. `cargo tauri build` signs with the identity it finds when `APPLE_SIGNING_IDENTITY` is set; otherwise sign manually:

```sh
codesign --force --options runtime --timestamp \
  --sign "Developer ID Application: Your Name (TEAMID1234)" \
  target/release/bundle/macos/Testd.app
codesign --force --options runtime --timestamp \
  --sign "Developer ID Application: Your Name (TEAMID1234)" \
  target/release/bundle/dmg/Testd_0.1.0_aarch64.dmg
```

Verify the signature and the hardened runtime before submitting:

```sh
codesign --verify --deep --strict --verbose=2 target/release/bundle/macos/Testd.app
codesign -d --entitlements - target/release/bundle/macos/Testd.app
```

No entitlements are required: the app is not sandboxed, uses no JIT, and shells out only through Tauri's process API. Keep it that way; every entitlement is a notarization review question.

#### Notarize

```sh
xcrun notarytool submit target/release/bundle/dmg/Testd_0.1.0_aarch64.dmg \
  --keychain-profile testd-notary --wait
xcrun stapler staple target/release/bundle/dmg/Testd_0.1.0_aarch64.dmg
xcrun stapler validate target/release/bundle/dmg/Testd_0.1.0_aarch64.dmg
```

`--wait` prints the submission id and the result. Keep the id in the release notes. Notarize the `.dmg` after signing the `.app` inside it, or notarize both; stapling the `.dmg` is what users download.

#### Verify as a user would

```sh
spctl -a -vv -t exec target/release/bundle/macos/Testd.app
```

Expect `accepted` and `source=Notarized Developer ID`. Also copy the `.dmg` to a clean account, mount it, and launch once: Gatekeeper must accept the first run without a right-click.

#### Common failures

| Symptom | Cause |
|---|---|
| `Invalid` with "The signature of the binary is invalid" | A nested binary was signed after the bundle, or not at all. Re-sign inside out. |
| `Invalid` with "hardened runtime is not enabled" | Missing `--options runtime`. |
| `Invalid` with "The executable does not have the hardened runtime enabled" on a dependency | A sidecar or dylib shipped unsigned. |
| `spctl` rejects but notarization passed | Staple was skipped; the app was moved out of the stapled dmg. |
| Submission stuck in `In Progress` for hours | Apple side; resubmit with `--wait` and the same profile. |

### Windows

- Sign the NSIS installer and the binaries with an Authenticode certificate (`signtool sign /fd sha256 /tr http://timestamp.digicert.com /td sha256`).
- Submit the installer to SmartScreen once per certificate; fresh certificates start without reputation and gather it per download.
- This repository does not ship an updater; see [Release](#release).

### What not to sign

The root certificate and its private key are local data, not artifacts. They never enter the release pipeline.
