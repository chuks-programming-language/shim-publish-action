# shim-publish-action

Reusable GitHub Actions workflow for building and publishing native Rust/C
shim binaries to the [Chuks package registry](https://packages.chuks.org).

This is the Chuks equivalent of [cibuildwheel](https://cibuildwheel.pypa.io/)
for Python — it cross-compiles your package's `shim/` cdylib on every
supported platform and uploads each binary so end users never need
a Rust or C toolchain installed.

## Supported platforms

| Platform ID       | Runner                | Target triple                 |
| ----------------- | --------------------- | ----------------------------- |
| `darwin-arm64`    | `macos-14`            | `aarch64-apple-darwin`        |
| `darwin-x86_64`   | `macos-13`            | `x86_64-apple-darwin`         |
| `linux-x86_64`    | `ubuntu-22.04`        | `x86_64-unknown-linux-gnu`    |
| `linux-aarch64`   | `ubuntu-22.04-arm`    | `aarch64-unknown-linux-gnu`   |
| `windows-x86_64`  | `windows-2022`        | `x86_64-pc-windows-msvc`      |

## Usage

In your package repository (e.g. `chuks-lang/chuks_numpy`) create
`.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    tags: ["v*"]
  workflow_dispatch:
    inputs:
      version:
        description: "Package version (must match chuks.json)"
        required: true

jobs:
  publish:
    uses: chuks-programming-language/shim-publish-action/.github/workflows/build.yml@v1
    with:
      shim-dir: shim                       # path to Cargo.toml, default "shim"
      release: true                        # cargo build --release, default true
      platforms: |                         # optional subset, default all five
        darwin-arm64
        darwin-x86_64
        linux-x86_64
        linux-aarch64
        windows-x86_64
    secrets:
      CHUKS_TOKEN: ${{ secrets.CHUKS_TOKEN }}
```

Add a `CHUKS_TOKEN` repository secret (an API token with `packages:publish`
scope from your registry account).

## What it does

For each platform in the matrix:

1. Checks out your repository.
2. Installs Rust + the target's cross-compilation toolchain.
3. Runs `cargo build --release --manifest-path <shim-dir>/Cargo.toml`.
4. On macOS, runs `xattr -cr` + `codesign --force --sign -` on the produced
   dylib to clear the `com.apple.provenance` attribute that otherwise
   stalls binaries on end-user machines.
5. Installs the matching `chuks` CLI release.
6. Runs `chuks publish --prebuilt-only --platform <id> --file <built-lib>`,
   which sha256-hashes the binary, base64-encodes, and POSTs it to
   `https://packages.chuks.org/packages/<name>/<version>/prebuilt`.

The registry rejects overwrites (HTTP 409), so a successful publish for a
given `(name, version, platform)` triple is immutable. Re-running the
workflow after a successful publish is a no-op for that platform.

## Source publish

This workflow only publishes **prebuilt binaries**. The source tarball
(your Chuks code + `chuks.json` + `shim/` Cargo sources) must be published
separately with a plain `chuks publish` — typically as a manual step or a
separate job that runs once per version before the matrix kicks off.

## License

MIT
