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
rebasing onto the next release a no-op. One consequence of the published manifest is
worth stating: it carries `autotests = false` and no `[[test]]` targets, and upstream's
`tests/` expect the sibling `droidsaw-fixture-harness` crate that the normalized manifest
drops. `cargo test` therefore runs the crate's unit tests (`src/**` `#[cfg(test)]`), not
`tests/`; regression tests added on this branch belong in the unit-test modules (or, when
the shape needs a compiled fixture, compile it at test time with `javac` + `d8` and skip
cleanly when the toolchain is absent, as the constructor-pairing test below does).

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

Safety: `cargo test` (942 unit tests — the run that mentions integration tests only
happens in the upstream workspace, see §0 — including the structurer's tree-shape tests)
passes;
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

## 6. Enums whose constants carry no user arguments

`classes.rs` renders a javac-shaped enum as source-level declarations (`AMEX, VISA(...)`
at the top of the body) and, when it does, elides the members javac synthesises:
`$VALUES`, `$values()`, `values()`, `valueOf(String)`, the `(String name, int ordinal)`
pair on the constructor, the implicit `super(name, ordinal)` call, and the `<clinit>`
that populates the constants. The recogniser that decides whether that inline render is
safe requires the `$values()` static call in `<clinit>`.

R8 does not always emit it: `Lcom/mobsandgeeks/saripaar/annotation/CreditCard$Type;`
builds its `$VALUES` array inline, so the recogniser bailed — and because the
suppression rows were gated on the *structural* test (`ACC_ENUM` + superclass
`Ljava/lang/Enum;`) instead of on the inline render succeeding, the fall-through render
came out self-contradictory:

```java
private CreditCard$Type() { }                     // parameters and super() gone
static { ... new CreditCard$Type("AMEX", 0); }    // called with two arguments
```

with `values()` / `valueOf(String)` / `$VALUES` suppressed although the declarations that
stand in for them were never emitted. The fix (`enum_ctx.applies = false` when
`enum_constant_emitted` is empty, plus the matching gate on the constructor parameter
stripping in `decompile_method`) leaves the fall-through path rendering what the class
really contains:

```java
private CreditCard$Type(String v1, int v2) { super(v1, v2); }
public static CreditCard$Type valueOf(String v1) { ... }
public static CreditCard$Type[] values() { ... }
```

Measured effect: `cargo test` (942 unit tests) still passes; on the device corpus (1,080
classes across five archives) `bench/quality_vs_reference.py` drops from 157 to 146
flagged classes (method-set 129 -> 117) with no class regressing. Before this goes
upstream the shape wants a byte-for-byte fixture in `tests/` so the branch that R8
triggers here is pinned.

## 7. `invoke-direct/range` pairs with its `new-instance`

Emitting `new T(args)` requires pairing a `new-instance` with the `invoke-direct` that
runs the constructor: the emitter looks ahead for the `<init>` call, folds it into the
`new` expression, and records the receiver as inlined. That look-ahead matched one
opcode:

```rust
if init_insn.insn.op == Opcode::InvokeDirect {
```

The range form of the same call was invisible to it. `Lcom/xiaomi/push/r3;<clinit>` is
such a pair (receiver plus five argument words, one of them the `long` timeout):

```
new-instance v7, ThreadPoolExecutor;
const/4 v1, 1
const/4 v2, 1
const-wide/16 v3, 15
sget-object v5, TimeUnit.SECONDS
new-instance v6, LinkedBlockingQueue; invoke-direct {v6}, <init>()V
move-object v0, v7
invoke-direct/range {v0 .. v6}, <init>(I I J TimeUnit$ BlockingQueue)V
sput-object v7, r3.a
```

SSA coalesces the receiver copy, so the range call's first use *is* the `new-instance`'s
`VarId`. Because the look-ahead only knew the 35c form, it never reached the call, the
pair stayed unmerged, and the class came out as two statements — a bare `new T()` bound
to the variable and a discarded `new T(args)` expression:

```java
java.util.concurrent.ThreadPoolExecutor v7_0 = new java.util.concurrent.ThreadPoolExecutor();
new java.util.concurrent.ThreadPoolExecutor(1, 1, v3_3, v5_4, v6_5);
a = v7_0;
```

`new ThreadPoolExecutor()` is not valid Java (no zero-argument constructor exists), and
the field received an object that was never constructed while the real construction was
thrown away. The fix matches both encodings
(`Opcode::InvokeDirect | Opcode::InvokeDirectRange`); the emit site that renders the
paired form (`insn.dst.is_none() && matches!(op, InvokeDirect | InvokeDirectRange)`)
already handled the range case, so nothing else changed. The other look-aheads in the
module already listed both opcodes; this was the one that did not.

Measured effect: `Lcom/xiaomi/push/r3;<clinit>` now renders the construction as the
initialiser of the variable that is stored:

```java
long v3_3 = 15L;
java.util.concurrent.TimeUnit v5_4 = java.util.concurrent.TimeUnit.SECONDS;
java.util.concurrent.LinkedBlockingQueue v6_5 = new java.util.concurrent.LinkedBlockingQueue();
java.util.concurrent.ThreadPoolExecutor v7_0 = new java.util.concurrent.ThreadPoolExecutor(1, 1, v3_3, v5_4, v6_5);
a = v7_0;
```

`cargo test` gains `emit::tests::ctor_range_opcode_pairs_with_new_instance`, which
compiles a five-argument (one `long`) construction with `javac` + `d8` at test time,
asserts the fixture really contains an `invoke-direct/range` (so the test cannot quietly
cover the 35c path instead), and then requires that the constructed variable is the one
stored in the field with no bare `new T()` anywhere. Reverting the one-line matcher makes
it fail with the output above. ASC's acceptance scenario `ctor-invocation-pairing`
(`tests/acceptance/scenarios.json`) pins the same defect against the real device APK.

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
