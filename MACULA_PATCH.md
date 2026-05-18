# Macula fork of `rustc-hash` 2.1.2

Vendored fork of [rust-lang/rustc-hash](https://github.com/rust-lang/rustc-hash)
at version `2.1.2`, with a single patch flipping the default feature set
from `["std"]` to `[]` so `no_std` consumers (via Cargo's `[patch.crates-io]`)
do not get `std` re-enabled through feature unification.

Used by [macula-kernel](https://codeberg.org/macula-internal/macula-kernel)
to satisfy `quinn-proto`'s transitive dependency on `rustc-hash` without
pulling `std` into the kernel target.

## The patch

```diff
 [features]
-default = ["std"]
+default = []
 std = []
 nightly = []
```

Applied to both `Cargo.toml` (cargo-normalized) and `Cargo.toml.orig`
(upstream hand-written).

## Why this is needed

Upstream `rustc-hash` already correctly gates `extern crate std;` on
`feature = "std"` and declares `#![no_std]` at the top of `src/lib.rs`.
The crate's source compiles fine in a `no_std` environment when `std`
is off.

The problem is at the cargo-feature layer: `default = ["std"]` means any
consumer that does not pass `default-features = false` re-enables `std`
in feature unification. `quinn-proto` (the consumer that pulls
`rustc-hash` into macula-kernel's tree) takes `rustc-hash` with default
features, so `std` is unified on across the build. Source-level
`#![no_std]` is then irrelevant: `extern crate std;` at lib.rs:25 is
re-activated and emits `E0463: can't find crate for std` against the
kernel target.

Flipping `default = []` makes `std` strictly opt-in. Consumers that need
`HashMap` / `HashSet` (the only thing the `std` feature gates inside
this crate) can still enable it explicitly.

## Upstream contribution

The intent is to file an upstream PR proposing `default = []`. Some
projects have reasons to keep `std` in the default feature set (it is a
breaking change in feature semantics for anyone implicitly relying on
the default); upstream may or may not accept. If they reject, this fork
stays alive; if they accept, retire it.

## Versioning policy

Track `rustc-hash` upstream releases of 2.x. Re-roll this fork by:

1. Downloading the new `rustc-hash-X.Y.Z.crate` tarball from crates.io.
2. Extracting into a fresh checkout (replaces this directory contents).
3. Re-applying the `default = []` patch to both `Cargo.toml` and `Cargo.toml.orig`.
4. Tagging as `vX.Y.Z-macula1`.
5. Bumping the git ref in `macula-kernel`'s `Cargo.toml`.

When the upstream PR lands (if it does), retire this fork.

## License

Inherited from upstream `rustc-hash` (Apache-2.0 OR MIT). No new code,
only the Cargo.toml feature-default flip.
