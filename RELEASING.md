# Releasing this fork

This integration retains the release contract from `main` at `2bc87cb`:

- Package: `exgboost`, currently version `0.12.1`, maintained by Ian Perks.
- Source, homepage, and native download URLs: `https://github.com/iperks/exgboost`.
- Tags are unprefixed versions such as `0.12.1`, not `v0.12.1`.
- Bundled XGBoost: `v3.3.0`.
- Fetchable NIF ABI entries: `2.17` and `2.18`.
- Compiler and target selection: the existing `CCPrecompiler` configuration.
- Linux release jobs: Ubuntu 24.04 and GCC/G++ 14, using the existing OTP/Elixir matrix.
- macOS release job: the existing macOS 15 ARM runner with Homebrew OpenMP.

The URL contract remains:

```text
https://github.com/iperks/exgboost/releases/download/<version>/<archive>
exgboost-nif-<nif-abi>-<target>-<version>.tar.gz
```

The upstream history is recorded on `ip/sync-acalejos`, but its `0.6.0` identity,
four-native-target/OTP-26-only distribution, macOS runtime bundling, and automated
Hex publisher are intentionally deferred. See [the commit review](docs/upstream-review.md).

## Validate the integration

```sh
mix deps.get --check-locked
EXGBOOST_BUILD=true mix quality
make check-xgboost-c-api
EXGBOOST_BUILD=true mix run scripts/smoke.exs
```

Use the same `XGBOOST_CACHE` override for every command when testing with an
isolated cache. For production artifacts use `MIX_ENV=prod`. The Makefile retains
its home-cache layout, Ninja generator, `CC`/`CXX`, `CMAKE_FLAGS`,
`USE_LLVM_BREW`, and development/production build types. `BUILD_JOBS` bounds
Ninja parallelism without changing artifact names or download selection.

The legacy NIF cache is shared across OTP toolchains. When switching OTP versions,
force a NIF rebuild with the new ERTS headers or use a separate checkout/cache;
do not assume a cached wrapper was compiled for the current runtime. This is an
existing limitation, not a new cross-version compatibility guarantee.

## Prepare a release after review

1. Review the integration before changing `main`. Keep CUDA work on its separate
   feature branch.
2. Choose a new fork version for these unreleased changes. Update `@version`,
   the changelog, and installation examples together. Do not republish `0.12.1`
   with different code, checksums, or archives.
3. Run CI on the retained OTP/Elixir matrix and test the actual deployment OS,
   architecture, and runtime library requirements. A successful host build is not
   proof that cross-compiled archives or macOS binaries work.
4. Use the existing `.github/workflows/precompile.yml` and an unprefixed version
   tag when the release is approved. Native archives remain in this fork's
   GitHub release; there is no newly enabled Hex publication workflow.
5. Generate and verify `checksum.exs` for the actual release archives before a
   separate, approved Hex publication. Ignore any stale local checksum manifest
   left by earlier CPU/CUDA experiments. Check that consumers fetch the matching
   fork URL and archive rather than silently falling back to compilation.

No release, tag, GitHub upload, or Hex publication is part of the integration review.
