# Macula fork of `rustc-hash` 2.1.2 (macula2)

Vendored fork of [rust-lang/rustc-hash](https://github.com/rust-lang/rustc-hash)
at version `2.1.2`, with two patches: (1) default-feature flip from `["std"]`
to `[]`, and (2) `FxHashMap` / `FxHashSet` route through `hashbrown` when the
`std` feature is off, so `no_std` consumers retain the type aliases that
upstream only emits in `std` mode.

Used by [macula-kernel](https://github.com/macula-io/macula-kernel)
to satisfy `quinn-proto`'s transitive dependency on `rustc-hash` without
pulling `std` into the kernel target.

## The patches

### Cargo.toml — default-feature flip + hashbrown dep

```diff
 [features]
-default = ["std"]
+default = []
 std = []
 nightly = []
+
+[dependencies.hashbrown]
+version = "0.15"
+default-features = false
+features = ["default-hasher"]
```

### src/lib.rs — FxHashMap / FxHashSet aliases for no_std

```diff
 #[cfg(feature = "std")]
 pub type FxHashMap<K, V> = HashMap<K, V, FxBuildHasher>;
+#[cfg(not(feature = "std"))]
+pub type FxHashMap<K, V> = hashbrown::HashMap<K, V, FxBuildHasher>;

 #[cfg(feature = "std")]
 pub type FxHashSet<V> = HashSet<V, FxBuildHasher>;
+#[cfg(not(feature = "std"))]
+pub type FxHashSet<V> = hashbrown::HashSet<V, FxBuildHasher>;
```

Both patches applied to `Cargo.toml` (cargo-normalized). `Cargo.toml.orig`
also updated for the feature flip; hashbrown dep lives in the normalized
form only because the orig is workspace-driven.

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
