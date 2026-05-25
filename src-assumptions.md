# Source assumptions

Trust boundaries and behavioral claims introduced by modifications in
`src-modifications.md`. Each assumption is the **scope-limiter** for any
theorem that depends on a modified function or struct.

## Classification

| Tag | Meaning |
|---|---|
| `provable`  | This assumption can be discharged by a separate Lean theorem (linked to its proof obligation) |
| `axiomatic` | We accept this as a primitive trust boundary (e.g., "the test-vector check passes"). Justification given inline |
| `open`      | We have not yet decided how to discharge this; flagged as a gap requiring resolution before any downstream theorem claims completeness |
| `revertible`| This assumption disappears entirely when a specific upstream Aeneas/Charon fix lands (the modification it covers reverts; the assumption no longer applies) |

---

## A01 — `Handshake::{initiate, accept}` for `Pqxdh` ≡ direct call to `pqxdh_initiate` / `pqxdh_accept`

**Invoked by:** M02 (handshake.rs trait exclusion) + M03 (pqxdh.rs impl exclusion).

**Statement:** The `impl Handshake for Pqxdh` block in `rust/protocol/src/pqxdh.rs:42-60` contains exactly the following two methods:

```rust
fn initiate<R: Rng + CryptoRng>(
    params: &Self::InitiatorParams,
    rng: &mut R,
) -> Result<(Self::InitiatorMessage, Self::SessionSecret)> {
    let result = pqxdh_initiate(params, rng)?;
    Ok((result.kyber_ciphertext, result.keys))
}

fn accept(params: &Self::RecipientParams<'_>) -> Result<Self::SessionSecret> {
    pqxdh_accept(params)
}
```

Therefore, any caller invoking `<Pqxdh as Handshake>::initiate(p, r)` is
behaviorally indistinguishable from a caller invoking
`pqxdh_initiate(p, r).map(|h| (h.kyber_ciphertext, h.keys))`, and similarly
for `accept`. The trait + impl carry no behavior beyond name-mangling and
return-tuple repackaging.

**Status:** `axiomatic` — verified by inspection of the source. The
extraction targets are the free functions `pqxdh_initiate` /
`pqxdh_accept`; the trait + impl are excluded from the LLBC, so no
verification claim references the trait's methods.

**Justification:** The impl's body is fully visible at `pqxdh.rs:49-59`
(13 lines, single-statement bodies). There is no possibility of hidden
behavior. Any future change to the impl that introduces new behavior
*must* be paired with a re-evaluation of this A-entry.

**Discharge:** This assumption is `revertible` — when AENEAS-001 is fixed
(GAT support in Aeneas), M02 + M03 revert, the trait + impl re-enter the
LLBC, and this A-entry no longer applies.

---

## A02 — `RecipientParameters` owned vs. borrowed `their_kyber_ciphertext`: cryptographic equivalence

**Invoked by:** M04 (RecipientParameters ownership rewrite).

**Statement:** For all values of the other seven fields of
`RecipientParameters`, and for any cryptographically-valid Kyber1024
ciphertext byte sequence *S* (a sequence of `KYBER_1024_CIPHERTEXT_LENGTH`
bytes), the recipient-side PQXDH key agreement
`pqxdh_accept(&params)` produces the same `HandshakeKeys` value when
`params.their_kyber_ciphertext` holds *S* — whether the field stores
`S` by owned `Box<[u8]>` or by borrowed `&'a [u8]`.

**Status:** `axiomatic`, `revertible`.

**Justification (3-layer argument):**

1. **Type-system invariant (compiler-enforced).** The `their_kyber_ciphertext`
   field is private. The sole accessor is
   `pub fn their_kyber_ciphertext(&self) -> &kem::SerializedCiphertext`,
   returning an immutable borrow. After M04, this accessor's signature is
   unchanged; only its body changes from `self.their_kyber_ciphertext`
   (deref of the borrow field) to `&self.their_kyber_ciphertext` (borrow
   of the owned field). External callers observe the same `&Box<[u8]>`
   either way. The Rust compiler enforces that no external code can
   distinguish the two representations through the type system.

2. **Byte-content invariant (semantic).** `kem::SerializedCiphertext` is
   defined at `rust/protocol/src/kem.rs:75` as
   `pub type SerializedCiphertext = Box<[u8]>`. Both `&Box<[u8]>` and
   `Box<[u8]>` yield the same `&[u8]` slice via auto-deref. The bytes
   read by `pqxdh_accept` to perform Kyber decapsulation
   (`pqxdh.rs:402-406`) and to feed into the HKDF input
   (`pqxdh.rs:391-398`) are identical: same address, same length, same
   contents. The cryptographic protocol depends only on byte content.

3. **Test-vector check (behavioral).** The existing upstream test suite
   (`rust/protocol/tests/ratchet.rs:30-260`,
   `rust/protocol/tests/support/mod.rs:160-200`) runs full PQXDH
   handshakes with real Kyber1024 ciphertexts and asserts that initiator
   and recipient derive identical session keys. Running this suite
   against the modified code (post-M04) and observing the same pass/fail
   outcomes is empirical evidence — not proof — that observable behavior
   is preserved.

**Scope (explicit non-coverage):**

- This assumption covers **functional cryptographic behavior** (derived
  keys, byte-equality of session secrets, success/failure outcomes).
- This assumption does **not cover**:
  - **Drop semantics.** The owned-field variant frees its 1.5KB heap
    allocation when the struct drops; the borrowed-field variant does
    not. Observable via heap profilers but irrelevant to the
    cryptographic property being verified.
  - **Side-channel behavior.** Memory-access patterns differ slightly
    (one extra deref level in the borrowed variant). Out of scope for a
    functional-correctness verification target. A side-channel-aware
    verification would need to revisit this assumption.
  - **Cargo-check-time errors at downstream callers.** Callers passing
    `&kyber_ciphertext` to `BobSignalProtocolParameters::new` must be
    updated to pass `kyber_ciphertext.clone()` or move ownership. These
    are cascade edits owned by M04, not assumptions.

**Discharge plan:**

- This assumption is `revertible`: when AENEAS-002 is fixed upstream
  (lifetime-parameterized struct support in Aeneas's `PureTypeCheck`),
  M04 reverts, the struct re-acquires `<'a>`, and this A-entry no
  longer applies. The verified Lean is then about the borrowed
  variant directly.
- Until then, the assumption stands. We accept it as `axiomatic`
  pending upstream fix. The 3-layer argument above is the audit trail
  a domain expert can verify.

---

## A03 — `libcrux_ml_kem::kyber1024::{encapsulate, decapsulate}` ≡ FIPS 203 ML-KEM-1024 primitive

**Invoked by:** M07 (vtable→match rewrite in `Key<Public>::encapsulate` and `Key<Secret>::decapsulate`), via the wrapper's call to `kyber1024::Parameters::encapsulate/decapsulate` which in turn calls `libcrux_ml_kem::kyber1024::{generate_key_pair, encapsulate, decapsulate}` (see `rust/protocol/src/kem/kyber1024.rs:27,38,51`).

**Statement:** The `libcrux_ml_kem::kyber1024` primitives implement ML-KEM-1024 (Module-Lattice-based Key-Encapsulation Mechanism) as specified by [NIST FIPS 203](https://csrc.nist.gov/pubs/fips/203/final). Specifically:

- `kyber1024::generate_key_pair(rng_bytes) → (sk, pk)` produces a key pair drawn from the ML-KEM-1024 key distribution.
- `kyber1024::encapsulate(&pk, rng_bytes) → (ct, ss)` produces a ciphertext `ct` and a shared secret `ss` such that `decapsulate(sk, ct) = ss` whenever `(sk, pk)` were produced together (correctness).
- `kyber1024::decapsulate(&sk, &ct) → ss` recovers the shared secret from a valid ciphertext using the matching secret key.
- IND-CCA2 security holds under the Module-LWE and Module-SIVP hardness assumptions, as established by the ML-KEM-1024 specification.

**Status:** `axiomatic`, **NOT** `revertible`.

**Justification:** ML-KEM-1024 is a cryptographic primitive at our verification scope. `libcrux_ml_kem` is a verified-Rust implementation (developed by Cryspen, with portions formally proved correct in F\* / EasyCrypt and others verified via fuzz testing against the FIPS 203 reference). Verifying the primitive itself is **out of scope** for this project — it would require either:

1. Re-doing the libcrux ML-KEM verification effort, OR
2. Bridging libcrux's existing F\* / EasyCrypt proofs into Lean

Both are major projects on their own. The trust boundary for libsignal-lite-verify is drawn at the libcrux interface: we assume the primitive is functionally correct (and IND-CCA2-secure), and verify PQXDH's *use* of it.

This assumption is the post-quantum analogue of A08 (x25519-dalek correctness, from the OLD repo's catalog). Both are "trust the underlying audited cryptographic implementation" boundaries.

**Discharge:** None planned for this project. If/when libcrux's internal verification produces Lean-importable lemmas (a long-term Cryspen goal), A03 could be downgraded from `axiomatic` to `discharged-by-import`. For now, the assumption stands as the boundary of our verification claims.

**Scope (explicit non-coverage):**

- Side-channel resistance of `libcrux_ml_kem` (constant-time execution) — out of scope.
- Protection against fault attacks — out of scope.
- The correctness of any *future* libcrux algorithm changes (e.g., the migration from Kyber-round3 to FIPS-203 finalized parameters) — this assumption pins to whatever libcrux version is in our Cargo.lock at the verification time.

---

## A04 — Direct-`match` dispatch ≡ `DynParameters` vtable dispatch (Kyber1024-only build)

**Invoked by:** M07 (vtable→match rewrite in `kem.rs:354-357` and `kem.rs:381-384`).

**Statement:** Under the project's default Cargo feature flags (no `kyber768`, no `mlkem1024`), `KeyType` has exactly one variant: `Kyber1024`. Consequently, the runtime vtable dispatch `key_type.parameters().encapsulate(...)` (where `key_type: KeyType` and `parameters() → &'static dyn DynParameters`) is **observationally equivalent** to the direct dispatch `match key_type { KeyType::Kyber1024 => kyber1024::Parameters::encapsulate(...) }`. Specifically:

- For all valid `KeyType` values, both forms select the same impl function (`<kyber1024::Parameters as Parameters>::encapsulate`).
- For all input `KeyMaterial<Public>` and CSPRNG state, both forms produce identical `Result<(SharedSecret, RawCiphertext), BadKEMKeyLength>` values byte-for-byte.
- The same statement holds for `decapsulate` (mutatis mutandis with `DecapsulateError`).

**Status:** `axiomatic`, `revertible` (when AENEAS-009 is fixed upstream).

**Justification:** Verified by inspection of the source:

1. **Single-variant enumeration.** Under default features, `KeyType` (`kem.rs:200-210`) has only the `Kyber1024` variant active — the other two arms are `#[cfg]`-gated to disabled features.
2. **Vtable resolves to single impl.** `KeyType::parameters()` (`kem.rs:226-234`) is a `match` whose body, after `cfg`-expansion, returns `&kyber1024::Parameters` unconditionally.
3. **The blanket impl is the only dispatch target.** `impl<T: Parameters> DynParameters for T` (`kem.rs:128-173`) implements each `DynParameters::*` method by delegating *directly* to `<T as Parameters>::*` (no additional logic — pure forwarding).
4. **The direct-match form expresses the composition explicitly.** `match self.key_type { KeyType::Kyber1024 => kyber1024::Parameters::encapsulate(...) }` is the exact unfolding of the vtable composition under (1)+(2)+(3).

**Scope (explicit non-coverage):**

- This assumption holds **only** under the default Cargo feature set. If `kyber768` or `mlkem1024` features are enabled, the direct-match form is incomplete (the additional arms would need to be added). Verification must be re-checked when features change.
- The behavior of the dyn-trait dispatch under future Rust compiler optimizations or codegen changes is irrelevant here — both forms are source-level constructs; semantic equivalence is at the source-AST level.

**Discharge:** This assumption is `revertible` — when AENEAS-009 is fixed upstream and M07 reverts (restoring the vtable dispatch), this A-entry no longer applies. The verified Lean is then about the dyn-trait dispatch form directly. Until then, A04 stands as the bridge between the verification target's source form and its original form.

---

## A05 — `pqxdh_accept` recipient-side correctness (axiomatic until body extracted)

**Invoked by:** M09 (config-only opaque marking of `libsignal_protocol::pqxdh::pqxdh_accept`).

**Statement:** For any `RecipientParameters` values `p` with internally-consistent pre-keys and any initiator-side state that produced a matching ciphertext, `pqxdh_accept(&p)` returns `Ok(HandshakeKeys { root_key, chain_key, pqr_key })` whose bytes are derivable by the same KDF (HKDF-SHA256 with label `"WhisperText_X25519_SHA-256_CRYSTALS-KYBER-1024"`) over the same byte sequence the initiator produced. In particular:

1. The four EC DH operations (`DH(SPK_b, IK_a)`, `DH(IK_b, EK_a)`, `DH(SPK_b, EK_a)`, optionally `DH(OPK_b, EK_a)`) yield the same bytes on the recipient and initiator sides under valid Curve25519 inputs.
2. The Kyber1024 decapsulation (`SK_b_kem.decapsulate(ct)`) yields the same shared secret as the initiator's encapsulation produced.
3. The HKDF output bytes match the initiator's HKDF input → output mapping byte-for-byte.

Therefore `pqxdh_accept(&p)` returns the same `HandshakeKeys` that the initiator computed via `pqxdh_initiate`. This is the standard PQXDH correctness theorem.

**Status:** `open` (placeholder until M09 reverts).

**Justification:** The body of `pqxdh_accept` is opaque to the extraction. The correctness claim cannot be discharged by symbolic execution of the extracted Lean body; it must either be:

1. Stated as a meta-theorem about the un-extracted Rust source, or
2. Verified after a future iteration that successfully extracts `pqxdh_accept`'s body (by isolating and rewriting the "no bottoms" trigger — see AENEAS-010).

Until then, this assumption is the **scope limiter** for any theorem that depends on the recipient-side flow.

**Discharge plan:**

- *Short-term:* isolate the AENEAS-010 trigger via body bisection; either rewrite the offending pattern or file upstream.
- *Once `pqxdh_accept` body is transparently extracted:* the correctness theorem becomes provable in Lean by symbolic equivalence to the initiator-side computation (matched DH and KEM operations + identical HKDF). The assumption then becomes a *Theorem*, not an axiom.

**Scope (explicit non-coverage):**

- This assumption covers *functional correctness* of the recipient flow. It does **not cover**:
  - Misuse-resistance (e.g., behavior when pre-keys are reused).
  - Side-channel behavior.
  - Authenticated key exchange security (separate game-based argument).

---

## Cross-reference index

When AENEAS-xxx issues are fixed upstream, the corresponding entries become
resolved:

| Upstream issue   | M-entries reverted | A-entries resolved |
|---|---|---|
| AENEAS-001 (GAT) | M02                | A01                |
| AENEAS-002 (lifetime-parameterized struct) | M04, M06 | A02 |
| AENEAS-004 (PrePasses on excluded impls)   | M03 | A01 (already covers M03) |
| AENEAS-009 (dyn Trait)                     | M05, M07, M08 | A04 |
| AENEAS-010 (no-bottoms in pqxdh_accept)    | M09 | A05 (placeholder — promotable to Theorem when M09 reverts) |
| *(never — primitive trust boundary)*        | — | A03 (libcrux ML-KEM correctness) |

The M01 charon-prelude is a permanent prerequisite of any
`charon::*` annotation in `rust/protocol/`; it reverts only when the last
such annotation is removed. A03 (ML-KEM correctness) never reverts — it
is the cryptographic-primitive trust boundary, axiomatic by construction.
