# Charon & Aeneas: shortcomings encountered during libsignal-lite-verify M5

**Project context.** Extracting Signal's PQXDH key-agreement protocol (`rust/protocol/src/pqxdh.rs` at libsignal commit `6e5a0466b35e23925ad4b3a31a8c6572471c33d9`) to Lean 4 via Charon + Aeneas. Toolchain pins:

- Charon (submoduled in Aeneas): `e656e17bff6ca5efac8ab6919b9b74cb9a8dd8ad`
- Aeneas: `1b39931d2db787e7e5f298795f0d4934daf59eb4`
- Lean: `leanprover/lean4:v4.30.0-rc2`
- Rust: `nightly-2026-03-23`

**What this document is.** A precise catalog of every Charon/Aeneas issue encountered while attempting to extend extraction from `signal-crypto + libsignal-core` (M4, working) to `libsignal-protocol::pqxdh` (M5, blocked). Each entry is intended to be turned into an upstream issue or PR by the libsignal-lite-verify maintainer. Entries marked **WORKAROUND** are temporary patches we are applying so that work can proceed; they are tracked in `src-modifications.diff` and will be reverted when upstream lands a fix.

**Audience.** Charon and Aeneas maintainers; libsignal-lite-verify project maintainer.

**Companion file.** This document has an HTML mirror at `charon-aeneas-shortcomings.html` in the same directory.

---

## Severity legend

- **🔴 Blocker** — extraction cannot proceed without a workaround or upstream fix.
- **🟡 Limitation** — a documented or undocumented constraint that forces a methodological compromise (extra annotations, opaque axioms, source modifications).
- **🟢 Cosmetic** — does not block correctness but affects ergonomics (poor error messages, misleading exit codes, etc.).

---

## Issue index

| ID                  | Title                                                                            | Severity     |
|---------------------|----------------------------------------------------------------------------------|--------------|
| AENEAS-001          | GAT in trait associated type fails translation                                   | 🔴 Blocker   |
| AENEAS-002          | Lifetime-parameterized struct breaks `PureTypeCheck.get_adt_field_types`         | 🔴 Blocker   |
| AENEAS-003          | Uncaught `Not_found` exception instead of structured error                       | 🟡 Limitation |
| AENEAS-004          | `PrePasses.update_array_default.visit_trait_impl` traverses trait impls Aeneas cannot handle | 🔴 Blocker (with workaround) |
| AENEAS-005          | Failure cascade: one bad item poisons unrelated function signatures              | 🟡 Limitation |
| AENEAS-006          | `Option<&T>` accessor methods fail "Internal error"                              | 🟡 Limitation |
| AENEAS-007          | Aeneas backend Lean version is implicit, hard to discover                        | 🟢 Cosmetic   |
| AENEAS-008          | `lake update aeneas` is not auto-run on `lakefile.toml: rev` change              | 🟢 Cosmetic   |
| AENEAS-009          | Dynamic trait types (`&dyn Trait`) not supported in symbolic interpreter         | 🔴 Blocker (with workaround) |
| AENEAS-010          | "There should be no bottoms in the value" — symbolic interpreter fails on `pqxdh_accept` body | 🟡 Limitation (with workaround) |
| AENEAS-011          | Duplicate parent-clause field name in `ZeroablePrimitive` trait codegen          | 🟡 Limitation (with workaround) |
| CHARON-001          | HKDF/SHA-2 closure bodies cannot be translated (known)                           | 🟡 Limitation |
| CHARON-002          | `#[charon::opaque]` and `#[charon::exclude]` on trait declarations: documented as "does nothing" but effectively required for downstream Aeneas success | 🟢 Cosmetic |
| CHARON-003          | `aeneas-config.yml: exclude` does not prevent Aeneas pre-passes from visiting the item | 🟡 Limitation |
| TOOLING-001         | `prost-build` requires system `protoc` even when extraction is gated by feature  | 🟡 Limitation |

---

## AENEAS-001 · GAT in trait associated type fails translation

**Severity:** 🔴 Blocker
**Status:** Workaround applied (source-level `#[cfg_attr(feature = "extraction", charon::exclude)]` on the trait declaration; see `src-modifications.diff`).

### Symptom

```
[Error] Can not extract trait associated types with parameters
Source: 'rust/protocol/src/handshake.rs', lines 31:0-56:1
Compiler source: symbolic/SymbolicToPure.ml, line 209

[Warn] Could not translate the trait declaration 'libsignal_protocol::handshake::Handshake
       because of previous error
```

### Triggering code

```rust
// rust/protocol/src/handshake.rs:31-56
pub(crate) trait Handshake {
    type InitiatorParams;
    type RecipientParams<'a>;          // ← GAT: lifetime parameter on associated type
    type InitiatorMessage;
    type SessionSecret;

    fn initiate<R: Rng + CryptoRng>(
        params: &Self::InitiatorParams,
        rng: &mut R,
    ) -> Result<(Self::InitiatorMessage, Self::SessionSecret)>;

    fn accept(params: &Self::RecipientParams<'_>) -> Result<Self::SessionSecret>;
}
```

The lifetime parameter `<'a>` on `RecipientParams` makes the associated type a Generic Associated Type (GAT). Aeneas's `SymbolicToPure` pass cannot extract GATs in trait declarations.

### Reproduction

Any trait with a `type X<'a>;` (or `type X<T>;`) associated type pattern. Minimal repro:

```rust
trait T {
    type X<'a>;
}
```

### Current workaround

Source-level annotation on the trait declaration:

```rust
#[cfg_attr(feature = "extraction", charon::exclude)]
pub(crate) trait Handshake { ... }
```

This works around the issue by removing the trait from Charon's emission entirely. Implementations of the trait (which transitively depend on the GAT signature) must also be excluded — see AENEAS-004.

### Suggested upstream fix

Two reasonable paths:

1. Implement GAT translation in `SymbolicToPure`. This is the proper fix but likely a substantial engineering effort.
2. As a stop-gap, emit a structured `error: trait declaration uses GAT, skipping translation of this trait and its impls` and continue translating the rest of the crate. Currently the error blocks the entire crate's translation in cascade form (see AENEAS-005).

### References

- Aeneas source: `symbolic/SymbolicToPure.ml:209`
- Triggering libsignal code: `rust/protocol/src/handshake.rs:31-56`

---

## AENEAS-002 · Lifetime-parameterized struct breaks PureTypeCheck.get_adt_field_types

**Severity:** 🔴 Blocker
**Status:** Investigation ongoing; planning a Path-β source modification (`RecipientParameters<'a>` rewritten to own its data) tracked in the M5 handoff.

### Symptom

After excluding the GAT-bearing trait (AENEAS-001), Aeneas's translation proceeds further but every function whose signature references a lifetime-parameterized struct fails:

```
[Error] Internal error: please file an issue
Source: 'rust/protocol/src/pqxdh.rs', lines 199:0-241:1
[Warn] Could not translate the function signature of 'libsignal_protocol::pqxdh::pqxdh_initiate'
       because of previous error
Compiler source: Translate.ml, line 365
```

Eventually the crate-level translation crashes with:

```
Uncaught exception: Not_found
Raised at Stdlib__Map.Make.find in file "map.ml", line 141, characters 10-25
Called from Aeneas__PureTypeCheck.get_adt_field_types in file "pure/PureTypeCheck.ml", line 25
Called from Aeneas__PureMicroPassesAnnots.add_type_annotations_to_fun_decl.visit_App
            in file "pure/PureMicroPassesAnnots.ml", lines 313-314
```

### Triggering code

```rust
// rust/protocol/src/pqxdh.rs:245+
pub struct RecipientParameters<'a> {
    our_identity_key_pair: &'a IdentityKeyPair,
    our_signed_pre_key_pair: &'a KeyPair,
    our_one_time_pre_key_pair: Option<&'a KeyPair>,
    our_kyber_pre_key_pair: &'a kem::KeyPair,
    their_identity_key: &'a IdentityKey,
    their_ephemeral_key: &'a PublicKey,
    their_kyber_ciphertext: &'a kem::SerializedCiphertext,
    self_session: bool,
}
```

The struct holds borrowed references (`&'a T` fields) and exposes a lifetime parameter `<'a>`. Aeneas's `PureTypeCheck.get_adt_field_types` cannot find field-type info for this lifetime-parameterized ADT, raising an uncaught `Not_found` from `Stdlib.Map.Make.find`.

### Reproduction

Likely any non-trivial struct with a lifetime parameter that is referenced from a function signature emitted to LLBC. Minimal repro would be:

```rust
pub struct S<'a> { x: &'a u32 }
pub fn f(s: &S<'_>) -> u32 { *s.x }
```

(Untested; may need additional triggering conditions.)

### Current workaround

None yet. Planned Path-β fix: rewrite `RecipientParameters<'a>` to own its data (`Clone` the borrowed references at construction time). This introduces a verification-equivalence assumption (the verified Lean is about an owned-data variant, with an axiom that this matches the upstream borrowed variant's semantics). To be tracked in `src-modifications.md`.

### Suggested upstream fix

`PureTypeCheck.get_adt_field_types` should either:

1. Handle lifetime-parameterized ADTs correctly (lifetimes are erased to `()` in Aeneas's pure model anyway, so the field types should be discoverable).
2. Raise a structured error pointing at the offending ADT, rather than letting `Map.find` propagate `Not_found` as an uncaught exception.

### References

- Aeneas source: `pure/PureTypeCheck.ml:25`, `pure/PureMicroPassesAnnots.ml:313-314, 368`
- Triggering libsignal code: `rust/protocol/src/pqxdh.rs:245-285`

---

## AENEAS-003 · Uncaught Not_found exception instead of structured error

**Severity:** 🟡 Limitation
**Status:** No workaround; this is a meta-issue about error reporting.

### Symptom

When the symbolic-to-pure translation hits an unsupported pattern, Aeneas sometimes crashes with `Stdlib.Map.Make.find` raising `Not_found` as an uncaught OCaml exception, rather than a structured "please file an issue" error pointing at the user's source location.

Example call stack (from AENEAS-002):

```
Uncaught exception: Not_found
Raised at Stdlib__Map.Make.find in file "map.ml", line 141
Called from Aeneas__PureTypeCheck.get_adt_field_types in pure/PureTypeCheck.ml:25
```

The user has no indication of which type or function caused the crash. The only way to localize is to comment out source items and re-run.

### Suggested upstream fix

Wrap `Map.find` calls in `PureTypeCheck` (and similar lookup sites) with structured-error reporting that includes:

- The ADT name being looked up
- The function/decl that referenced it
- The source span where the reference originates

A general approach: introduce a `Result.t`-returning variant of `get_adt_field_types` and let the caller emit an `Errors.craise` with a span. The existing `Errors.craise_opt_span` infrastructure (used at `Errors.ml:123`) is the right pattern.

### References

- Aeneas source: `pure/PureTypeCheck.ml:25`

---

## AENEAS-004 · PrePasses traverses trait impls Aeneas cannot handle

**Severity:** 🔴 Blocker (with workaround)
**Status:** Workaround applied (source-level `#[cfg_attr(feature = "extraction", charon::exclude)]` on the impl block).

### Symptom

Even when an offending trait is marked `--exclude` in `aeneas-config.yml`, Aeneas's pre-passes still traverse the trait *impl* block of that excluded trait. The traversal can crash, e.g.:

```
[Error] Internal error: please file an issue
Source: 'rust/protocol/src/pqxdh.rs', lines 38:0-55:1
Uncaught exception: Aeneas.Errors.CFailure(_)
Raised at Aeneas__Errors.craise_opt_span in file "Errors.ml", line 123
Called from Aeneas__PrePasses.update_array_default.visit_trait_impl
            in file "PrePasses.ml", lines 190-191
```

The trait impl at `pqxdh.rs:38-55` is `impl Handshake for Pqxdh`, where `Handshake` was excluded via aeneas-config. The pre-pass still visits the impl, presumably because it walks all items in the LLBC regardless of opacity classification.

### Triggering code

```rust
// rust/protocol/src/pqxdh.rs:38
impl Handshake for Pqxdh {
    type InitiatorParams = InitiatorParameters;
    type RecipientParams<'a> = RecipientParameters<'a>;
    type InitiatorMessage = kem::SerializedCiphertext;
    type SessionSecret = HandshakeKeys;
    fn initiate<R>(...) -> Result<...> { pqxdh_initiate(params, rng) ... }
    fn accept(params: &Self::RecipientParams<'_>) -> Result<...> { pqxdh_accept(params) }
}
```

### Current workaround

Source-level annotation on the impl block:

```rust
#[cfg_attr(feature = "extraction", charon::exclude)]
impl Handshake for Pqxdh { ... }
```

This works because `charon::exclude` is honored at MIR emission time (see CHARON-002), so the impl is dropped from the LLBC entirely and Aeneas never sees it.

### Suggested upstream fix

Two possibilities:

1. **In Aeneas:** Pre-passes (PrePasses.ml) should filter trait impls of excluded traits before visiting them. The opacity classification at LLBC import time should propagate through to pre-passes.
2. **In Charon:** Document explicitly that excluding a trait at the CLI/config layer does *not* exclude its impls; impls must be excluded individually (or via source-level `#[charon::exclude]`).

The current ergonomics are surprising: excluding a trait via config makes Aeneas crash, which the user has to interpret as "I also need source-level annotations on every impl."

### References

- Aeneas source: `PrePasses.ml:190-191`, `Errors.ml:123`
- Charon docs (item selection mechanics): `libsignal-verify/item-selection.md:39`

---

## AENEAS-005 · Failure cascade: one bad item poisons unrelated signatures

**Severity:** 🟡 Limitation
**Status:** No workaround at the user level; this is a robustness issue in Aeneas.

### Symptom

When one item fails translation (e.g., the GAT in AENEAS-001), every function that *transitively references* anything in that item's neighborhood fails too with:

```
[Warn] Could not translate the function signature of '<fn>' because of previous error
```

In our M5 attempt, the GAT failure in `Handshake` cascaded to ~30 functions in pqxdh.rs, including some (like the structs' simple accessors) that have no actual dependency on the GAT.

### Suggested upstream fix

Translation passes should be per-item with structured error containment:

- A failed item produces a translation-failure axiom rather than poisoning the crate.
- Downstream items should still translate if their declared signature can be expressed in terms of the failure axiom.

This is a substantial change to Aeneas's translation pipeline, but it would make iterative extraction (the typical workflow) much more usable.

---

## AENEAS-006 · Option<&T> accessor methods fail "Internal error"

**Severity:** 🟡 Limitation
**Status:** Workaround applied (mark functions opaque in `aeneas-config.yml`).

### Symptom

```
[Error] Internal error: please file an issue
Source: 'rust/protocol/src/pqxdh.rs', lines 170:4-172:5
Compiler source: Translate.ml, line 266
```

### Triggering code

```rust
// rust/protocol/src/pqxdh.rs:170
pub fn their_one_time_pre_key(&self) -> Option<&PublicKey> {
    self.their_one_time_pre_key.as_ref()
}
```

The signature `Option<&PublicKey>` (an Option of a borrow) appears to interact badly with Aeneas's region analysis. The exact root cause is not yet isolated; the body uses `.as_ref()` to convert `&Option<T>` to `Option<&T>`.

### Current workaround

Mark these accessors opaque in `aeneas-config.yml`:

```yaml
opaque:
  - "libsignal_protocol::pqxdh::_::their_one_time_pre_key"
  - "libsignal_protocol::pqxdh::_::our_one_time_pre_key_pair"
```

### Suggested upstream fix

Minimal repro needed first. Suspected interaction: Aeneas's region analysis on functions returning `Option<&T>` where `T` is a foreign type or an enum.

---

## AENEAS-007 · Aeneas backend Lean version is implicit, hard to discover

**Severity:** 🟢 Cosmetic
**Status:** Workaround in the libsignal-lite-verify SKILL.md (read `<aeneas-repo>/backends/lean/lean-toolchain` empirically).

### Symptom

Each Aeneas commit corresponds to a specific Lean toolchain version (the version the `backends/lean/` library is authored against). This pairing is:

1. Not advertised in Aeneas's README or release notes.
2. Discoverable only by reading `<aeneas-repo>/backends/lean/lean-toolchain` at the chosen commit.
3. Easy to get wrong, with bad symptoms: kernel type-mismatch errors in `Aeneas/Std/Primitives.lean` referencing constructs that exist in one Lean version but not the other.

We initially set Lean to `v4.30.0-rc2` based on a third-party project's pin, only to find that the relevant Aeneas commit actually used `v4.30.0-rc2` (correct) — but our `lake-manifest.json` cached an older Aeneas commit (`1180be60`) whose backend wanted `v4.28.0-rc1`. The mismatch was undiscoverable without inspecting the lake cache.

### Suggested upstream fix

In the Aeneas repo, add a `LEAN-COMPATIBILITY.md` (or annotate the release tags) with the Lean version each commit was tested against. Even a single-line stamp like `# Compatible with leanprover/lean4:v4.30.0-rc2` in `backends/lean/README.md` would help.

---

## AENEAS-008 · `lake update aeneas` is not auto-run on `lakefile.toml: rev` change

**Severity:** 🟢 Cosmetic
**Status:** Workaround in the libsignal-lite-verify SKILL.md (manual `lake update aeneas`).

### Symptom

After bumping `lakefile.toml: rev = "<new-commit>"`, the next `lake build` silently uses the OLD cached commit (from `.lake/packages/aeneas/`). Symptom: kernel type-mismatch errors that look like a Lean version bug.

This is technically a Lake/lake-manifest issue, not Aeneas's. But the user impact is in the Aeneas pipeline. A note in the Aeneas `aeneas-cli`-style scripts (or the SPQR-derived scripts our project uses) to auto-`lake update aeneas` on `aeneas.commit` changes would prevent the foot-gun.

### Suggested upstream fix

Add a hint to `aeneas-install.ts` (or the equivalent in `aeneas-cli`) that detects `lakefile.toml: rev` ≠ `lake-manifest.json: inputRev` and prints:

```
warning: lakefile.toml has aeneas rev <X> but lake-manifest pins <Y>.
         Run `lake update aeneas` to refresh.
```

---

## AENEAS-009 · Dynamic trait types (`&dyn Trait`) not supported

**Severity:** 🔴 Blocker (with manual monomorphization workaround)
**Status:** Workaround applied (source-level `#[cfg_attr(feature = "extraction", charon::exclude)]` on the trait + its blanket impl, plus inline replacement of vtable dispatch with direct `match`-on-enum at call sites; see `src-modifications.md` M05–M07).

### Symptom

```
[Error] Dynamic trait types are not supported yet
Source: 'rust/protocol/src/kem.rs', lines 145:4-147:5
```

### Triggering code

```rust
// rust/protocol/src/kem.rs:108-126
trait DynParameters {
    fn public_key_length(&self) -> usize;
    fn ciphertext_length(&self) -> usize;
    fn encapsulate(&self, pub_key: &KeyMaterial<Public>, csprng: &mut dyn CryptoRng)
        -> Result<(SharedSecret, RawCiphertext)>;
    // ...
}

// kem.rs:226-234 — vtable accessor returning &'static dyn DynParameters
impl KeyType {
    const fn parameters(&self) -> &'static dyn DynParameters {
        match self {
            KeyType::Kyber1024 => &kyber1024::Parameters,
            // (other cfg-gated variants)
        }
    }
}

// kem.rs:354-357 — vtable dispatch at the encapsulate call site
let (ss, ct) = self.key_type.parameters().encapsulate(&self.key_data, csprng)?;
```

### Root cause

Aeneas's translation is a symbolic interpreter that requires "what does the next instruction do" to be computable *statically*. A method call on `&dyn Trait` goes through a runtime vtable lookup, which the symbolic interpreter cannot fold into a fixed-arity functional term.

A more structural discussion (including the connection between Aeneas's forward/backward function pairs and morphisms of polynomial functors / lenses) is kept as a local working note in the source tree and is not part of this published catalog.

### Reproduction

Define a trait with `&dyn Trait` parameters and call a method on a `&'static dyn Trait` value through static dispatch on an enum. Aeneas's `Translate` phase will reject the impl with "Dynamic trait types are not supported yet".

### Workaround

Three coordinated source-level changes (manual monomorphization):

1. `#[cfg_attr(feature = "extraction", charon::exclude)]` on the dyn trait declaration (e.g., `DynParameters`).
2. `#[cfg_attr(feature = "extraction", charon::exclude)]` on the blanket `impl<T: StaticTrait> DynTrait for T`.
3. Inline replacement of each vtable dispatch site `obj.dyn_method(...)` with a direct `match` on the variant tag, calling the corresponding concrete impl: `match key_type { KeyType::Kyber1024 => kyber1024::Parameters::encapsulate(...) }`.

This is the libsignal-lite-verify M5 "Plan A" rewrite — applied to `kem.rs` at the two PQXDH-relevant call sites (`Key<Public>::encapsulate` and `Key<Secret>::decapsulate`). Other vtable callers in `kem.rs` (e.g., `KeyPair::generate`, `KeyKind::key_length` impls) remain opaque in `aeneas-config.yml: opaque` for this scope.

### Suggested upstream fix

Two framings, in increasing order of engineering effort:

1. **Bounded-monomorphization translation mode** (cheap, probably sufficient). When the trait has a closed set of impls in the crate, emit a `match` over the variant tag at LLBC time, inlining each impl's method body. This automates the manual rewrite above.

2. **Dictionary-passing translation** (the general solution). Compile each `&dyn Trait` site into an extra function argument that is a record of the trait's methods. This is how GHC compiles Haskell typeclasses and how Hax handles limited `dyn` cases. Substantial engineering: requires changes to LLBC, the type translator, and the borrow tracker.

The libsignal use case is well-served by (1); (2) handles the general case but produces ergonomically painful Lean (existential elimination at every call site) that verification consumers typically prefer to avoid.

### References

- The Aeneas paper (Ho & Protzenko, ICFP 2022) — discusses static dispatch but explicitly punts on dynamic dispatch.
- Hax (the other major Rust → formal-method tool): https://github.com/hacspec/hax — has begun limited `dyn` support via dictionary passing.

---

## AENEAS-010 · "There should be no bottoms in the value" — early-return-with-Err carrying `&'static str` payload

**Severity:** 🟠 Bug (symbolic-interpreter completeness gap)
**Status:** Trigger isolated by bisection; source rewrite applied. Original opaque workaround removed.

### Symptom

```
[Error] There should be no bottoms in the value
Source: 'rust/protocol/src/pqxdh.rs', lines 331:0-381:1
[Warn] Could not translate the body of function 'libsignal_protocol::pqxdh::pqxdh_accept
       because of previous error
Compiler source: interp/Interp.ml, line 550
```

### Triggering code (minimum failing example)

An early-return-with-Err where the Err variant carries a `&'static str` literal payload:

```rust
pub(crate) fn f(parameters: &SomeStruct) -> Result<T> {
    if !parameters.cond() {
        return Err(SomeError::Variant("some literal"));   // ← &'static str payload
    }

    // ... rest of body ...
}
```

The original libsignal pattern at `rust/protocol/src/pqxdh.rs:331-338`:
```rust
if !parameters.their_ephemeral_key.is_canonical() {
    return Err(SignalProtocolError::InvalidMessage(
        CiphertextMessageType::PreKey,
        "incoming base key is invalid",     // ← this &'static str triggers AENEAS-010
    ));
}
```

### Bisection findings

Progressive simplification of `pqxdh_accept`'s body isolated the trigger. Variants that **eliminate the error**:

- Remove the early-return entirely → ✓ extracts.
- Keep the `is_canonical()` call but drop the early-return (`let _ = parameters.x.is_canonical();`) → ✓ extracts.
- Replace `InvalidMessage(CiphertextMessageType::PreKey, "...")` with a unit variant `InvalidKeyAgreement` → ✓ extracts.
- Replace with a 2-arg variant *without* `&'static str` (e.g. `BadKeyLength(KeyType, usize)`) → ✓ extracts.

Variant that **reproduces the error**:

- Replace with any single-arg variant carrying `&'static str` (e.g. `InvalidSessionStructure("...")`) → ✗ "no bottoms".

So the trigger is specifically: **early-return-with-Err where the Err carries any `&'static str` literal payload**. The trigger is not about the `is_canonical()` call, the `?`-propagation, the `Vec::extend_from_slice` chain, the `decapsulate`/`Box<[u8]>` chain, or the optional `if let Some(...)` path — all of those translate fine when the offending early-return is removed.

The symmetric `pqxdh_initiate` (lines 199-241) translates because it has no early-return-with-string-Err pattern.

### Root cause (hypothesis)

The symbolic interpreter (`interp/Interp.ml:550`) appears to lose track of a value when an `&'static str` literal flows through an early-return path and the join-point analysis encounters the value as `⊥`. The bug may be in how literal `&'static str` values are projected at the return-edge, or how the symbolic state is reconciled at the post-if join when one branch returns early with a value containing a string literal.

### Workaround (now applied via source rewrite, not opaque)

Source rewrite at `rust/protocol/src/pqxdh.rs`: replace the `InvalidMessage(...)` early-return with the unit variant `InvalidKeyAgreement`:

```rust
if !parameters.their_ephemeral_key.is_canonical() {
    return Err(SignalProtocolError::InvalidKeyAgreement);   // M10: was InvalidMessage(...)
}
```

Semantic equivalence: `calculate_agreement` invoked on a non-canonical key produces an `InvalidKeyAgreement` error downstream, so the early-return-with-`InvalidKeyAgreement` and the original early-return-with-`InvalidMessage("incoming base key is invalid")` are observably equivalent up to the message string (which is for human consumption only). Recorded as M10 in `src-modifications.md`.

### Reproduction

Apply the libsignal-lite-verify M5 Plan A patches (M01-M09) at the Aeneas `1b39931d` pin + Charon at the matching pin, then revert M10 (re-apply the `InvalidMessage(CiphertextMessageType::PreKey, "incoming base key is invalid")` early-return at `rust/protocol/src/pqxdh.rs:331-338`). `npm run aeneas-extract` will report the "no bottoms" error.

A minimal standalone reproducer (no libsignal dependency) would be a Rust function returning `Result<T, E>` where `E` is an enum with a variant carrying `&'static str`, and the function body returns `Err(E::Variant("any literal"))` from inside an `if` branch before defining any locals. This shape should be enough to fire the symbolic-interpreter bottom error.

### Suggested upstream fix

Investigate the symbolic-interpreter's handling of `&'static str` literals on early-return paths. The minimum failing example above should make the trigger pattern easy to isolate. Two reasonable directions:

1. Track string-literal values concretely through symbolic state (they are constants — they should never be ⊥).
2. If the bottom is arising from a derived expression rather than the literal itself, identify which derived expression and fix the symbolic-state propagation at the early-return edge.

The "no bottoms" diagnostic should also be improved to point at the specific value / program point where `⊥` was encountered, instead of just naming the enclosing function.

### References

- Source rewrite applied: M10 in `src-modifications.md`. Original opaque workaround (M09) removed.
- Funs.lean now extracts `pqxdh_accept` as a transparent def alongside `pqxdh_initiate`.

---

## AENEAS-011 · Duplicate parent-clause field name in trait codegen

**Severity:** 🟡 Limitation (with tweaks workaround)
**Status:** Workaround applied — `aeneas-config.yml: tweaks: substitutions` rename the duplicate field. Note: the issue only manifests when the M5 extraction graph reaches `core::num::nonzero::ZeroablePrimitive`; opaquing `pqxdh_accept` (AENEAS-010 workaround) currently keeps the graph small enough that `ZeroablePrimitive` is not pulled in, so the tweaks substitutions do not fire in practice for this build configuration. The tweaks are retained for forward-compatibility.

### Symptom

```
error: Libsignal/Code/Types.lean:42:2: Field `markerCopyInst` has already been declared
```

The generated Lean structure has two fields with the same name:

```lean
@[rust_trait "core::num::nonzero::ZeroablePrimitive"
  (parentClauses := ["markerCopyInst", "privateSealedInst", "markerCopyInst"])]
structure core.num.nonzero.ZeroablePrimitive (Self : Type) (Self_NonZeroInner : Type) where
  markerCopyInst : core.marker.Copy Self                  -- ←
  privateSealedInst : core.num.nonzero.private.Sealed Self
  markerCopyInst : core.marker.Copy Self_NonZeroInner      -- ← duplicate!
```

### Triggering code

In Rust standard library:
```rust
// core::num::nonzero
pub unsafe trait ZeroablePrimitive: Sized + Copy + private::Sealed {
    type NonZeroInner: Sized + Copy;
    // ...
}
```

The trait has **two** Copy bounds: one on `Self` (via the `: Copy` supertrait constraint) and one on the associated type `NonZeroInner`. Aeneas's codegen emits both as fields in the structure encoding of the trait, naming them by their trait class. Both get named `markerCopyInst` — collision.

### Reproduction

Extract any libsignal-protocol code whose type graph reaches `Uuid` → `NonZeroU64Inner` → the `ZeroablePrimitive` trait. The structure emission produces the duplicate field.

### Workaround

Tweaks substitutions in `aeneas-config.yml: tweaks: substitutions` rename the second `markerCopyInst` field to `markerCopyInnerInst` (and update the parentClauses list to match):

```yaml
- find: '(parentClauses := ["markerCopyInst", "privateSealedInst", "markerCopyInst"])'
  replace: '(parentClauses := ["markerCopyInst", "privateSealedInst", "markerCopyInnerInst"])'
- find: "markerCopyInst : core.marker.Copy Self_NonZeroInner"
  replace: "markerCopyInnerInst : core.marker.Copy Self_NonZeroInner"
- find: "markerCopyInst := core.num.niche_types.NonZeroU64Inner.Insts.CoreMarkerCopy"
  replace: "markerCopyInnerInst := core.num.niche_types.NonZeroU64Inner.Insts.CoreMarkerCopy"
- find: "markerCopyInst := core.num.niche_types.NonZeroU8Inner.Insts.CoreMarkerCopy"
  replace: "markerCopyInnerInst := core.num.niche_types.NonZeroU8Inner.Insts.CoreMarkerCopy"
```

(Last 3 substitutions do not currently fire because opaquing `pqxdh_accept` reduces the type-graph reach. Kept in place for forward-compatibility.)

### Suggested upstream fix

When emitting a structure for a trait with multiple parent-clause Copy / Clone / similar marker-trait constraints, mangle the field name with the parameter index or type: `markerCopyInst_Self`, `markerCopyInst_NonZeroInner`, etc.

### References

- Aeneas source: presumably in the trait-encoding pass that emits structure fields from trait clauses. Look for "parentClauses" handling.

---

## AENEAS-012 · Inconsistent tuple arity between def and call site for fn with `&mut R` reached via DerefMut blanket

**Severity:** 🟠 Bug (codegen inconsistency between def-site and call-site emitters)
**Status:** Workaround applied — `aeneas-config.yml: tweaks: substitutions` rewrite the call-site destructure and the back-fn invocation.

### Symptom

A function `f(&self, csprng: &mut R)` extracts to a def returning a 2-tuple `Result (X × R)` — the forward result and the threaded R, no backward continuation. Internally the def's body destructures inner axiom calls as 2-tuple, consistent with the def's return type.

But at a *call site* where R is reached through `rand_core.CryptoRng.Blanket (Mut0T.Insts.CoreOpsDerefDerefMut R) ...` (the blanket impl for any `T: DerefMut<Target: RngCore>`), Aeneas emits a **3-tuple** destructure as if `f` returned `(X × R × (R → R))` — with a phantom backward continuation. Lean rejects this:

```
error: expected a product type, got R
```

(Lean tries to unpack the second slot — which has type `R` — as if it were a pair `(intermediate_R, back_fn)`.)

### Triggering code

The Rust source (libsignal `rust/protocol/src/kem.rs`):

```rust
pub fn encapsulate<R: CryptoRng>(
    &self,
    csprng: &mut R,
) -> Result<(SharedSecret, SerializedCiphertext)> { ... }
```

Called from `pqxdh::pqxdh_initiate` as `parameters.their_kyber_pre_key.encapsulate(csprng)?`. In Aeneas's extracted Lean:

- **Def site** (`Funs.lean`, `kem.KeyPublic.encapsulate`):
  ```lean
  def kem.KeyPublic.encapsulate
    {R : Type} (rand_coreCryptoRngInst : rand_core.CryptoRng R)
    (self : kem.Key kem.Public) (csprng : R) :
    Result ((core.result.Result _ _) × R)
  ```
  Returns 2-tuple `(X × R)`.

- **Call site** (`Funs.lean`, inside `pqxdh.pqxdh_initiate`):
  ```lean
  let (r3, csprng1, encapsulate_back) ←
    kem.KeyPublic.encapsulate (rand_core.CryptoRng.Blanket
      (Mut0T.Insts.CoreOpsDerefDerefMut R) rand_coreCryptoRngInst)
      parameters1.their_kyber_pre_key csprng
  -- ...
  let csprng2 := encapsulate_back csprng1
  ```
  Expects 3-tuple `(X × R × (R → R))`.

The two emitters disagree on whether the backward continuation exists.

### Reproduction

Extract `libsignal_protocol::pqxdh::pqxdh_initiate` (which calls `Key<Public>::encapsulate(csprng)` where `csprng: &mut R` and the outer fn is itself generic over `R: CryptoRng`). The call site reaches `encapsulate` through `rand_core::CryptoRng` for `&mut R` (the blanket impl), which in Lean is `rand_core.CryptoRng.Blanket` wrapping `Mut0T.Insts.CoreOpsDerefDerefMut R`. Aeneas codegen at this call site emits the 3-tuple destructure, but the called function's def is 2-tuple.

### Workaround

Tweaks substitutions in `aeneas-config.yml: tweaks: substitutions` patch the call sites post-extraction to:
1. Drop the phantom `, encapsulate_back` slot from the destructure.
2. Replace each `encapsulate_back csprng_N` invocation with bare `csprng_N` (identity — equivalent because the def doesn't actually produce a back-fn).

```yaml
- find: "let (r3, csprng1, encapsulate_back) ←"
  replace: "let (r3, csprng1) ←"
- find: "let (r4, csprng1, encapsulate_back) ←"
  replace: "let (r4, csprng1) ←"
- find: "let csprng2 := encapsulate_back csprng1"
  replace: "let csprng2 := csprng1"
```

The find strings are specific to the variable names emitted in `pqxdh_initiate` (`r3`, `r4`, `csprng1`, `csprng2`). If Aeneas's emitter renames these in a future version, the tweaks will need updating.

### Suggested upstream fix

Reconcile the def-site and call-site code emitters so they agree on whether `&mut R`-via-DerefMut produces a backward continuation. Two reasonable resolutions:

- **(a) Always emit the backward continuation** for any &mut argument, regardless of whether the call site reaches it through a Blanket DerefMut path. Then the def returns 3-tuple and the call sites are consistent.
- **(b) Never emit the backward continuation** when the inner def doesn't need one (no nested borrows). The call-site emitter should match the def-site shape.

Whichever is chosen, the invariant "if call-site expects N-tuple, def-site must return N-tuple" should be enforced as a codegen postcondition.

### References

- Reproducible in this repo at `Libsignal/Code/Funs.lean` lines 761–822 (pre-tweaks). After tweaks apply, the destructure is 2-tuple and `lake build` succeeds.
- Workaround tweaks in `aeneas-config.yml` (search for `Fix Aeneas codegen: call sites for kem.KeyPublic.encapsulate`).

---

## CHARON-001 · HKDF/SHA-2 closure bodies cannot be translated (known)

**Severity:** 🟡 Limitation
**Status:** Workaround (standard pattern: mark caller opaque). Known pattern from SPQR and libsignal-verify.

### Symptom

```rust
let (root_key_bytes, ...) = derive_arrays(|bytes| {
    hkdf::Hkdf::<sha2::Sha256>::new(None, secret_input)
        .expand(label, bytes)
        .expect("valid length")
});
```

The closure body references `hkdf::Hkdf::<sha2::Sha256>` which pulls in deep generic trait hierarchies. Charon's MIR analysis times out or panics; even when it completes, Aeneas's translation produces unusable output.

### Current workaround

Annotate the caller function with `#[cfg_attr(feature = "extraction", charon::opaque)]` (or list it in `aeneas-config.yml: opaque`). The closure body is never read by Charon.

### Suggested upstream fix

This is the broadest "Charon-hostile crate" pattern. A proper fix would require Aeneas to support extraction of arbitrary trait hierarchies, which is research-grade work. The existing workaround (opaque caller) is acceptable for the libsignal-lite-verify use case.

### References

- SPQR extraction-notes: `SparsePostQuantumRatchet-verify/extraction-notes.md:41-49`

---

## CHARON-002 · #[charon::opaque/exclude] on traits and trait impls: documentation gap

**Severity:** 🟢 Cosmetic
**Status:** Documentation clarification needed in Charon's `item-selection.md`.

### Symptom

Charon's `item-selection.md:11` says:

> For traits and trait impls, this doesn't change anything.

This led us initially to skip applying `#[charon::opaque]` and `#[charon::exclude]` to traits/impls when working around AENEAS-001 and AENEAS-004. But empirically, **`#[charon::exclude]` on a trait declaration is the only way to keep its GAT out of the LLBC**, and `#[charon::exclude]` on a trait impl is the only way to keep that impl out of Aeneas's pre-pass traversal.

The "doesn't change anything" wording refers to the *opacity classification* (transparent/foreign/opaque/invisible) on the *trait or impl item itself*, but it doesn't say what happens to the contained associated types or methods. In practice, the source-level annotation makes those items invisible to Charon's MIR emission step (which is *before* opacity is applied).

### Suggested upstream fix

Reword `item-selection.md` to explicitly say:

> For traits and trait impls, the opacity classification on the trait/impl item itself does not change Charon's translation of its contained methods. However, `#[charon::exclude]` operates at MIR emission time and *will* prevent the trait/impl and its methods from being emitted to LLBC entirely — this is the right mechanism when an associated type cannot be translated (e.g., GATs).

---

## CHARON-003 · aeneas-config.yml `exclude` does not prevent Aeneas pre-passes from visiting

**Severity:** 🟡 Limitation
**Status:** Documented in SKILL.md; aligns with AENEAS-004's workaround.

### Symptom

Adding `libsignal_protocol::handshake::Handshake` to `aeneas-config.yml: exclude` did NOT prevent Aeneas from visiting `impl Handshake for Pqxdh` during pre-passes (see AENEAS-004 above). Only source-level `#[charon::exclude]` (which operates at MIR time, before Aeneas sees anything) worked.

### Suggested upstream fix

Same as AENEAS-004's suggestion: Aeneas pre-passes should respect the LLBC's opacity classification, including items excluded via Charon's CLI flags.

---

## TOOLING-001 · prost-build requires system protoc even when extraction is gated by feature

**Severity:** 🟡 Limitation
**Status:** Workaround: install `protoc` (`brew install protobuf`). Rationale documented separately in `why-protobuf-for-charon.html`.

### Symptom

`libsignal-protocol/build.rs` invokes `prost_build::compile_protos`, which shells out to a system `protoc` binary. Even if we want to extract only `pqxdh.rs` (which is prost-free), the crate cannot compile without `protoc` because `src/proto/storage.rs` etc. use `include!(concat!(env!("OUT_DIR"), "/signal.proto.storage.rs"))` to pull in prost-build's output.

### Current workaround

Install `protoc` system-wide and rely on `#[cfg_attr(feature = "extraction", charon::opaque)]` annotations to keep Charon out of prost-touching function bodies.

### Suggested upstream fix

Not a Charon/Aeneas issue per se. The libsignal upstream could optionally feature-gate `prost-build` to a non-default feature, allowing extraction-only builds to skip protoc. But this is upstream's call.

For Aeneas's documentation: a note in the "common pitfalls" section that `prost-build` (and other `build.rs`-based codegen tools) must be resolvable at extraction time, even if their output is annotated opaque.

---

## Workflow: temporary patches and revert plan

While the upstream issues above are open, the libsignal-lite-verify project is applying **temporary patches** to make M5 extraction proceed. Every patch is tracked precisely:

1. **In `src-modifications.diff`** — the authoritative diff against upstream libsignal `6e5a0466`. Regenerated by `npm run src-diff`.
2. **In `src-modifications.md`** — narrative catalog with rationale, alternatives considered, and the verification-equivalence assumption introduced (if any).
3. **In `src-assumptions.md`** — each patch's assumption is classified (provable / axiomatic / open gap) with a discharge plan.

### Patches currently applied (M5, Path α attempt)

| Patch location                                         | What changes                                                                | Linked issue |
|--------------------------------------------------------|------------------------------------------------------------------------------|--------------|
| `rust/protocol/src/handshake.rs:31`                    | Add `#[cfg_attr(feature = "extraction", charon::exclude)]` to `Handshake` trait | AENEAS-001   |
| `rust/protocol/src/pqxdh.rs:38`                        | Add `#[cfg_attr(feature = "extraction", charon::exclude)]` to `impl Handshake for Pqxdh` | AENEAS-004   |
| `rust/protocol/src/lib.rs:18-20`                       | Add `register_tool(charon)` prelude under `extraction` feature              | M5 prerequisite |

### Patches to be applied (M5, Path β)

To unblock M5 extraction, the planned next change is:

| Patch location                              | What changes                                                                 | Linked issue |
|---------------------------------------------|------------------------------------------------------------------------------|--------------|
| `rust/protocol/src/pqxdh.rs:245-285`        | Rewrite `RecipientParameters<'a>` to own its data (drop `<'a>`, change `&'a T` to `T`) | AENEAS-002   |

This is a **verification-equivalence assumption**: the verified Lean will be about an owned-data variant of `RecipientParameters`. An axiom in `FunsExternal.lean` will assert the semantic equivalence to the upstream borrowed variant. This is the methodologically standard "last-resort change" path.

### Revert plan

When AENEAS-001, AENEAS-002, AENEAS-004 are fixed upstream:

1. Bump the Aeneas pin in `aeneas-config.yml: aeneas.commit` + `lakefile.toml: rev` (and run `lake update aeneas`).
2. Remove the source-level `#[cfg_attr(feature = "extraction", charon::exclude)]` annotations and the owned-data rewrite.
3. Re-run `npm run aeneas-extract` and `lake build`.
4. Regenerate `src-modifications.diff` via `npm run src-diff` — the patches we authored should fall out of the diff.
5. Remove the corresponding assumption entries from `src-assumptions.md`.

### Filing upstream

Suggested venues for each issue:

- `AENEAS-001` through `AENEAS-008`: file as separate issues at https://github.com/AeneasVerif/aeneas/issues
- `CHARON-001` through `CHARON-003`: file at https://github.com/AeneasVerif/charon/issues
- `TOOLING-001`: not an upstream issue per se; document in our project notes

For each filing, attach the minimum failing example from the "Reproduction" section above (or extract one from the libsignal codebase).

---

## Document maintenance

When new issues are encountered, add them as new sections following the same template (Severity / Status / Symptom / Triggering code / Reproduction / Workaround / Suggested upstream fix / References). Increment the issue index at the top.

When an issue is filed upstream, add the upstream issue URL to the "References" section. When an issue is fixed upstream, add a "Fixed in" line with the commit / release tag.

**Document timeline:**

- 2026-05-24 — initial draft from libsignal-lite-verify M5 attempt at the Aeneas `1b39931d` / Charon `e656e17b` pin pair.
