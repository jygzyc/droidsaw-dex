# rasc branch: droidsaw-dex 2.0.0 + the `rasc` patches

This branch is [`droidsaw/droidsaw-dex`](https://github.com/droidsaw/droidsaw-dex) at the
`2.0.0` release commit (`7ab14972`, 2026-06-11) plus the changes the native Rust CLI
`rasc` (the `rust` branch of the ASC repository) needs. The ASC repository vendors it at
`vendor/droidsaw-dex` (a plain directory, not a submodule) and patches `droidsaw-dex` to
that path in its `Cargo.toml`.

Updating to a new upstream release: on this branch, `git fetch upstream`, `git rebase
<new tag>`, push, then copy the tree over ASC's `vendor/droidsaw-dex` and rebuild. The
commits are ordered so that the rebase is a plain replay — the manifest commit first (it is
the one upstream will conflict with, on a version bump), then the parser patches.

Everything outside `Cargo.toml`, `src/parser/mod.rs`, `src/structure.rs` and the register
list in `src/decode.rs` is upstream's tree, untouched: the rest of `src/` is byte-identical
to the published crate, and `LICENSE` (BSD-3-Clause) is kept as shipped. The patched lines
are marked `PATCHED (rasc)`.

Two patches change decompiled output, and they are the reason this branch carries
`src/structure.rs` and touches the decoder at all: [4] keeps a shared `||` guard-chain body
on every path (including paths that share a body inside a loop), and [5] represents
`/range` register lists longer than five registers, which the SSA pass and the emitter need
in order to render every argument of a call.

## 0. Self-contained manifest (fork-only, not an upstream request)

Upstream's repository is the crate directory of the droidsaw development workspace: its
`Cargo.toml` says `rust-version.workspace = true` / `[lints] workspace = true` and depends on
`../droidsaw-common` and `../droidsaw-fixture-harness` by path. None of those exist in this
repository, so a plain checkout cannot build — the repository is only buildable inside the
workspace it was split out of. This branch therefore carries the **crates.io-normalized
manifest** (`Cargo.toml`, exactly what cargo generated when publishing 2.0.0, which resolves
the workspace keys and drops the path dependencies) and keeps upstream's own manifest beside
it as `Cargo.toml.orig`. The single functional difference: `droidsaw-common` comes from
crates.io instead of a sibling directory.

`tests/`, `fuzz/`, `proofs/` and `examples/` are all retained from upstream. They are
excluded from the published package, are harmless here, and keeping them is what makes
rebasing onto the next release a no-op.

The published tarball's `.cargo_vcs_info.json` (a crates.io artifact recording the source
revision) is not carried: this branch *is* the source, not a repackaged release.

## 1. `DexFile::parse_for_decompilation`

Skips two emit-only stages:

- the whole-input SHA-1 that only feeds `input_checksums_canonical`, read solely by
  `emit_dex.rs`;
- `parse_map_driven_sections`, whose outputs are method handles, call-site ids and
  the section-walk parse-error side channel. All three are consumed by emit; rasc
  reads none of them.

Measured effect on its own: ~6 ms of a ~100 ms parse, so it exists mainly to make
the scoped parse below expressible without changing the full parse's behaviour.

## 2. `DexFile::parse_for_class`

Parses the per-class `class_data`, `code_item` and static-value tables for the
requested descriptor only; every other class keeps its `class_def` row but no
bodies. Also falls back to the full parse when the descriptor matches no class, so
a non-canonical spelling can never yield a DEX whose classes have no code.

Measured effect: 120.7 ms -> 23.7 ms on the 9.8 MiB `classes.dex` of the benchmark APK
(24,436 class_def rows); the per-class decompile cost (~21 ms) is unchanged, because it
already only walks the requested class.

## 3. `parse_string_pool` decodes the pool in parallel

The entries are independent, and rayon's ordered collect keeps the first error
(lowest index) winning, so behaviour is unchanged while the pool decodes on every
core. Measured on the same 9.8 MiB DEX: 9.4 ms -> sub-millisecond for a class-scoped
parse, which is most of what a `getclass` scoped parse still spent.

## 4. `||` guard chains keep their body on every path

Dalvik lowers `a || b || c` in a condition to a *guard chain*: each test branches to
one shared body block placed after the chain, otherwise it falls through to the next
test. The structurer's block walk (`drive_processing` in `src/structure.rs`) marks a
block as visited when it first structures it and skips it on every later visit, so
the walk structured the body at the first guard and the two later guards then saw an
already-visited target. The if/else frames render an already-visited target as no
body at all, so the body silently disappeared from those paths — for
`androidx.core.content.IntentSanitizer$Api31Impl.checkOtherMembers` the
`Consumer.accept(...)` call survived only in the first of the three `||` arms.

Two changes in `src/structure.rs`:

- The guard frames now carry their branch target and merge block, and the walk
  remembers the statement tree it emitted for a `(target, merge)` region
  (`shared_regions`, capped at 64 regions per method). A later guard on the same
  target emits an identical copy of the remembered region instead of an empty body,
  so each path keeps the body and the region is still emitted once at its original
  position. Targets that are loop *headers* are never reused — that target is a
  `continue` shape, not a region — but a target inside a loop body is a normal shared
  body and is copied: `if (cond1 || cond2) { body }` inside a loop (the
  `getA11yFeatureToTileMapInternal` shape in `services.jar`) lost its body on the
  second guard without this.
- A post-pass rewrites `if (c) { } else { body }` to `if (!c) { body }`. That empty
then-body is exactly what a guard chain produced; the negated form says the same
thing the source-level condition said, and matches the reference implementation's
output for these methods.

Measured effect (whole-corpus `getclass` sweep, classes whose output contained an
empty control-flow body):

| corpus | classes | before | after |
| --- | ---: | ---: | ---: |
| fixture APK | 6,220 | 458 | 80 |
| `services.jar` | 18,692 | 1,206 | 232 |

`unsupported` (the raw-smali fallback) is unchanged at 260 classes on `services.jar`:
this patch removes the *silent* loss, not the classes the emitter declines. What is left
is small and cosmetic: mostly an inverted `if (cond) { } else { body }` where the
reference writes `if (!cond) { body }`, guard chains whose shared target is a
switch-case arm, and paths whose copies ran into the per-method budget below.

A copy is only worth making while it stays small. A remembered region can itself
contain a copy — a guard chain inside a loop reuses a region that was already built
from earlier copies — so an unbounded reuse compounds with nesting depth.
`shared_region` therefore charges every copy against a per-method budget (65,536
statements) and `remember_region` stores no region larger than 4,096 statements.
Before the bound, `services.jar`'s
`com.android.server.display.DisplayPowerState$PhotonicModulator` structured to **2.27 GB**
of Java in 103 s by repeatedly copying a region that contained copies of another one;
with the bound it renders 821 KB in 0.5 s, and the empty-control-flow counts of both
corpora above are unchanged. Exhausting the budget costs only the copies that would
have followed — those paths keep the historical empty-body shape, never wrong code —
so a method can never come out worse than before the patch.

Safety: `cargo test` (955 tests — 941 unit + 14 integration/doc — including the
structurer's tree-shape tests) passes;
the two corpus sweeps above decompile every class of both archives without an error;
`bench/quality_vs_reference.py` in ASC — which compares `rasc getclass` against the
reference decompiler per class — reports, on the same fixture sample, flagged classes
dropping 16 -> 7, empty control-flow bodies 7 -> 0 and lost string literals 9 -> 1, with
no class losing a method or literal on `services.jar`'s
`AccessibilityManagerService` (the largest class in the corpus, 5,523 statements
rendered).

## 5. `/range` register lists longer than five registers

`invoke-*/range`, `filled-new-array/range` and `invoke-polymorphic/range` carry an 8-bit
`arg_count`, so a range list can name up to 255 registers. `RegList` stores five
registers inline (`regs: [u16; 5]`) and the decoder capped the list at five:

```rust
let count = arg_count.min(5) as u8;   // registers past the fifth never existed
insn.src = RegList::from_slice(registers[..count]);
```

The SSA builder, the emulator and the smali printer all read `insn.src.as_slice()`, so
every call with more than five argument registers lost its tail: `rasc getclass` printed
a call with fewer arguments than the callee's descriptor. `AccessibilityManagerService`
illustrates it — the reference and the upstream (dexdec-based) `rasc` both print five
arguments, this build printed four:

```java
// reference: five arguments, the last one the empty string
persistColonDelimitedSetToSettingLocked(v5, userId, v7, v8, "");
```

Fix: `RegList` gains a `range_len: u8`. When it is non-zero the list is a `/range` list
and `regs[0]` is the base; `register(index)` derives `base + index` (checked in `u32`),
`registers()` yields the whole list, and `len()` reports the wire `arg_count`, so
`as_slice()` keeps returning the inline prefix the existing callers expect. `regs[0]`
holds the base even at `arg_count == 0`, where the spec calls the field irrelevant but
the DEX encoder needs the original byte for byte-identity. `from_slice` takes the range
form for contiguous lists longer than five registers, and the decoder validates
`start_reg + arg_count` before building the list — which is what makes the compact
representation sound.

Measured effect: `services.jar`'s `AccessibilityManagerService` keeps all five arguments
and no longer differs from the reference on a single string literal;
`bench/quality_vs_reference.py` shows no other class regressing. `cargo test` covers the
representation with three new decoder cases: a seven-register `/range` call, an
`arg_count == 0` list whose `CCCC` field must round-trip, and `RegList`'s plain/range
byte-identity (`emit_dex`).

## Safety

`rasc` never re-emits DEX bytes, so none of the skipped structures are observable
in its output. Three layers guard the claim:

1. `src/apk.rs` unit tests against a synthetic APK (the DEX fixture carries a valid
   Adler-32 checksum and a `proto_ids` section so `droidsaw-dex` accepts it): the
   scoped parse keeps fewer class bodies yet decompiles the requested class to the
   same source, an unmatched descriptor falls back to the full parse, and
   `decompile_class` works end to end.
   One qualification, found by the corpus sweep: droidsaw's `@droidsaw R8Origin(...)`
   comments come from an analysis that walks *every* class body, so they are dropped
   by a scoped parse. The Java source is unaffected (verified by stripping those
   comment lines), and `bench/decompile_equivalence.py` reports them separately
   instead of treating them as a difference.
2. `bench/decompile_equivalence.py` diffs `rasc getclass` against an unpatched
   binary over a stratified sample of classes on real APKs.
3. `bench/contracts.sh` decompiles the same packaged class through its dotted and
   slashed spellings and requires identical sources.

## Dropping this branch

`UPSTREAM.md` in this branch is the ready-to-file summary of these changes
(measurements included). Once upstream grows a public parse scope, or accepts the
0x78-byte DEX 041 header discussed there, this branch can go back to a pointer at the
published release.
