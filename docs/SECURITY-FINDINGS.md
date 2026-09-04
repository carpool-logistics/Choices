# Security Findings — Oneleet Code Security

Fork of [jshjohnson/Choices](https://github.com/jshjohnson/Choices), maintained at
`carpool-logistics/Choices`.

**Supply-chain note:** TMS consumes this repository directly from the default branch
(`"choices.js": "github:carpool-logistics/Choices#master"` in `TMS/package.json`), not
from the npm registry. Anything that can write to `master` — or poison an artifact this
repo's CI produces — lands in the TMS build. That raises the severity of CI findings
here above what it would be for a standalone library, and is why the two Actions
findings below were treated as real rather than accepted as fork-inherited upstream
debt.

Parent epic WA-5810. TMS tracked under WA-5804; driver under DMA-1119.

---

## 1. Command injection via `github.head_ref` — `.github/workflows/build-and-test.yml:49`

**Status: TRUE POSITIVE — remediated.**

`${{ github.head_ref }}` was interpolated directly into a `run:` step. GitHub Actions
substitutes `${{ }}` into the script text *before* the shell parses it, so the branch
name is evaluated as shell source, not passed as data.

`head_ref` is attacker-controlled. `git check-ref-format` was used to confirm which
shell metacharacters a branch name may legally contain — verified allowed:

```
a;id      a$(id)     a`id`     a|id     a&&id     a'x     a"x
```
(only whitespace is rejected)

Demonstrated against the exact command shape used by the step:

```
head_ref = main;echo PWNED-ARBITRARY-CODE-EXECUTED;#

before:  echo UPLOAD -B main;echo PWNED-ARBITRARY-CODE-EXECUTED;# -Z
         -> UPLOAD -B main
         -> PWNED-ARBITRARY-CODE-EXECUTED        <-- executed
after:   echo UPLOAD -B "$HEAD_REF" -Z
         -> UPLOAD -B main;echo PWNED-ARBITRARY-CODE-EXECUTED;# -Z   <-- inert
```

**Reachability:** this workflow triggers on `push` to `master`, where `head_ref` is
empty, so the sink is *latent rather than live* in its current configuration. It was
fixed regardless: the guard is the trigger, not the code, and a future trigger change
would silently arm it. The step holds `CODECOV_TOKEN`.

**Fix:** the value is bound to an `env:` var and referenced as `"$HEAD_REF"`. Values
passed through `env:` are never re-parsed by the shell. Same treatment for
`github.sha`.

## 2. Command injection via `github.head_ref` — `.github/workflows/unit-tests.yml:33`

**Status: TRUE POSITIVE — remediated. This was the live one.**

Identical sink and identical fix, but this workflow triggers on `pull_request`, so
`head_ref` *is* populated and attacker-controlled by anyone who can open a PR.

Impact by PR origin:

- **Fork PR** — GitHub withholds secrets, so `CODECOV_TOKEN` is empty, but arbitrary
  code still executes on the runner. The pivot is the `~/.npm` and Cypress caches
  (`actions/cache` in `e2e-tests.yml` / `browsers.yml`): a poisoned cache entry is
  restored by later runs on trusted branches, which given the `github:...#master`
  consumption path reaches TMS.
- **Same-repo branch PR** (any collaborator with push access, or a compromised
  account) — secrets *are* injected, making `CODECOV_TOKEN` directly exfiltratable.

**Fix:** as above.

### Verification for 1 & 2

All nine workflow files were parsed and every `run:` block scanned; no `${{ }}`
interpolation remains inside any `run:` in the repository.

`e2e-tests.yml:69-71` also consumes `github.head_ref` / `github.event.sender.login`,
and `browsers.yml:36` consumes `toJson(github)` — all already via `env:`, which is the
correct pattern. Correctly not flagged; left unchanged.

## 3. Dependencies can run arbitrary install scripts — `package-lock.json`

**Status: TRUE POSITIVE — remediated.**

`npm ci` executes `preinstall`/`install`/`postinstall` for any dependency that declares
one. The lockfile resolves **1140 packages**, so absent a global guard, a single
compromised transitive package executes arbitrary code on every developer machine and
CI runner at install time — the standard npm supply-chain path.

Current actual exposure is small (4 packages declare install scripts, all
devDependencies, two of them optional native builds):

| package | version | note |
| --- | --- | --- |
| `cypress` | 9.4.1 | downloads the runner binary — functionally required |
| `core-js` | 3.49.0 | funding banner only |
| `fsevents` | 2.3.3 | optional native, macOS watch |
| `@parcel/watcher` | 2.5.6 | optional native, watch |

The finding is about the standing capability, not those four — any future version bump
can introduce a `postinstall`.

**Fix:** `ignore-scripts=true` in `.npmrc`.

Behaviour confirmed empirically before adopting it:

- dependency `postinstall` is **not** executed on install ✅
- `npm run <script>` still works, so `npm run build` / `lint` / `test` in CI are
  unaffected ✅
- `prepublishOnly` is **also** suppressed ⚠️ — this package's own build would have been
  skipped on publish, so `deployment.yml` now runs `npm run build` explicitly before
  `npm publish`.
- Cypress's binary download is suppressed, so `e2e-tests.yml` and `build-and-test.yml`
  now run `npx cypress install` explicitly. Workflows that already set
  `CYPRESS_INSTALL_BINARY: 0` needed no change.

Scope: a package's `.npmrc` is not read by its consumers, so this hardens development
and CI in this repository only. It does not change what TMS executes on install — and
does not need to: this package declares no `prepare` script, so the `github:` install
path in TMS runs no build script and uses the committed
`public/assets/scripts/choices.js`.

---

## CI toolchain remediation

Not an Oneleet finding, but required to land the above: the first PR run failed at
workflow setup with

```
Error: This request has been automatically failed because it uses a deprecated
version of `actions/cache: v2`.
```

GitHub hard-fails the retired cache and artifact actions, so the job never reached a
step. Fixed in the same change, along with the EOL toolchain behind it:

| item | was | now | why |
| --- | --- | --- | --- |
| `actions/cache` | v2 | v4 | hard-failed by GitHub |
| `actions/upload-artifact` | v2 | v4 | same shutdown; `browsers.yml` |
| `::set-output` | deprecated | `$GITHUB_OUTPUT` | feeds the Cypress cache path |
| `actions/checkout` | v2 | v4 | EOL runtime |
| `actions/setup-node` | v2 | v4 | EOL runtime |
| Node | 12 | 24 | see below |

The Node bump was **not** cosmetic. `package-lock.json` is `lockfileVersion: 3`, which
npm 7+ writes without the back-compat top-level `dependencies` key (confirmed absent).
Node 12 ships npm 6, which reads only that key — so `npm ci` could not have installed
this lockfile. The lockfile was regenerated in f7ccc68 ("unblock Node 20+/24") but the
workflows were never moved, leaving CI unable to install. Node 24 matches that commit's
stated target.

Verified locally on modern Node before bumping: `npm ci` resolves 1127 packages and the
full unit suite passes (377 tests, exit 0). The Cypress binary cache stayed empty,
confirming `ignore-scripts` works end to end.

Still unverified: Cypress 9.4.1 (Feb 2022) has not been exercised on Node 24, and could
not be meaningfully tested on the darwin/arm64 machine used for this work — Cypress 9
predates Apple Silicon support. If the e2e job fails after this, a Cypress upgrade is
the likely next step and belongs in its own change.


### `browsers.yml` — quarantined

Bumping the actions above made this workflow eligible to run for the first time in
years (its `paths:` filter includes its own file), surfacing two further breaks:

- **`TypeError: pixelmatch is not a function`.** The job installs `pixelmatch`
  unpinned and outside `package-lock.json`, so it floated to v7.2.0. pixelmatch v6+ is
  ESM-only while `actions-scripts/*.js` use `require()`, which on Node 24 yields the
  module namespace object rather than the function. Reproduced locally: unpinned →
  `typeof object`; `pixelmatch@^5.3.0` → `typeof function`. Now pinned to the last
  CommonJS major.
- **`Could not find Chrome`.** A regression from `ignore-scripts=true` — Puppeteer's
  postinstall downloads the browser. Same class as the Cypress case, missed initially
  because Puppeteer is installed ad-hoc in this workflow rather than via
  `package.json`. Fixed with an explicit `npx puppeteer browsers install chrome`.

Both jobs are now `continue-on-error: true`. The `__snapshots__` PNG baselines were
last regenerated 2022-02-13 (7c360b4) and the scripts fail at `pixelDifference > 200`,
so current browsers (Chrome 152 at time of writing) will drift past the threshold
regardless of the fixes above. `selenium-webdriver` is likewise installed unpinned
against 2022-era API usage. The matrix therefore still runs and reports, but cannot
block merges until someone regenerates the baselines — upstream-flavoured work that
does not belong in a security remediation.

**Supply-chain note:** the ad-hoc `npm i` steps in this workflow bypass
`package-lock.json` entirely — no pinning, no integrity checking. That is the same
family as finding 3 and worth closing separately.


## Noted, not remediated

Out of scope for these three findings; recorded so they are tracked rather than
silently carried.

1. **Codecov bash uploader is `curl | bash`.** Both coverage steps run
   `bash <(curl -s https://codecov.io/bash)` in a step holding `CODECOV_TOKEN` —
   an unpinned remote script piped into a shell. This is the *same* uploader that was
   backdoored in the 2021 Codecov compromise to exfiltrate CI environment variables.
   The endpoint was confirmed still live during this review (serves uploader v1.0.6),
   so this executes on every run. Replacing it with `codecov/codecov-action` pinned to
   a commit SHA would remove the vector; deferred because it is a functional change to
   a secret-bearing step that cannot be verified outside CI, and its failure is
   currently masked by `|| echo 'Codecov upload failed'`.
2. **Actions pinned to mutable tags** (`actions/checkout@v4`, `actions/cache@v4`,
   `peaceiris/actions-gh-pages@v3`, `release-drafter/release-drafter@v5`). A tag can be
   repointed by the action owner. SHA pinning is the remaining hardening step.
