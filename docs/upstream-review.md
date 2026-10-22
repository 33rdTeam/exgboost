# Review of the acalejos integration

The downstream baseline is `main` at `2bc87cb888f8de667e5954dbdee4d47b62e36bc5`
(fork version `0.12.1`). The reviewed upstream tip is
`c272b63c58fb36f22db26d39dec0108ed1b78496`. Their common ancestor is `5b3f3e6`
(fork version `0.11.0`). The existing merge `5686bb8` preserves upstream ancestry;
follow-up commits on `ip/sync-acalejos` restore downstream policy and address
review findings. `main` stays at the baseline until the result is reviewed.

This is a source and compatibility review of each incoming commit. Every
historical intermediate tree has not been rebuilt: some commits mix features and
packaging, and later commits correct earlier build problems. Validation applies
to the resulting downstream tree and the focused regressions.

## Compatibility contract retained

| Area | Downstream decision |
| --- | --- |
| Release identity | Keep `0.12.1`, Ian Perks, `iperks/exgboost`, and unprefixed version tags. Keep changes unreleased; bump the fork version only when preparing a reviewed release. |
| Native engine | Retain XGBoost `v3.3.0`; defer the separate upgrade to 3.4.2. |
| Runtime declarations | Retain Elixir `~> 1.17`, Nx `~> 0.9`, and NIF ABI entries `2.17`/`2.18`. |
| Dependencies | Keep the baseline lockfile, including Decimal 3.2.0, required image conversion, and the development SciData dependency. |
| CI | Retain Ubuntu 24.04/GCC 14 and OTP 28.3 with Elixir 1.18.4/1.19.5, plus OTP 29.0.2 with Elixir 1.20.2. |
| Precompilation | Retain `CCPrecompiler` target selection, filenames, release URL and tag triggers, and the existing macOS 15 ARM job. |
| Source builds | Retain Ninja, home cache layout, compiler/CMake overrides, `USE_LLVM_BREW`, and development/production build selection. |
| CUDA | Leave `ip/cuda-support` unchanged. Reconcile it separately after this baseline is accepted. |

These entries preserve the baseline configuration. They do not certify previously
untested targets. The legacy cross-compiler discovery and shared native cache
need independent packaging validation. No new macOS minimum or four-platform
release promise is introduced here.

## Incoming commits, in ancestry order

| Commit | Change and review decision |
| --- | --- |
| `3fe0f1d` — Update mix.exs to external 0.2 | Historical contributor change to Exterval 0.2 and package 0.5.2. The fork already has Exterval 0.2.0. Keep ancestry; retain current dependency and version metadata. |
| `2075de9` — update mix.lock | Historical dependency refresh, superseded by the fork's newer lockfile. Retain the baseline lockfile rather than importing historical downgrades. |
| `eda1b10` — Make exgboost compile with exterval 0.2 and clang > 3.5 | Reviewed interval sigil conversion, pointer cast in an old raw-address helper, and CMake policy override. Interval changes are already present; the unsafe helper was removed by the fork and stays removed. The retained XGBoost build does not need the historical CMake override. |
| `dfe688c` — Merge contributor PR #45 | Records the preceding three commits. Preserve ancestry without reviving historical APIs or metadata. |
| `e871b3e` — Merge contributor PR #47 | Integrates `cdmwebs/main` at `f793aee`, already in the fork's ancestry. Preserve history; this is not a new compatibility upgrade. |
| `62b2a74` — Integrate MatchSense contributor maintenance and NIF memory safety fixes | Includes the fork through `5b3f3e6`. Reviewed against that parent: memory safety, JSON/UBJ persistence, removal of raw-address helpers, and custom-gradient training are preserved. Retain without duplicate cherry-picks. |
| `f35b794` — Modernize XGBoost, native packaging, early stopping and project hygiene | Large mixed commit. Accept independent training, metric parsing, label validation, URI-wrapper, offline schema, and regression-test changes. Adapt custom metric direction (see findings). Defer XGBoost 3.4.2, Makefile/precompiler redesign, upstream release identity, four-platform packaging, dependency/coverage changes, and altered build baselines. Keep obsolete compiled-model notebook removal; its APIs no longer match the fork. |
| `a7bb517` — Exercise the compatibility matrix in PR builds and repair documentation links | Retain documentation fixes and exposed module docs. Keep the fork's PR CI trigger and runtime/compiler matrix. Defer the replacement native workflow, coverage dependency, and direct Castore 1.x test dependency; it conflicts with the retained development SciData/Castore 0.1 dependency. |
| `96a0b44` — Make macOS archives relocatable and keep image conversion optional | Defer macOS runtime redistribution/signing and the optional image converter: these alter distribution/dependency behavior. Keep image conversion required and retain the existing macOS build path. Retain the corrected description of forwarded plot options. |
| `a936e7e` — Define native resource types once for modern GCC linking | Accept. Resource globals are `extern` declarations with one definition in `exgboost.c`. Remove the linker flag masking duplicate definitions; fix the source instead of relying on permissive linking. |
| `1db8ef8` — Preserve the install prefix when precompiling and refresh CI tooling | Retain the portable Python C API checker and plotting documentation correction. Defer the new install layout, compiler map, and workflow/action versions. The fork's existing install prefix remains explicit. Add Makefile as a NIF prerequisite so build-flag edits rebuild the wrapper. |
| `e3fdc0d` — Merge PR #48 | Its resulting tree matches maintenance branch tip `1db8ef8`; no additional merge-resolution changes. Preserve ancestry and apply the decisions above. |
| `81014ea` — Publish verified CI-built Hex packages when GitHub releases go live | Defer as a unit. Publisher, manifest format, four-target ABI assumptions, and hardcoded acalejos URLs form a different release contract. Remove the imported publisher workflow, its helper scripts, and fixture tests. No automatic Hex publishing is enabled. |
| `336a6ec` — Date the 0.6.0 release being prepared | Upstream-only bookkeeping. Preserve the fork's changelog history and an Unreleased section instead of presenting 0.6.0 as a fork release. |
| `f9779c6` — Verify macOS runtime minimums and test downloads without credentials | Useful checks for the deferred distribution, but they assume bundled OpenMP/macOS 14 and upstream public URLs. Defer with that pipeline; do not introduce those compatibility promises. |
| `c272b63` — Merge PR #52 | Its resulting tree matches release branch tip `f9779c6`; no additional merge-resolution changes. Preserve ancestry without enabling the deferred release contract. |

## Functional review findings

### Custom metric names can reverse early stopping

Upstream inferred maximization from prefixes such as `auc` and `map` even for
`feval` results, contradicting the documented custom-metric minimization default.
A decreasing `auc-loss` sequence `3, 2, 1, 0` stopped after two rounds and kept 3.

The downstream fix retains metric provenance internally. Explicit `maximize`
always wins; otherwise custom metrics minimize and built-in metrics retain their
automatic direction. Public `Booster.eval_set/4` still returns three-tuples.
An empty `feval` result leaves built-in metric selection and direction intact.
Regression tests cover prefix collisions, explicit overrides, and falling
built-in AUC both with and without an empty custom evaluator.

### Training behavior changes need acceptance

The retained fixes correct early stopping patience and preserve the true best
iteration/score. They forward `disable_default_eval_metric`, retain custom
evaluation results, handle hyphenated dataset/parameterized metric names and
scientific notation, and reject invalid/non-finite native evaluation output.
Positive iteration/patience validation is stricter than before. These fixes may
change stopping rounds in existing workloads. Tests and an application workload
should be reviewed before promotion to `main`.

### Build policy and source correctness are separate

The URI compatibility wrapper and single-definition native resources work
independently of the deferred engine upgrade. CI forces source compilation via
`EXGBOOST_BUILD=true`, avoiding a false pass from an existing 0.12.1 native
download. The lockfile and dependency requirements remain aligned with `main`;
plotting image export remains available as before.

The retained home cache shares a NIF wrapper across OTP toolchains. Validation
must force recompilation with each toolchain's ERTS headers. Discovery of a
cross-compiler is not proof of a correctly built XGBoost dependency for that
target. These existing limitations are recorded rather than silently replaced
with upstream's native-only distribution policy.

The baseline's two build-type assignments were tab-indented immediately after
the `.PHONY` rule. GNU Make treated them as recipe text, leaving
`CMAKE_BUILD_TYPE` unset and letting CMake default to Release. Unindenting those
assignments restores the documented `MIX_ENV=prod`/development choice; `?=` also
allows an explicit override. Existing upstream build stamps still require a
fresh cache or rebuild when changing CMake options. Both retained packaging jobs
explicitly set `CMAKE_BUILD_TYPE=Release`, including their test step, so switching
Mix environments during packaging does not reuse a development engine build.

### Existing raw NIF fixture lifetime issue

An additional Elixir 1.17/OTP 26 run exposed an intermittent failure when saving
an unconfigured raw booster: XGBoost reported zero features after the cached
DMatrix was collected. The booster creation implementation is identical to
`main`. A controlled worker-process check reproduced failure when only the
booster survived, and success when its DMatrix remained alive. XGBoost 3.3 keeps
weak cache references until it first configures the booster.

The save/load fixture tests now explicitly use the DMatrix after the round trip,
keeping the caller-owned reference alive for the whole operation. No production
resource ownership or NIF signature is changed. The underlying raw-API lifetime
limitation remains a separate concern; this integration does not claim to fix it.

## Validation

The final downstream code was checked on Linux x86_64 with GCC 15.2.0, CMake
4.2.3, Ninja, and XGBoost 3.3.0 built from source. The engine was built in Release
and RelWithDebInfo configurations; the final runtime matrix used RelWithDebInfo.

| Elixir | Running OTP | Result | Selected native archive ABI |
| --- | --- | --- | --- |
| 1.17.3 | 26.2.1 | 88 tests + 4 doctests passed | 2.17 |
| 1.18.4 | 28.3 | 88 tests + 4 doctests passed | 2.17 |
| 1.19.5 | 28.3 | 88 tests + 4 doctests passed | 2.17 |
| 1.20.2 | 29.0.2 | 88 tests + 4 doctests passed | 2.18 |

The installed Elixir 1.18.4 executable was compiled against OTP 27 and run on
OTP 28.3. All NIF wrappers were rebuilt with the running OTP's ERTS headers.
An existing compatible Rebar executable and absolute isolated build paths were
used to avoid local tool-path issues; dependency versions were not changed.

Additional checks passed:

- Project compilation with warnings as errors on each runtime combination.
- Formatting and `git diff --check`.
- Training, dense/in-place prediction, and JSON/UBJ round trips on each combination.
- Actual loading and prediction under OTP 29 using the NIF 2.17 build made on OTP 26.
- Fork archive selection on each runtime: `iperks/exgboost`, unprefixed `0.12.1`,
  and the appropriate existing ABI slot. This checks selection, not a download
  of an unpublished reviewed archive.
- C API declarations and native shared-library exports for all 51 used symbols.
- The five custom-name regression tests fail against the unmodified upstream
  Booster/Training modules and pass with the provenance fix.
- The lockfile and devcontainer match `main`; the precompile workflow differs
  only by explicit Release build settings. Both protected branch pointers remain
  unchanged (`main`: `2bc87cb`; CUDA: `0b57201`).

Before promotion, review the functional training changes and exercise a real
application workload. The exact Ubuntu 24.04/GCC 14 CI environment, macOS,
cross-architecture artifacts, and public archive downloads were not exercised
locally. Those require the retained CI/release jobs and actual consumer builds;
no compatibility claim beyond the observed local checks is made.
