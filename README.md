# nix-507531-x86-verify

Out-of-tree GitHub Actions harness that verifies the `x86_64-darwin`
code paths of [NixOS/nix#15638](https://github.com/NixOS/nix/pull/15638)
(the darwin Mach-O page-hash fixup for `nixpkgs#507531`) on a native
Intel macOS runner. Exists because the PR author has no native x86_64
hardware and Rosetta 2 on arm64 hosts bypasses the page-hash enforcement
the PR is supposed to restore.

## What it runs

Two independent jobs on `macos-13` (native Intel x86_64):

1. **`functional-tests`** — builds `nix#checks.x86_64-darwin.nix-functional-tests`
   from [ak2k/nix@fat-macho-wip](https://github.com/ak2k/nix/tree/fat-macho-wip).
   The in-tree `tests/functional/macho-rewrite.sh` test synthesizes the full
   fixture matrix (thin, fat32 1-arch, fat32 multi-arch, fat64,
   self-reference, codesign-signed, symlink-skip, malformed-fat-header
   adversarial) using the host's `/usr/bin/cc` + `/usr/bin/lipo`, runs the
   patched daemon over the trigger derivations, and asserts
   `nix-build --check` reports 0 page-hash mismatches.

2. **`reproduce-bug-on-real-binary`** — substitutes one of the known-failing
   x86_64 binaries from `cache.nixos.org`
   (`filen-cli-0.0.36/bin/.filen-wrapped`), confirms `codesign --verify`
   rejects it pre-fix, and attempts to execute it to observe the kernel's
   `cs_invalid_page` SIGKILL. This is independent of the patched daemon —
   it just demonstrates that the bug affects native x86_64 macOS.

Both jobs run from a cold cache each time, so the results are not
contaminated by prior worker store state.

## Why a separate repo

The [`ak2k/nix-507531-repro`](https://github.com/ak2k/nix-507531-repro) flake
is `aarch64-darwin`-only by design (it exercises fish-4.2.1, which is what
breaks the cached darwin closure). Rather than retrofit x86_64 paths
into it, this repo keeps the x86_64-specific verification isolated and
tied to GHA's `macos-13` runner lifecycle.

## Related links

- PR: <https://github.com/NixOS/nix/pull/15638>
- Upstream issue: <https://github.com/NixOS/nixpkgs/issues/507531>
- Scope scan (empirical prevalence): <https://github.com/ak2k/nix-507531-scope>
- `aarch64-darwin` repro flake: <https://github.com/ak2k/nix-507531-repro>
