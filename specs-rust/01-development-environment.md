# 01 — Development Environment (Nix, Bazel, Task runner, lint/codegen, CI)

> **Status:** Conforms to `00-overview-and-conventions.md`. Crate names, the
> external-dependency table (§4 of `00`), and the workspace layout (§3 of `00`)
> are authoritative; this file does not change them. It defines *how the repo is
> built, tested, linted, generated, and gated in CI*, mirroring the Go tooling
> 1:1 in capability while being idiomatic Rust.

This document is the Rust analogue of the Go repo's development tooling: the Nix
dev shell (`flake.nix`), the Bazel bzlmod + gazelle setup, the
[Task](https://taskfile.dev) runner (`Taskfile.yml` + `scripts/run_task.sh`), the
lint/format/codegen tasks, and the CI gates (`.github/workflows/ci.yml`). For
each Go mechanism we name the exact Rust replacement, pin versions, and provide
copy-pasteable config.

Source-of-truth Go files this mirrors: `flake.nix`, `.envrc`, `MODULE.bazel`,
`.bazelrc`, `.bazelversion`, `Taskfile.yml`, `scripts/run_task.sh`,
`scripts/build.sh`, `scripts/lint.sh`, `.golangci.yml`, `.editorconfig`,
`header.yml`, `.github/workflows/ci.yml`.

---

## 1. Principles carried over from the Go repo

The Go repo's tooling has a few load-bearing properties we reproduce exactly:

1. **Pinned, reproducible toolchain.** Go pins `1.25.x` in `go.mod`; Nix pins the
   whole dev shell; Bazel sources the Go SDK version *from* `go.mod` so there is a
   single source of truth. Rust mirrors this: `rust-toolchain.toml` is the single
   source of truth, the Nix flake reads it via `rust-overlay`, and Bazel's
   `rust` toolchain is pinned to the same version.
2. **One entrypoint for every workflow.** Everything goes through
   `./scripts/run_task.sh <task>`; you never need `task`/`cargo`/`bazel`
   installed globally, and tasks run inside the Nix dev shell when Nix is
   present. We keep `scripts/run_task.sh` and `Taskfile.yml` verbatim in spirit
   (same task names where they map), swapping the underlying commands.
3. **CI never calls the build tool directly** — it always calls a task. This keeps
   local and CI behavior identical. We preserve that.
4. **Dirty-tree codegen gates.** Go regenerates protobuf/mocks/canoto/bazel
   metadata then fails if `git` is dirty. Rust does the same for protobuf
   (`tonic`/`prost`), mocks (`mockall` — only where artifacts are committed),
   `Cargo.lock`, and Bazel BUILD files.
5. **A stricter lint pass for the SAE code.** Go runs `gosec` with `G115`
   (integer-overflow conversions) on `vms/saevm/...` via `lint-saevm`. Rust runs
   `clippy::pedantic` + `clippy::arithmetic_side_effects` + `overflow-checks` on
   the `ava-saevm*` crates via `lint-saevm` (§7.7 of `00`).
6. **`-tags test` parity.** Go compiles tests with a `test` build tag. Rust uses a
   `test-util` Cargo feature (and `#[cfg(test)]`) for the equivalent; the unit-test
   task passes `--all-features` so test-only code compiles.

> The full top-level layout is in `00` §3. The build files that live at the repo
> root are recapped in §11 below.

---

## 2. Toolchain pinning — `rust-toolchain.toml`

Go pins the exact patch version in `go.mod` (`1.25.10`) and a CI check
(`check-go-version`) asserts every module agrees. Rust's equivalent single source
of truth is `rust-toolchain.toml` at the repo root. It is consumed by `rustup`,
by the Nix flake (`rust-overlay` reads this file directly), and referenced by the
Bazel toolchain version.

**Channel policy (mirrors Go's "pin an exact patch, bump deliberately"):** pin an
exact stable patch release, never a floating `stable`. Bump it in one place; CI's
`check-rust-version` job (the analogue of `check-go-version`) asserts the Bazel
toolchain and the `nextest`/CI matrix agree with this file.

```toml
# rust-toolchain.toml — single source of truth for the Rust version.
# Bump deliberately (one PR); CI asserts Bazel + Nix + workflow matrix agree.
[toolchain]
channel = "1.90.0"        # exact stable patch; bump in lock-step with reth's MSRV
profile = "minimal"        # we add components explicitly below
components = [
    "rustfmt",
    "clippy",
    "rust-src",            # needed by rust-analyzer and some build deps
    "llvm-tools",          # for cargo-llvm-cov (coverage), mirrors -coverprofile
]
# Cross/extra targets compiled in CI release jobs (mirror Go's GOOS/GOARCH matrix).
targets = [
    "x86_64-unknown-linux-gnu",
    "aarch64-unknown-linux-gnu",
    "x86_64-apple-darwin",
    "aarch64-apple-darwin",
]
```

> **Why an exact patch, and how to choose it:** reth and the `revm`/`alloy`
> stack drive the effective MSRV. Pin to the highest stable that the chosen reth
> release supports, exactly as Go pins to what the codebase requires. When reth is
> bumped (`10-cchain-evm-reth.md`), bump this file in the same PR.

Edition: `edition = "2021"` is set per-crate in each `Cargo.toml` (or `2024` once
the whole tree compiles on it — decided workspace-wide, not here). The workspace
`Cargo.toml` sets `[workspace.package] edition`/`rust-version` so all crates
inherit it.

---

## 3. Nix flake — `flake.nix` / `flake.lock` / `.envrc`

### 3.1 Design

The Go flake (`flake.nix`) provides one `devShells.default` per supported system,
built from `nixpkgs/nixos-25.11`, containing: bazelisk (+ a `bazel` symlink),
`go-task`, the pinned Go, monitoring/kube tools, linters (`shellcheck`,
`buildifier`, `yamlfmt`), protobuf tooling (`buf`, `protoc-gen-*`), `ripgrep`,
`jq`, `solc`, `s5cmd`. A `shellHook` puts `scripts/` and the Go bin dir on
`PATH`. `.envrc` opt-in-enables the flake via direnv + `nix-direnv`.

The Rust flake keeps the **same shape and the same supported-systems list**, adds
`rust-overlay` as an input, and swaps the language toolchain + tooling:

- **Toolchain:** `rust-bin.fromRustupToolchainFile ./rust-toolchain.toml` from
  `oxalica/rust-overlay` (reads the pinned channel/components/targets directly,
  so there is exactly one version source). We use `rust-overlay` over `fenix`
  because it reads `rust-toolchain.toml` natively and lets us pin a specific
  stable without an extra flake input or manifest hash.
- **Cargo tooling:** `cargo-nextest` (test runner), `cargo-deny` (dependency
  policy = `go-mod-tidy`/license analogue), `cargo-llvm-cov` (coverage),
  `cargo-audit` (advisory DB), `cargo-machete`/`cargo-udeps` (unused-dep check),
  `taplo` (TOML fmt for `Cargo.toml`s), `just` (optional convenience runner),
  `go-task` (the canonical runner — see §5).
- **Native build deps** for our FFI crates: `rust-rocksdb` builds RocksDB from
  source (needs `clang`/`libclang`, `cmake`, `llvmPackages`); `firewood` and
  `secp256k1`/`blst` need a C/C++ toolchain (`clang`, `gcc`); `rustls`'s default
  provider and some deps want `pkg-config` + `openssl`. We ship `clang`/`libclang`
  (with `LIBCLANG_PATH`), `cmake`, `pkg-config`, `openssl`, and `protobuf`/`buf`.
- **Proto tooling:** `protobuf` (provides `protoc`) and `buf` — the proto sources
  are shared with Go (`00` §3 `proto/`), so we keep `buf` for lint/breaking
  checks; `tonic-build`/`prost` do the Rust codegen (§8).
- **Ops/dev tooling kept from Go:** monitoring (`prometheus`, `promtail`), kube
  (`kubectl`, `k9s`, `kind`, `kubernetes-helm`), `jq`, `ripgrep`, `s5cmd`,
  `bazelisk` (+ `bazel` symlink), `shellcheck`, `yamlfmt`, `buildifier` (for
  Bazel `BUILD` files), `actionlint`.

```nix
{
  # To use:
  #  - install nix: `./scripts/run_task.sh install-nix`
  #  - run `nix develop` or use direnv (see CONTRIBUTING.md / .envrc)
  description = "avalanche-rs development environment";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-25.11";
    rust-overlay = {
      url = "github:oxalica/rust-overlay";
      inputs.nixpkgs.follows = "nixpkgs";
    };
    flake-utils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, rust-overlay, flake-utils }:
    let
      # Same supported systems as the Go flake.
      allSystems = [
        "x86_64-linux"
        "aarch64-linux"
        "x86_64-darwin"
        "aarch64-darwin"
      ];
    in
    flake-utils.lib.eachSystem allSystems (system:
      let
        pkgs = import nixpkgs {
          inherit system;
          overlays = [ (import rust-overlay) ];
        };

        # Single source of truth for the Rust version: rust-toolchain.toml.
        rustToolchain = pkgs.rust-bin.fromRustupToolchainFile ./rust-toolchain.toml;
      in
      {
        devShells.default = pkgs.mkShell {
          packages = with pkgs; [
            # Pinned Rust toolchain (rustc, cargo, clippy, rustfmt, rust-src, llvm-tools)
            rustToolchain

            # Cargo tooling
            cargo-nextest        # canonical test runner (mirrors `go test`)
            cargo-deny           # dependency policy: licenses/bans/advisories/sources
            cargo-audit          # RustSec advisory scan
            cargo-llvm-cov       # coverage (mirrors -coverprofile/-covermode)
            cargo-machete        # unused-dependency detection
            taplo                # TOML formatter/linter for Cargo manifests
            just                 # optional convenience runner (Taskfile is canonical)

            # Build / FFI requirements (rocksdb, firewood, blst, secp256k1, rustls)
            clang
            llvmPackages.libclang
            cmake
            pkg-config
            openssl
            git

            # Bazel
            bazelisk
            (runCommand "bazel" {} ''mkdir -p $out/bin && ln -s ${bazelisk}/bin/bazelisk $out/bin/bazel'')
            buildifier           # format BUILD.bazel files

            # Task runner (canonical entrypoint)
            go-task

            # Protobuf (sources shared with Go) + lint/breaking checks
            protobuf             # provides protoc
            buf

            # Monitoring / kube (kept from Go flake)
            prometheus
            promtail
            kubectl
            k9s
            kind
            kubernetes-helm

            # Linters / misc
            shellcheck
            yamlfmt
            actionlint
            jq
            ripgrep
            solc                 # solidity compiler (EVM test contracts)
            s5cmd                # rapid S3 interactions for reexec datasets
          ];

          # rocksdb/firewood/secp256k1 builds find libclang via this var.
          LIBCLANG_PATH = "${pkgs.llvmPackages.libclang.lib}/lib";

          shellHook = ''
            export PATH="$PWD/scripts:$PWD/bin:$PATH"
            # Faster, reproducible incremental builds
            export CARGO_HOME="''${CARGO_HOME:-$PWD/.cargo-home}"
          '';
        };
      });
}
```

`flake.lock` is committed and bumped via `nix flake update` (the analogue of
bumping `nixos-25.11`); CI uses the locked inputs.

### 3.2 `.envrc`

Mirrors the Go `.envrc`: opt-in flake activation via `nix-direnv`, prepends
`scripts/` and `bin/` to `PATH`, and sets repo-local defaults.

```bash
# .envrc — opt-in nix dev shell via direnv (mirrors avalanchego's .envrc)
if [ -n "${AVALANCHE_RS_DIRENV_USE_FLAKE}" ]; then
  if ! command -v nix > /dev/null; then
    echo "To enable a dev shell via this .envrc: ./scripts/run_task.sh install-nix"
  else
    if ! has nix_direnv_version; then
      source "$(nix build nixpkgs#nix-direnv --no-link --print-out-paths)/share/nix-direnv/direnvrc"
    fi
    use flake
  fi
fi

PATH_add bin
PATH_add scripts

# Default path to the locally built node binary (drop-in name kept: `avalanchego`).
export AVALANCHEGO_PATH="${AVALANCHEGO_PATH:-$PWD/target/release/avalanchego}"

# Plugin dir parity with Go (rpcchainvm plugins).
mkdir -p "$PWD/build/plugins"
export AVAGO_PLUGIN_DIR="${AVAGO_PLUGIN_DIR:-$PWD/build/plugins}"

source_env_if_exists .envrc.local
```

`scripts/run_task.sh` and `scripts/nix_run.sh` are carried over verbatim from Go
(they wrap tasks in `nix develop --command` when Nix is available and otherwise
run `task` from PATH); the only change is the binary they ultimately invoke.

---

## 4. Bazel — bzlmod + rules_rust + crate_universe + gazelle

### 4.1 Background and decision

avalanchego uses **bzlmod** (`MODULE.bazel`), `rules_go` + `gazelle` to generate
`BUILD.bazel` from `go.mod`/`go.work`, and patches a few external deps. The Rust
analogue:

- **`rules_rust` 0.70.0** (latest as of 2026-04-22; bump in lock-step with the
  toolchain). Pinned in `.bazelversion`-adjacent fashion via `MODULE.bazel`.
- **`crate_universe`** is rules_rust's gazelle-for-crates: it consumes
  `Cargo.toml` + **`Cargo.lock`** (Cargo is the source of truth) and materializes
  Bazel repos for every external crate. This is the direct analogue of
  `go_deps.from_file(go_work=...)`.
- **`gazelle_rust`** (Calsign/gazelle_rust, requires rules_rust ≥ 0.40) generates
  and updates the per-crate `BUILD.bazel` (`rust_library`/`rust_binary`/`rust_test`)
  by parsing `.rs` files and `use` statements. This is the analogue of
  `bazelisk run //:gazelle`.

> **Decision — Cargo is the source of truth, Bazel consumes it.** This is the
> standard, recommended coexistence pattern. Developers use Cargo day-to-day;
> `Cargo.lock` is committed; `crate_universe` reads it so Bazel and Cargo never
> disagree on dependency versions. `bazel build //...` / `bazel test //...` is the
> hermetic/CI path and the cross-language (proto, vendored C) path. If
> `gazelle_rust` proves too immature for a corner of the tree, the **fallback** is
> `crate_universe` for third-party crates + hand-maintained `BUILD.bazel` for
> first-party crates (generation is a convenience, not a correctness requirement).

### 4.2 `MODULE.bazel`

```starlark
module(
    name = "avalanche_rs",
    version = "0.0.0",
)

# --- Core rules -----------------------------------------------------------
bazel_dep(name = "rules_rust", version = "0.70.0")
bazel_dep(name = "rules_proto", version = "7.1.0")
bazel_dep(name = "toolchains_protoc", version = "0.4.3")  # hermetic protoc
bazel_dep(name = "platforms", version = "1.0.0")
bazel_dep(name = "bazel_skylib", version = "1.7.1")

# Gazelle + the Rust language plugin (analogue of rules_go's gazelle).
bazel_dep(name = "gazelle", version = "0.45.0")
bazel_dep(name = "gazelle_rust", version = "0.2.0")

# --- Rust toolchain: pinned to rust-toolchain.toml ------------------------
rust = use_extension("@rules_rust//rust:extensions.bzl", "rust")
rust.toolchain(
    edition = "2021",
    # Keep in lock-step with rust-toolchain.toml; CI's check-rust-version asserts.
    versions = ["1.90.0"],
)
use_repo(rust, "rust_toolchains")
register_toolchains("@rust_toolchains//:all")

# --- crate_universe: external crates from Cargo.lock ----------------------
# Cargo is the source of truth. This reads the workspace manifests + Cargo.lock
# and generates @crates//... repos for every third-party crate.
crate = use_extension("@rules_rust//crate_universe:extensions.bzl", "crate")
crate.from_cargo(
    name = "crates",
    cargo_lockfile = "//:Cargo.lock",
    manifests = ["//:Cargo.toml"],  # workspace root; members discovered via [workspace]
)
use_repo(crate, "crates")

# --- prost/tonic toolchain for proto codegen under Bazel (see §8) ---------
prost = use_extension("@rules_rust//proto/prost:extensions.bzl", "rust_prost")
prost.toolchain(
    prost_runtime = "@crates//:prost",
    prost_types = "@crates//:prost-types",
    tonic_runtime = "@crates//:tonic",
)
use_repo(prost, "rules_rust_prost")
register_toolchains("@rules_rust_prost//:default_prost_toolchain")
```

> **Patched deps:** the Go tree patches `blst`, `firewood-go-ethhash`,
> `gnark-crypto`, `libevm`. In Rust most of these vanish (we link the Rust
> `blst`/`firewood`/`secp256k1` crates), but `crate_universe` supports the same
> escape hatches — `crate.annotation(...)` to inject `build_script_env`,
> `additive_build_file_content`, `deps`, and `gen_build_script = "off"` for crates
> with hand-rolled `build.rs` (RocksDB, firewood, blst). Record each annotation
> with a comment citing why, mirroring the Go `GO_OVERRIDES` block.

### 4.3 Root `BUILD.bazel` — the gazelle binary

```starlark
load("@gazelle//:def.bzl", "gazelle", "gazelle_binary")

# gazelle:rust_cargo_lockfile //:Cargo.lock
# gazelle:rust_crates_prefix @crates//:

gazelle_binary(
    name = "gazelle_bin",
    languages = ["@gazelle_rust//rust_language"],
)

gazelle(
    name = "gazelle",
    gazelle = ":gazelle_bin",
)

# Verify generated BUILD files are current (analogue of //:gazelle_test).
gazelle(
    name = "gazelle_check",
    gazelle = ":gazelle_bin",
    mode = "diff",
)
```

`bazel run //:gazelle` regenerates BUILD files; `bazel run //:gazelle_check`
(or the `_lint`/CI diff mode) fails if they are stale — the analogue of
`check-bazel-gazelle-generate`.

### 4.4 `.bazelrc`

Mirrors the Go `.bazelrc` (lockfile auto-update, platform-specific config,
default-on stricter mode, fast local mode, disk cache, release stamping). The
"race on by default" Go knob becomes "overflow-checks + debug-assertions on by
default" for tests (the closest Rust analogue of catching latent bugs in CI),
with a `fast` config that drops them for quick local iteration.

```bash
# Keep MODULE.bazel.lock current automatically (matches Go).
common --lockfile_mode=update

# Platform-specific config sections (build:macos, build:linux).
common --enable_platform_specific_config

# Test output: errors only (matches Go).
test --test_output=errors

# Default ON: overflow + debug assertions (the Rust analogue of Go's race=on).
# These catch latent arithmetic/logic bugs; disable for fast local iteration.
build --@rules_rust//rust/settings:experimental_use_cc_common_link=False
build --@rules_rust//:extra_rustc_flags=-Coverflow-checks=on -Cdebug-assertions=on

# Config: fast local iteration (drop assertions, like Go's --config=fast/norace).
build:fast --@rules_rust//:extra_rustc_flags=
test:fast --@rules_rust//:extra_rustc_flags=

# Optimized release builds.
build:opt --compilation_mode=opt
build:opt --@rules_rust//:extra_rustc_flags=-Ccodegen-units=1

# libclang / native build deps for rocksdb/firewood/secp256k1/blst.
build --action_env=LIBCLANG_PATH
build --repo_env=CARGO_BAZEL_REPIN=0   # set to 1 to force crate re-pin

# macOS: use the default CommandLineTools toolchain (matches Go + CI runners).
build:macos --repo_env=DEVELOPER_DIR=/Library/Developer/CommandLineTools
build:macos --action_env=DEVELOPER_DIR=/Library/Developer/CommandLineTools
build:macos --action_env=SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX.sdk
try-import %workspace%/.bazelrc.local

# Disk cache for faster builds (matches Go).
build --disk_cache=~/.cache/bazel-disk-cache

# Release: stamp with git commit (matches Go).
build:release --stamp
build:release --workspace_status_command=scripts/bazel_workspace_status.sh
```

`.bazelversion` pins Bazel `8.0.1` (same as Go) so bazelisk fetches a consistent
launcher.

### 4.5 How `bazel build/test //...` maps to Cargo

| Bazel | Cargo equivalent | Notes |
|---|---|---|
| `bazel build //crates/ava-node:avalanchego` | `cargo build -p avalanchego --release` | `rust_binary` target generated by gazelle. |
| `bazel build //...` | `cargo build --workspace` | full hermetic build. |
| `bazel test //...` | `cargo nextest run --workspace` | each `rust_test` ↔ a `#[test]` crate; nextest is the dev runner, Bazel the hermetic runner. |
| `bazel test --config=fast //...` | `cargo nextest run` (no overflow/asserts) | mirrors `test-unit-fast`. |
| `@crates//:tokio` etc. | `[workspace.dependencies] tokio` resolved via `Cargo.lock` | crate_universe consumes `Cargo.lock`. |

Cargo remains the **inner-loop** tool (fast, incremental, IDE-integrated); Bazel
is the **hermetic/CI/cross-language** tool. Both must agree because both read
`Cargo.lock`. A `bazel-vs-cargo` smoke job in CI builds the binary both ways.

---

## 5. Task runner — `Taskfile.yml`

**Decision (cross-spec):** the **canonical runner is [Task](https://taskfile.dev)
(`go-task`)**, exactly as in the Go repo, invoked through the carried-over
`./scripts/run_task.sh`. This keeps muscle memory, the nix-wrapper behavior, and
the CI "always call a task" rule identical. A `justfile` is provided as an
optional thin convenience layer for people who prefer `just`, but it only shells
out to the same scripts; `Taskfile.yml` is authoritative and is what CI runs.

**Decision (cross-spec):** the **test runner is `cargo-nextest`** (faster, better
output, per-test isolation, JUnit for CI), with `cargo test` retained only for
doctests (nextest does not run doctests).

### 5.1 Go task → Rust task mapping

| Go task | Rust task | Underlying command |
|---|---|---|
| `build` | `build` | `cargo build -p avalanchego --release` |
| `build-race` | `build-debug-checks` | `cargo build --profile dev-checks` (overflow + debug-assertions) — Rust has no data-race detector; nextest under `--profile ci` + `-Zsanitizer` (nightly) is the closest, gated behind a `race`/`sanitizer` task |
| `test-unit` | `test-unit` | `cargo nextest run --workspace --all-features --profile ci` + `cargo test --doc` |
| `test-unit-fast` | `test-unit-fast` | `cargo nextest run --workspace` (no `--all-features`, no checks profile) |
| `lint` | `lint` | `cargo clippy --workspace --all-targets --all-features -- -D warnings` + `rustfmt --check` + license-header + `taplo fmt --check` |
| `lint-fix` | `lint-fix` | `cargo clippy --fix` + `cargo fmt` + `taplo fmt` |
| `lint-saevm` | `lint-saevm` | pedantic+overflow clippy on `ava-saevm*` (§7) |
| `generate-protobuf` | `generate-protobuf` | `buf generate` / build.rs codegen check (§8) |
| `generate-mocks` | `generate-mocks` | regenerate committed `mockall` artifacts (mostly a no-op; §8) |
| `go-mod-tidy` | `deps-tidy` | `cargo update --workspace --locked --dry-run` check + `cargo deny check` + `cargo machete` + `bazel run //:crates_vendor`/repin |
| `bazel-build` / `bazel-test` / `bazel-build-race` / `bazel-test-fast` | same names | `bazelisk build/test //...` (+ `--config=fast`) |
| `bazel-gazelle-generate` | `bazel-gazelle-generate` | `bazelisk run //:gazelle` |
| `bazel-check-metadata` | `bazel-check-metadata` | `bazelisk run //:gazelle_check` + `Cargo.lock` clean check |
| `check-generate-*` | `check-generate-*` | run generator then assert clean tree |
| `lint-all` / `lint-all-ci` | `lint-all` / `lint-all-ci` | aggregate of the above |
| `lint-shell` / `lint-action` / `check-yaml-fmt` | same | `shellcheck` / `actionlint` / `yamlfmt` (kept verbatim) |
| `test-e2e` / `test-upgrade` / `test-load` / `test-fuzz` | same names | see `02-testing-strategy.md`; fuzz = `cargo +nightly fuzz` (cargo-fuzz) / `proptest` |
| `check-go-version` | `check-rust-version` | assert `rust-toolchain.toml` == MODULE.bazel == CI matrix |

### 5.2 `Taskfile.yml`

```yaml
# https://taskfile.dev
# Run via ./scripts/run_task.sh (prefers `task` on PATH, else `go tool`/nix).
version: '3'

vars:
  # Wrap entrypoints in the nix dev shell when available (carried over from Go).
  NIX_RUN: '{{printf "%s/scripts/nix_run.sh" .TASKFILE_DIR}}'
  WORKSPACE: '--workspace'

tasks:
  default: './scripts/run_task.sh --list'

  # --- Build -------------------------------------------------------------
  build:
    desc: Build the avalanchego binary (release)
    cmd: '{{.NIX_RUN}} cargo build -p avalanchego --release'

  build-debug-checks:
    desc: Build with overflow + debug assertions (closest analogue of build-race)
    cmd: '{{.NIX_RUN}} cargo build {{.WORKSPACE}} --profile dev-checks'

  # --- Test --------------------------------------------------------------
  test-unit:
    desc: Run unit tests (all features, CI profile) + doctests
    cmds:
      - '{{.NIX_RUN}} cargo nextest run {{.WORKSPACE}} --all-features --profile ci'
      - '{{.NIX_RUN}} cargo test --doc {{.WORKSPACE}} --all-features'

  test-unit-fast:
    desc: Fast local unit tests (no all-features, no checks profile)
    cmd: '{{.NIX_RUN}} cargo nextest run {{.WORKSPACE}}'

  test-coverage:
    desc: Run tests with coverage (analogue of -coverprofile/-covermode)
    cmd: '{{.NIX_RUN}} cargo llvm-cov nextest {{.WORKSPACE}} --all-features --lcov --output-path lcov.info'

  # --- Lint / format -----------------------------------------------------
  lint:
    desc: clippy (deny warnings) + rustfmt check + TOML + license headers
    cmds:
      - '{{.NIX_RUN}} cargo clippy {{.WORKSPACE}} --all-targets --all-features -- -D warnings'
      - '{{.NIX_RUN}} cargo fmt --all -- --check'
      - '{{.NIX_RUN}} taplo fmt --check'
      - '{{.NIX_RUN}} ./scripts/check_license_headers.sh'

  lint-fix:
    desc: Auto-fix clippy + format
    cmds:
      - '{{.NIX_RUN}} cargo clippy {{.WORKSPACE}} --all-targets --all-features --fix --allow-dirty --allow-staged'
      - '{{.NIX_RUN}} cargo fmt --all'
      - '{{.NIX_RUN}} taplo fmt'
      - '{{.NIX_RUN}} ./scripts/add_license_headers.sh'

  lint-saevm:
    desc: Stricter pedantic + overflow clippy on SAE crates (analogue of lint-saevm/G115)
    cmd: '{{.NIX_RUN}} ./scripts/lint_saevm.sh'

  lint-shell:
    desc: shellcheck (carried over from Go)
    cmd: '{{.NIX_RUN}} ./scripts/shellcheck.sh'

  lint-action:
    desc: actionlint on GitHub workflows (carried over)
    cmd: '{{.NIX_RUN}} actionlint'

  check-yaml-fmt:
    desc: Assert YAML is yamlfmt-clean (carried over)
    cmds:
      - '{{.NIX_RUN}} yamlfmt .'
      - task: check-clean-branch

  lint-all:
    desc: Run all lint checks
    deps: [lint, lint-saevm, lint-shell, lint-action]
    cmds:
      - task: check-yaml-fmt
      - task: deps-tidy
      - task: check-rust-version
      - task: bazel-check-metadata

  lint-all-ci:
    desc: Run all lint checks sequentially (CI)
    cmds:
      - task: lint
      - task: lint-saevm
      - task: lint-shell
      - task: lint-action
      - task: check-yaml-fmt
      - task: check-rust-version
      - task: deps-tidy
      - task: deps-unused

  # --- Dependencies (go-mod-tidy analogue) -------------------------------
  deps-tidy:
    desc: Verify Cargo.lock is current and dependency policy passes
    cmds:
      - '{{.NIX_RUN}} cargo update --workspace --locked'   # fails if Cargo.lock stale
      - '{{.NIX_RUN}} cargo deny check'
      - '{{.NIX_RUN}} bazelisk mod tidy'

  deps-unused:
    desc: Detect unused dependencies
    cmd: '{{.NIX_RUN}} cargo machete'

  deps-audit:
    desc: RustSec advisory scan
    cmd: '{{.NIX_RUN}} cargo audit'

  # --- Codegen -----------------------------------------------------------
  generate-protobuf:
    desc: Regenerate committed proto bindings (tonic/prost) and check buf
    cmd: '{{.NIX_RUN}} ./scripts/protobuf_codegen.sh'

  generate-mocks:
    desc: Regenerate committed mocks (mockall) where applicable
    cmd: '{{.NIX_RUN}} ./scripts/generate_mocks.sh'

  check-generate-protobuf:
    desc: Assert proto bindings are current (clean tree)
    cmds:
      - task: generate-protobuf
      - task: check-clean-branch

  check-generate-mocks:
    desc: Assert mocks are current (clean tree)
    cmds:
      - task: generate-mocks
      - task: check-clean-branch

  # --- Bazel -------------------------------------------------------------
  bazel-build:
    desc: Build avalanchego with Bazel
    cmd: '{{.NIX_RUN}} bazelisk build //crates/ava-node:avalanchego'

  bazel-build-opt:
    desc: Build avalanchego with Bazel (optimized)
    cmd: '{{.NIX_RUN}} bazelisk build --config=opt //crates/ava-node:avalanchego'

  bazel-test:
    desc: Run unit tests with Bazel
    cmd: '{{.NIX_RUN}} bazelisk test //...'

  bazel-test-fast:
    desc: Run unit tests with Bazel (no extra checks)
    cmd: '{{.NIX_RUN}} bazelisk test --config=fast //...'

  bazel-gazelle-generate:
    desc: Regenerate BUILD files with gazelle_rust
    cmd: '{{.NIX_RUN}} bazelisk run //:gazelle'

  bazel-fmt:
    desc: Format BUILD.bazel files
    cmd: '{{.NIX_RUN}} buildifier -r .'

  bazel-check-metadata:
    desc: Check BUILD files + Cargo.lock are current (clean tree)
    cmds:
      - task: bazel-gazelle-generate
      - task: bazel-fmt
      - task: check-clean-branch

  # --- Checks ------------------------------------------------------------
  check-rust-version:
    desc: Assert Rust version is consistent across toolchain/Bazel/CI
    cmd: '{{.NIX_RUN}} ./scripts/check_rust_version.sh'

  check-clean-branch:
    desc: Assert the git working tree is clean
    cmd: '{{.NIX_RUN}} ./scripts/check_clean_branch.sh'

  install-nix:
    desc: Install nix with an OS-appropriate installer
    cmd: './scripts/install_nix.sh'
```

### 5.3 `justfile` (optional convenience layer)

```just
# justfile — optional thin wrapper. Taskfile.yml is canonical (what CI runs).
build:        ; ./scripts/run_task.sh build
test:         ; ./scripts/run_task.sh test-unit
test-fast:    ; ./scripts/run_task.sh test-unit-fast
lint:         ; ./scripts/run_task.sh lint
lint-fix:     ; ./scripts/run_task.sh lint-fix
lint-saevm:   ; ./scripts/run_task.sh lint-saevm
lint-all:     ; ./scripts/run_task.sh lint-all
bazel-build:  ; ./scripts/run_task.sh bazel-build
bazel-test:   ; ./scripts/run_task.sh bazel-test
```

---

## 6. Cargo workspace profiles & the `dev-checks` profile

Add to the root `Cargo.toml` (alongside `[workspace]` / `[workspace.dependencies]`
from `00` §3/§4):

```toml
[profile.dev]
# Local iteration: fast compile, panics on overflow (catch bugs early).
overflow-checks = true
debug-assertions = true

[profile.dev-checks]            # build-race analogue: max runtime checking
inherits = "dev"
overflow-checks = true
debug-assertions = true

[profile.ci]                    # nextest CI profile uses this for the test binary
inherits = "dev"
overflow-checks = true
debug-assertions = true

[profile.release]
overflow-checks = false         # except SAE crates — see §7
lto = "thin"
codegen-units = 1
panic = "unwind"                # node catches/handles panics per-chain like Go recover()
```

`.config/nextest.toml` (the test-runner config; mirrors Go's `-timeout=120s`,
shuffle, and per-category timeouts in `.bazelrc`):

```toml
[profile.default]
retries = 0
# 120s slow-test warning + hard timeout mirrors Go's -timeout=120s.
slow-timeout = { period = "60s", terminate-after = 2 }
failure-output = "immediate-final"
fail-fast = false

[profile.ci]
retries = 1                     # tolerate flakes in CI only
slow-timeout = { period = "120s", terminate-after = 3 }
final-status-level = "fail"
junit = { path = "target/nextest/ci/junit.xml" }

# Long-running suites (reexec, differential) get a longer leash.
[[profile.ci.overrides]]
filter = "package(ava-saevm) or test(/differential_/)"
slow-timeout = { period = "900s", terminate-after = 4 }
```

---

## 7. Lint & format configuration

### 7.1 `rustfmt.toml`

Checked-in, matching `00` §8. Indentation is spaces (rustfmt does not support
tabs meaningfully for Rust; `.editorconfig` already sets `[*.rs]` to spaces).

```toml
# rustfmt.toml
edition = "2021"
max_width = 100
use_small_heuristics = "Default"
imports_granularity = "Module"     # group `use` per module path
group_imports = "StdExternalCrate" # std → external → crate (mirrors Go's gci order)
reorder_imports = true
reorder_modules = true
newline_style = "Unix"             # LF (mirrors .editorconfig)
format_code_in_doc_comments = true
normalize_comments = true
```

> `group_imports = "StdExternalCrate"` is the analogue of the Go `gci` ordering
> (standard → third-party → local). The blank/alias/dot subtleties of Go's gci
> have no Rust equivalent and are dropped.

### 7.2 `.editorconfig`

Extend the existing file with a Rust stanza (Go used tabs for `.go`; Rust uses
spaces):

```ini
# https://editorconfig.org/
root = true

[*]
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
indent_style = space
indent_size = 2

[*.rs]
indent_size = 4          # rustfmt default

[*.{toml}]
indent_size = 4
```

### 7.3 Clippy configuration

Workspace-wide lints live in the root `Cargo.toml` `[workspace.lints]` so every
crate inherits them (each crate sets `[lints] workspace = true`):

```toml
[workspace.lints.rust]
unsafe_code = "forbid"          # opt out per-crate for blst/firewood/rocksdb/secp256k1 (00 §7.6)
missing_docs = "warn"           # library crates (00 §8)
unused_crate_dependencies = "warn"

[workspace.lints.clippy]
all = { level = "deny", priority = -1 }
# Project bans mirroring Go's forbidigo/depguard (see §7.5):
unwrap_used = "deny"            # no unwrap() in non-test lib code (00 §8)
expect_used = "warn"
indexing_slicing = "warn"
arithmetic_side_effects = "warn"  # nudge toward checked arithmetic (00 §6.1)
todo = "warn"
dbg_macro = "deny"
```

`clippy.toml` for shared knobs:

```toml
# clippy.toml
avoid-breaking-exported-api = false
allow-unwrap-in-tests = true
allow-expect-in-tests = true
allow-dbg-in-tests = true
```

### 7.4 SAE strict pass — `lint-saevm`

The Go `lint-saevm` runs `gosec` with **G115** (integer-overflow on type
conversion) over `vms/saevm/...`. The Rust analogue holds the `ava-saevm*` crates
to a higher bar (`00` §7.7):

- `#![deny(clippy::pedantic, clippy::arithmetic_side_effects, clippy::cast_possible_truncation, clippy::cast_sign_loss, clippy::cast_possible_wrap)]`
  in each SAE crate's `lib.rs` (the `cast_*` lints are the direct G115 analogue),
- `overflow-checks = true` in **all** profiles (including release) for SAE crates,
  via a per-crate `[profile.release] overflow-checks = true` override is not
  possible in Cargo (profiles are workspace-global), so instead SAE crates gate
  overflow behavior with `#![deny(clippy::arithmetic_side_effects)]` and use
  explicit `checked_*`/`saturating_*` everywhere; the `lint-saevm` task asserts.

`scripts/lint_saevm.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
# Stricter clippy pass for SAE crates — analogue of `gosec -include=G115`.
SAE_CRATES=$(cargo metadata --no-deps --format-version 1 \
  | jq -r '.packages[].name | select(startswith("ava-saevm"))')
for crate in $SAE_CRATES; do
  cargo clippy -p "$crate" --all-targets --all-features -- \
    -D warnings \
    -W clippy::pedantic \
    -D clippy::arithmetic_side_effects \
    -D clippy::cast_possible_truncation \
    -D clippy::cast_sign_loss \
    -D clippy::cast_possible_wrap
done
```

### 7.5 Forbidden patterns (mirroring `forbidigo`/`depguard`)

Enforced by the clippy lints above plus `cargo deny` (§9). Mirroring Go's
deny-list:

| Go forbidden | Rust equivalent | Mechanism |
|---|---|---|
| `container/list` → `utils/linked` | use `std`/`indexmap`/`ava-utils` collections | code review + `deny.toml` bans |
| `golang/mock` → `go.uber.org/mock` | `mockall` only (`00` §4.7) | `deny.toml` bans alternatives |
| `io/ioutil` | `std::fs`/`std::io` | n/a (no such crate) |
| `require.Error` → `ErrorIs` | assert on typed error variants (`assert_matches!`) | review + `02` |
| `sort.Slice` → `slices` | `slice::sort*`/`sort_unstable*` | n/a |
| bare `time.Add(TauSeconds)` (tausecondslint) | a grep CI check forbidding raw `as` casts on Tau and unchecked `Duration` math in SAE | `scripts/tau_lint.sh` (see §10) |
| `fmt.Errorf` w/o verb → `errors.New` | `thiserror` variants, not ad-hoc strings | review |

### 7.6 License headers (mirroring `header.yml` + `go-license`)

Every `.rs` file carries the header from `00` §8. Generated files
(`*.pb.rs`, `mock_*.rs`) are exempt, mirroring the Go exemptions for `*.pb.go` /
`mock_*.go`. Enforced by a small script (analogue of `go-license --verify`):

```bash
#!/usr/bin/env bash
# scripts/check_license_headers.sh — analogue of `go-license --verify`.
set -euo pipefail
HEADER='// Copyright (C) 2019, Ava Labs, Inc. All rights reserved.'
missing=0
while IFS= read -r f; do
  case "$f" in
    *.pb.rs|*mock_*.rs|*/target/*) continue ;;
  esac
  if ! head -n1 "$f" | grep -qF "$HEADER"; then
    echo "missing license header: $f"; missing=1
  fi
done < <(git ls-files '*.rs')
exit "$missing"
```

(`scripts/add_license_headers.sh` is the non-`--verify` counterpart that prepends
the two-line header to any file missing it.)

---

## 8. Codegen story

### 8.1 Protobuf / gRPC (mirrors `generate-protobuf`)

Proto sources live in `proto/` (shared with Go, `00` §3). We use `prost` +
`tonic` (`00` §4.2). Two coexisting paths, same as Go's `buf` vs Bazel:

- **Cargo path (inner loop):** each crate that needs generated types
  (`ava-message`, `ava-vm-rpc`, …) has a `build.rs` calling `tonic_build` /
  `prost_build`, generating into `OUT_DIR`. **Generated proto code is NOT
  committed** in the Cargo path — it is produced at build time, which is the
  idiomatic Rust approach and avoids dirty-tree churn.
- **Bazel path (hermetic):** `rust_prost_library` targets (toolchain wired in
  `MODULE.bazel` §4.2) generate the same code hermetically with `protoc` from
  `toolchains_protoc`.
- **`buf` lint/breaking** is kept from the Go flake and run in `generate-protobuf`
  to guard wire compatibility (`00` §1 compatibility surface).

> **Decision (cross-spec):** proto bindings are **generated via `build.rs`
> (tonic/prost), not committed.** Therefore `check-generate-protobuf` for the
> Cargo path reduces to "buf lint + buf breaking + the build compiles"; the
> dirty-tree check applies to the **Bazel-generated** artifacts only if any are
> checked in (none are by default). `02-testing-strategy.md` and the RPC specs
> (`05`, `07`, `12`) must assume `OUT_DIR`-generated proto types reachable via
> `include!(concat!(env!("OUT_DIR"), "/<pkg>.rs"))` or `tonic::include_proto!`.

`scripts/protobuf_codegen.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
buf lint
buf breaking --against '.git#branch=master'   # guards wire compatibility
# Compile every crate that runs proto build.rs (codegen happens during build).
cargo check --workspace --all-features
```

### 8.2 Mocks (mirrors `generate-mocks`)

`mockall` generates mocks **at compile time** via `#[cfg_attr(test, automock)]`
on traits (`00` §4.7, §6). The idiomatic approach commits **no** generated mock
files — they are produced inline by the macro. So `generate-mocks` /
`check-generate-mocks` are **essentially no-ops** (the macro expansion is checked
by `cargo check --all-features`). We keep the task names for parity and to host
any rare `mockall_double`-style committed mock if one is ever needed.

```bash
#!/usr/bin/env bash
# scripts/generate_mocks.sh — mockall mocks are macro-generated; verify they compile.
set -euo pipefail
cargo check --workspace --all-features --tests
```

### 8.3 Canoto / contract bindings

- **Canoto** (Go's `generate-canoto`) has no analogue — the linear codec is
  hand-written in `ava-codec` with a derive macro (`ava-codec-derive`), so there
  is nothing external to regenerate.
- **Load contract bindings** (`generate-load-contract-bindings`): EVM test
  contracts compile via `solc` (kept in the flake) and bind via `alloy`'s
  `sol!` macro at compile time — again macro-generated, not committed. Covered in
  `10-cchain-evm-reth.md` / `02-testing-strategy.md`.

> **Committed vs build.rs summary:** Rust strongly prefers build-time/macro
> generation over committed artifacts. We commit **nothing** generated by
> default; the dirty-tree CI gates therefore primarily guard `Cargo.lock`,
> `MODULE.bazel.lock`, and gazelle-generated `BUILD.bazel`.

---

## 9. Dependency policy — `deny.toml` (mirrors `go-mod-tidy` + license check)

`cargo-deny` is the analogue of Go's `go mod tidy` consistency + license header
checks at the dependency level: it enforces a single version per crate (Go's
`check-require-directives`), an allow-list of licenses, banned crates (Go's
`depguard`), and advisory scanning.

```toml
# deny.toml
[graph]
all-features = true

[advisories]
db-path = "~/.cargo/advisory-db"
db-urls = ["https://github.com/rustsec/advisory-db"]
yanked = "deny"
ignore = []                      # add RUSTSEC-XXXX with justification + expiry

[licenses]
# Match avalanchego's permissive posture; the repo itself is BSD-licensed.
allow = [
    "Apache-2.0",
    "MIT",
    "BSD-2-Clause",
    "BSD-3-Clause",
    "ISC",
    "Unicode-3.0",
    "Zlib",
    "MPL-2.0",          # used by some crypto deps; review case-by-case
]
confidence-threshold = 0.9
exceptions = [
    # { allow = ["..."], name = "some-crate" },
]

[bans]
multiple-versions = "warn"       # tighten to "deny" once the tree stabilizes
wildcards = "deny"               # no `*` version requirements (mirrors pinned go.mod)
deny = [
    # Enforce the single-crate-per-job rule from 00 §4 and §7.5:
    { name = "openssl", wrappers = ["native-tls"] },  # prefer rustls (00 §4.3)
    # { name = "<banned alt mock crate>" },           # mockall is the only mock lib
]
skip = []
skip-tree = []

[sources]
unknown-registry = "deny"
unknown-git = "deny"
allow-git = [
    # Pinned git deps (reth/firewood may be git until crates.io releases land).
    "https://github.com/paradigmxyz/reth",
    "https://github.com/ava-labs/firewood",
]
```

`deps-tidy` runs `cargo update --locked` (fails if `Cargo.lock` is stale — the
analogue of `go mod tidy` leaving the tree dirty), then `cargo deny check`, then
`bazelisk mod tidy` (keep `MODULE.bazel.lock` synced). `deps-unused`
(`cargo machete`) catches dependencies declared but unused.

---

## 10. CI — `.github/workflows/ci.yml`

Mirrors the Go CI gates: a `tests-required` aggregator that fails unless all
needed jobs pass, a `Unit` matrix across OSes/arches, `Lint` (= `lint-all-ci`),
dirty-tree codegen checks, the SAE/tau lint, and Bazel build/test. e2e / upgrade
/ load / fuzz / differential jobs are defined in `02-testing-strategy.md` and
referenced here. Every job calls a **task**, never `cargo`/`bazel` directly
(the Go invariant).

```yaml
name: Tests

on:
  push:
    tags: ["*"]
    branches: [master, dev]
  pull_request:
  merge_group:
    types: [checks_requested]

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  tests-required:
    if: ${{ always() }}
    needs:
      - Unit
      - Lint
      - taulint
      - check_generated_protobuf
      - check_mocks
      - check_deps_tidy
      - bazel
      - differential
    runs-on: ubuntu-24.04
    steps:
      - name: Fail unless all needed jobs succeeded
        env:
          NEEDS_JSON: ${{ toJSON(needs) }}
        run: |
          failed="$(jq -r 'to_entries[] | select(.value.result != "success") | .key' <<<"$NEEDS_JSON")"
          if [[ -n "$failed" ]]; then echo "Failed: $failed"; exit 1; fi

  Unit:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [macos-26, ubuntu-22.04, ubuntu-24.04, ubuntu-24.04-arm]
    steps:
      - uses: actions/checkout@v5
      - uses: ./.github/actions/install-nix      # provides pinned toolchain via flake
      - name: test-unit
        shell: nix develop --command bash -x {0}
        run: ./scripts/run_task.sh test-unit

  Lint:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v5
      - uses: ./.github/actions/install-nix
      - name: lint-all-ci
        shell: nix develop --command bash -x {0}
        run: ./scripts/run_task.sh lint-all-ci

  taulint:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v5
      - name: Forbid raw casts/unchecked Duration math on Tau (analogue of tausecondslint)
        run: ./scripts/tau_lint.sh

  check_generated_protobuf:
    name: Up-to-date protobuf
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v5
      - uses: ./.github/actions/install-nix
      - shell: nix develop --command bash -x {0}
        run: ./scripts/run_task.sh check-generate-protobuf

  check_mocks:
    name: Mocks compile
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v5
      - uses: ./.github/actions/install-nix
      - shell: nix develop --command bash -x {0}
        run: ./scripts/run_task.sh check-generate-mocks

  check_deps_tidy:
    name: Cargo.lock + dependency policy
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v5
      - uses: ./.github/actions/install-nix
      - shell: nix develop --command bash -x {0}
        run: ./scripts/run_task.sh deps-tidy

  bazel:
    name: Bazel build + test
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v5
      - uses: ./.github/actions/install-nix
      - name: bazel build + test + metadata check
        shell: nix develop --command bash -x {0}
        run: |
          ./scripts/run_task.sh bazel-check-metadata
          ./scripts/run_task.sh bazel-build
          ./scripts/run_task.sh bazel-test

  differential:
    name: Differential tests vs Go node
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v5
      - uses: ./.github/actions/install-nix
      # Defined in 02-testing-strategy.md (golden vectors + cross-impl harness).
      - shell: nix develop --command bash -x {0}
        run: ./scripts/run_task.sh test-differential
```

`.github/actions/install-nix` is carried over from Go (installs Nix and warms the
flake). The `tau_lint.sh` script (the analogue of the Go `tausecondslint` grep
gate) forbids raw `as` casts and unchecked `Duration` arithmetic on `Tau`
quantities in SAE code:

```bash
#!/usr/bin/env bash
# scripts/tau_lint.sh — analogue of avalanchego's tausecondslint.
set -euo pipefail
if grep -rnE '\bTauSeconds\b.*\bas\b|Instant::now\(\)\s*\+\s*[^;]*TauSeconds' \
     --include='*.rs' crates/ava-saevm*; then
  echo "ERROR: use a typed Duration (params::TAU), never raw casts on TauSeconds"
  exit 1
fi
```

> e2e / upgrade / load / fuzz / robustness jobs (the long tail of Go CI) are
> defined in `02-testing-strategy.md`; this file owns only the build/lint/codegen
> gates. They are added to `tests-required` there.

---

## 11. Repo layout recap (build files)

Defer to `00` §3 for the crate tree. The build-tooling files at the repo root:

```
avalanche-rs/
├── Cargo.toml                 # [workspace] members, [workspace.dependencies],
│                              # [workspace.lints], [workspace.package], [profile.*]
├── Cargo.lock                 # committed; source of truth for crate_universe
├── rust-toolchain.toml        # pinned stable toolchain (§2)
├── rustfmt.toml               # §7.1
├── clippy.toml                # §7.3
├── deny.toml                  # §9
├── .config/nextest.toml       # §6
├── flake.nix / flake.lock     # §3
├── .envrc                     # §3.2
├── MODULE.bazel               # §4.2 (bzlmod: rules_rust + crate_universe + gazelle)
├── MODULE.bazel.lock          # committed; kept in sync via `bazelisk mod tidy`
├── BUILD.bazel                # §4.3 (gazelle binary, root targets)
├── .bazelrc / .bazelrc.local  # §4.4
├── .bazelversion              # `8.0.1`
├── .editorconfig              # §7.2
├── header.yml                 # license header text (shared with check scripts)
├── Taskfile.yml               # §5.2 (canonical runner)
├── justfile                   # §5.3 (optional convenience)
├── AGENTS.md / CLAUDE.md      # §12
├── .github/workflows/ci.yml   # §10
├── scripts/                   # run_task.sh, nix_run.sh, lint/codegen helpers
│   ├── run_task.sh            # carried over from Go (task launcher)
│   ├── nix_run.sh             # carried over (nix develop wrapper)
│   ├── check_license_headers.sh
│   ├── add_license_headers.sh
│   ├── lint_saevm.sh
│   ├── tau_lint.sh
│   ├── protobuf_codegen.sh
│   ├── generate_mocks.sh
│   ├── check_rust_version.sh
│   └── check_clean_branch.sh
├── crates/                    # all ava-* crates (00 §3)
├── proto/                     # .proto sources shared with Go
└── tests/                     # cross-crate + differential harness (02)
```

---

## 12. `AGENTS.md` and `CLAUDE.md` (verbatim, copy-pasteable)

Both files mirror the Go `CLAUDE.md` in the system context. `AGENTS.md` is the
tool-neutral version; `CLAUDE.md` adds Claude-Code-specific phrasing. Copy these
into the repo root verbatim.

### 12.1 `AGENTS.md`

````markdown
# AGENTS.md

Guidance for AI coding agents working in **avalanche-rs** — a from-scratch Rust
implementation of an Avalanche node, a drop-in replacement for `avalanchego`.
Read this before making changes; it captures how to build, test, lint, and
conform to repo conventions so your work passes CI on the first try.

## Repo at a glance

- **Workspace:** a single Cargo workspace; crates live under `crates/`, all named
  `ava-*` (the binary is `avalanchego` for drop-in invocation).
- **Rust version:** pinned exactly in `rust-toolchain.toml` (e.g. `1.90.0`). Bump
  it in lock-step with `MODULE.bazel` and the CI matrix; `check-rust-version`
  asserts they agree.
- **Goal:** byte-for-byte wire/codec/API/genesis compatibility with `avalanchego`.
  EVM execution is built on **reth**; the Merkle state DB is **Firewood** (direct
  Rust dep, no FFI shim).
- **Build tooling:** [Task](https://taskfile.dev) (`Taskfile.yml`) is the
  canonical runner; Nix (`flake.nix`) pins the toolchain; Bazel (`bazelisk`,
  bzlmod + rules_rust + crate_universe + gazelle_rust) is the hermetic path.
- **Test runner:** `cargo-nextest` (doctests via `cargo test --doc`).

### Stricter bar for SAE (`crates/ava-saevm*`)

`ava-saevm*` implements **SAE (Streaming Asynchronous Execution, ACP-194)**.
No API-stability guarantees. It has a **dedicated stricter lint pass**
(`lint-saevm`: clippy pedantic + `arithmetic_side_effects` + `cast_*` deny,
overflow checks everywhere) — the analogue of avalanchego's `gosec G115`. Hold
this code to a higher bar; use `checked_*`/`saturating_*`, never raw casts.

## Running tasks

Everything goes through the Task runner. You do **not** need `task` installed —
use the bootstrap wrapper (it uses `task` from PATH, else `go tool`, and wraps
tasks in the Nix dev shell when Nix is present):

```sh
./scripts/run_task.sh <task-name>     # primary entrypoint
./scripts/run_task.sh --list          # list tasks
```

CI does **not** call `cargo`/`bazel` directly — it always goes through tasks.

## Build

```sh
./scripts/run_task.sh build               # cargo build -p avalanchego --release
./scripts/run_task.sh build-debug-checks  # overflow + debug assertions
./scripts/run_task.sh bazel-build         # hermetic Bazel build
```

## Test

```sh
./scripts/run_task.sh test-unit       # nextest --all-features --profile ci + doctests
./scripts/run_task.sh test-unit-fast  # nextest, no all-features/checks (fast)
./scripts/run_task.sh test-coverage   # cargo llvm-cov
```

Single crate / test:

```sh
cargo nextest run -p ava-codec
cargo nextest run -p ava-snow -E 'test(TestName)'
```

## Lint & format

```sh
./scripts/run_task.sh lint          # clippy -D warnings + rustfmt --check + TOML + license
./scripts/run_task.sh lint-fix      # clippy --fix + cargo fmt + taplo fmt + add headers
./scripts/run_task.sh lint-saevm    # stricter pedantic/overflow pass on ava-saevm*
./scripts/run_task.sh lint-all      # everything CI's lint-all-ci runs
```

`rustfmt` (config in `rustfmt.toml`) handles all formatting — there is no separate
fmt-only gate beyond `cargo fmt --check`.

## Code generation

Rust prefers build-time/macro generation; we commit **no** generated code.

| What | Command | Notes |
|------|---------|-------|
| Protobuf/gRPC | `./scripts/run_task.sh generate-protobuf` | `build.rs` via tonic/prost; runs `buf lint`+`buf breaking` |
| Mocks | `./scripts/run_task.sh generate-mocks` | `mockall` is macro-generated; this just checks it compiles |
| Deps tidy | `./scripts/run_task.sh deps-tidy` | `cargo update --locked` + `cargo deny check` + `bazelisk mod tidy` |
| Bazel BUILD | `./scripts/run_task.sh bazel-gazelle-generate` | `gazelle_rust`; commit the result |

## Before you push — pass CI

1. `./scripts/run_task.sh lint-all` (clippy, rustfmt, license, SAE, shell,
   actionlint, yaml, deps, rust-version, bazel-metadata).
2. `./scripts/run_task.sh test-unit`.
3. If you touched `proto/`: `./scripts/run_task.sh check-generate-protobuf`.
4. If you changed dependencies: `./scripts/run_task.sh deps-tidy` and commit
   `Cargo.lock` + `MODULE.bazel.lock`.
5. If you touched `.rs` files affecting Bazel: `./scripts/run_task.sh
   bazel-check-metadata` and commit regenerated `BUILD.bazel`.
6. **Never** apply a raw `as` cast or unchecked `Duration` math to a `Tau`
   quantity — the `taulint` CI gate (`scripts/tau_lint.sh`) fails on it. Use
   `params::TAU` (a typed `Duration`) and `checked_*`.

## Conventions (see `00-overview-and-conventions.md` for the full set)

- **Edition 2021**, tabs are *not* used for Rust (`.editorconfig` → 4 spaces).
- **License header** on every `.rs` file:
  ```rust
  // Copyright (C) 2019, Ava Labs, Inc. All rights reserved.
  // See the file LICENSE for licensing terms.
  ```
- **Errors:** per-crate `thiserror` `Error` enum + `pub type Result<T>`; `anyhow`
  only in the `avalanchego` binary and tests. Preserve Go sentinel errors as
  variants; assert with `assert_matches!` (mirrors Go's `ErrorIs` rule).
- **Imports:** grouped std → external → crate (`group_imports = "StdExternalCrate"`).
- **No `unwrap()`/`expect()`** in non-test library code (clippy denies it) except
  with a documented proven invariant.
- **`#![forbid(unsafe_code)]`** by default; opt out only in FFI wrapper modules
  (blst, firewood, rocksdb, secp256k1) with a `// SAFETY:` rationale.
- **Determinism:** never serialize a `HashMap` directly — sort keys / use
  `BTreeMap` exactly where Go sorts. Checked arithmetic, no float in
  consensus/codec paths.
- **Tests:** `cargo-nextest`; table tests via arrays + `for`/`rstest`; property
  tests via `proptest`; mocks via `mockall` (narrow, local, `#[cfg_attr(test, automock)]`).

## Forbidden patterns

- No `unwrap()`/`expect()`/`dbg!`/`todo!` in library code (clippy-denied).
- No direct `HashMap` serialization in consensus/codec paths.
- No second crate for a job already covered by `00` §4 (`cargo deny` bans
  alternatives; `mockall` is the only mock lib; `rustls` over `native-tls`).
- No raw `as` casts in `ava-saevm*` (use `checked_*`/`try_into()`).
- No floating-point in codec/consensus.

## Key files

`Cargo.toml` · `rust-toolchain.toml` · `Taskfile.yml` · `scripts/run_task.sh` ·
`flake.nix` · `MODULE.bazel` · `.bazelrc` · `rustfmt.toml` · `clippy.toml` ·
`deny.toml` · `.config/nextest.toml` · `.github/workflows/ci.yml` ·
`specs-rust/00-overview-and-conventions.md`
````

### 12.2 `CLAUDE.md`

````markdown
# CLAUDE.md

Guidance for Claude Code working in the **avalanche-rs** repository — the Rust
implementation of an Avalanche node (a drop-in replacement for `avalanchego`).
Read this before making changes; it captures how to build, test, lint, and
conform to conventions so your work passes CI on the first try.

> The canonical, tool-neutral version of this guidance is `AGENTS.md`. This file
> is identical in substance with Claude-Code-specific notes. When they diverge,
> `AGENTS.md` + `specs-rust/00-overview-and-conventions.md` win.

## Repo at a glance

- **Module:** a single Cargo workspace; `ava-*` crates under `crates/`; the binary
  is `avalanchego`.
- **Rust version:** pinned exactly in `rust-toolchain.toml`. CGO-equivalent FFI
  (rocksdb, firewood, blst, secp256k1) needs `clang`/`libclang` — provided by the
  Nix dev shell.
- **Multi-tool build:** Cargo (inner loop), Bazel (bzlmod + rules_rust +
  crate_universe + gazelle_rust, hermetic/CI), Nix (pinned toolchain), Task
  (runner). Cargo is the source of truth; `crate_universe` consumes `Cargo.lock`.
- **EVM = reth; state DB = Firewood (direct dep).** The grafted Go EVM forks
  (`coreth`/`evm`/`subnet-evm`) are *reference inputs*, not transliteration
  targets.

### Active stricter area: `crates/ava-saevm*`

SAE (Streaming Asynchronous Execution, ACP-194). No API-stability guarantees.
Dedicated stricter lint pass `lint-saevm` (clippy pedantic + `arithmetic_side_effects`
+ `cast_*` deny, overflow checks). Use `checked_*`/`saturating_*`, never raw casts.
See `specs-rust/11-saevm.md` and `00` §7.7.

## Directory map (crates under `crates/`)

| Crate | Purpose |
|-------|---------|
| `ava-types` | ids, fixed byte arrays, primitive newtypes, errors |
| `ava-codec` (+ `ava-codec-derive`) | hand-written linear codec (byte-exact) |
| `ava-crypto` | secp256k1, BLS (blst), hashing, TLS/staking certs |
| `ava-utils` | set, bag, sampler, math, timers, windows |
| `ava-version` | versions + network upgrade schedule |
| `ava-database`, `ava-merkledb`, `ava-blockdb`, `ava-archivedb` | storage (rocksdb, Firewood) |
| `ava-message`, `ava-network` | P2P wire + networking |
| `ava-snow`, `ava-engine`, `ava-validators`, `ava-proposervm`, `ava-simplex` | consensus |
| `ava-vm`, `ava-vm-rpc`, `ava-secp256k1fx` | VM framework + rpcchainvm plugin |
| `ava-platformvm`, `ava-avm`, `ava-evm`, `ava-saevm*` | the VMs |
| `ava-chains`, `ava-api`, `ava-indexer`, `ava-wallet`, `ava-genesis`, `ava-config` | node services |
| `ava-node`, `avalanchego` | node assembly + binary |

## Running tasks

```sh
./scripts/run_task.sh <task>     # primary entrypoint (wraps Nix dev shell)
./scripts/run_task.sh --list     # list tasks
```

CI always calls a task, never `cargo`/`bazel` directly.

## Build / Test / Lint

```sh
./scripts/run_task.sh build               # release binary
./scripts/run_task.sh test-unit           # nextest --all-features --profile ci + doctests
./scripts/run_task.sh test-unit-fast      # fast local nextest
./scripts/run_task.sh lint                # clippy -D warnings + rustfmt + license + TOML
./scripts/run_task.sh lint-fix            # auto-fix
./scripts/run_task.sh lint-saevm          # stricter SAE pass
./scripts/run_task.sh lint-all            # everything lint-all-ci runs
```

Single test: `cargo nextest run -p <crate> -E 'test(Name)'`.

## Code generation (commit nothing generated by default)

| What | Command |
|------|---------|
| Protobuf | `./scripts/run_task.sh generate-protobuf` (build.rs tonic/prost; buf lint+breaking) |
| Mocks | `./scripts/run_task.sh generate-mocks` (mockall macros; compile-check only) |
| Deps | `./scripts/run_task.sh deps-tidy` (cargo update --locked + cargo deny + bazelisk mod tidy) |
| Bazel BUILD | `./scripts/run_task.sh bazel-gazelle-generate` (commit results) |

## Before you push — pass CI

1. `./scripts/run_task.sh lint-all`
2. `./scripts/run_task.sh test-unit`
3. Touched `proto/`? `check-generate-protobuf`.
4. Changed deps? `deps-tidy`; commit `Cargo.lock` + `MODULE.bazel.lock`.
5. Touched `.rs` affecting Bazel? `bazel-check-metadata`; commit `BUILD.bazel`.
6. **Never** raw-cast or do unchecked `Duration` math on a `Tau` quantity — the
   `taulint` gate fails on it. Use `params::TAU` + `checked_*`.

## Rust coding conventions (enforced)

- **License header** on every `.rs`:
  ```rust
  // Copyright (C) 2019, Ava Labs, Inc. All rights reserved.
  // See the file LICENSE for licensing terms.
  ```
- **4-space** indent for `.rs` (`.editorconfig`), LF endings, final newline.
- **Import grouping** std → external → crate (`group_imports = "StdExternalCrate"`).
- **Errors:** `thiserror` per-crate enum + `Result<T>`; `anyhow` only in the
  binary/tests. Sentinel errors → variants; assert via `assert_matches!`.
- **No `unwrap()`/`expect()`/`dbg!`/`todo!`** in library code (clippy denies).
- **`#![forbid(unsafe_code)]`** except FFI wrappers with `// SAFETY:` + tests.
- Lints on: `clippy::all` (deny), `unwrap_used` (deny), `arithmetic_side_effects`
  (warn; deny in SAE), `missing_docs` (warn on libs), `unused_crate_dependencies`.

### Forbidden patterns

- No direct `HashMap` serialization in consensus/codec — sort keys / `BTreeMap`.
- No floating-point in codec/consensus paths.
- No second crate for a job covered by `00` §4 (`cargo deny` enforces;
  `mockall` only; `rustls` over `native-tls`).
- No raw `as` casts in `ava-saevm*`.

## Testing conventions

- **`cargo-nextest`** is the runner; doctests via `cargo test --doc`.
- Use **`proptest`** for property tests; **`mockall`** for narrow local mocks
  (`#[cfg_attr(test, automock)]`).
- Table tests via arrays/`rstest`; assertions via `assert_matches!` /
  `pretty_assertions`.
- Differential tests against the Go node guard protocol parity — see
  `specs-rust/02-testing-strategy.md`.

## Key files

`Cargo.toml` · `rust-toolchain.toml` · `Taskfile.yml` · `scripts/run_task.sh` ·
`flake.nix` · `MODULE.bazel` · `.bazelrc` · `rustfmt.toml` · `clippy.toml` ·
`deny.toml` · `.config/nextest.toml` · `.github/workflows/ci.yml` ·
`specs-rust/00-overview-and-conventions.md`
````

---

## 13. Cross-spec decisions recorded here (other authors must honor)

1. **Task runner = [Task](https://taskfile.dev) (`go-task`)** via
   `./scripts/run_task.sh`, identical to Go. Optional `justfile` only wraps it.
2. **Test runner = `cargo-nextest`** (`--profile ci`), doctests via
   `cargo test --doc`. All test specs target nextest filter syntax (`-E 'test(...)'`).
3. **Proto codegen = `tonic`/`prost` via `build.rs`, NOT committed** (Bazel path
   uses `rust_prost_library`). Specs `05`/`07`/`12` must access generated types
   via `OUT_DIR`/`tonic::include_proto!`, not from checked-in `*.pb.rs`.
4. **Mocks = `mockall`, macro-generated, NOT committed** (`generate-mocks` is a
   compile-check).
5. **Dependency policy = `cargo-deny` (`deny.toml`)**; single source of versions
   is `Cargo.lock`, consumed by Bazel `crate_universe`. One crate per job (`00`
   §4); `cargo deny` enforces the bans.
6. **Toolchain pinned in `rust-toolchain.toml`** (exact stable patch), read by
   Nix (`rust-overlay`) and mirrored in `MODULE.bazel`; bump in lock-step with
   reth's MSRV. `check-rust-version` gates consistency.
7. **Nix = `oxalica/rust-overlay`** (not fenix), reading `rust-toolchain.toml`
   directly; supported systems = the four in the Go flake.
8. **Bazel = bzlmod + `rules_rust` 0.70 + `crate_universe` + `gazelle_rust`**;
   Cargo is source of truth, `.bazelversion` = `8.0.1`.
9. **SAE strictness:** `ava-saevm*` crates carry pedantic + `arithmetic_side_effects`
   + `cast_*` deny and use only checked arithmetic — every SAE spec must comply.
10. **No committed generated artifacts**; dirty-tree CI gates guard `Cargo.lock`,
    `MODULE.bazel.lock`, and gazelle `BUILD.bazel` only.

---

## Sources

- rules_rust releases / crate_universe bzlmod:
  <https://github.com/bazelbuild/rules_rust/releases>,
  <https://bazelbuild.github.io/rules_rust/crate_universe_bzlmod.html>
- gazelle_rust: <https://github.com/Calsign/gazelle_rust>
- rules_rust prost/tonic: <https://bazelbuild.github.io/rules_rust/rust_prost.html>
- oxalica/rust-overlay: <https://github.com/oxalica/rust-overlay>
- cargo-nextest config: <https://nexte.st/docs/configuration/>
- cargo-deny config: <https://embarkstudios.github.io/cargo-deny/checks/cfg.html>
