# Mímir Shielded Pool — Circuit Template

> **This is a template for Ian to build against, not a finished implementation.**
> The module layout + public inputs are the contract. The gadget internals are
> the parts you implement. `// TODO(ian)` marks your fill-in spots.

## Directory layout (per circuit, one Nargo package each)

```
src/
  poseidon.nr    — hash abstraction (implement this FIRST, thin wrapper)
  keys.nr        — key derivation (placeholder until BabyJubJub lands)
  note.nr        — Note struct: commitment() + nullifier()
  merkle.nr      — verify_membership(leaf, path, indices, root)
  deposit.nr     — main()
  transfer.nr    — main()
  withdraw.nr    — main()
```

## The primitives you're composing (the contract)

| Primitive | Signature you implement | Invariant it enforces |
|---|---|---|
| **Poseidon2** | `hash2(a,b)`, `hash3(a,b,c)` | collision-resistance on BN254; params cross-validated vs reference vectors |
| **Note commitment** | `commitment = hash3(value, owner_pubkey, blinding)` | hiding + binding; blinding is mandatory |
| **Nullifier** | `nullifier = hash2(owner_secret, commitment)` | one-to-one with a note; unlinkable without the secret |
| **Sparse Merkle** | `verify_membership(leaf, path, indices, root)` | binary tree, depth fixed, neighbor-hash = `hash2` |

## Circuit public inputs (the on-chain verifier sees ONLY these)

### deposit
```rust
pub fn main(
    value:            pub Field,   // amount shielded (public)
    commitment:       pub Field,   // commitment the contract inserts
    owner_secret:     Field,       // private
    blinding:         Field,       // private
) {
    // TODO(ian): assert commitment == hash3(value, derive_pubkey(owner_secret), blinding)
}
```

### transfer (2-in / 2-out)
```rust
pub fn main(
    root:             pub Field,            // a recent valid root
    nullifiers:       pub [Field; 2],       // spent inputs
    out_commitments:  pub [Field; 2],       // new outputs
    // -- private witness --
    in_values:        [Field; 2],
    in_secrets:       [Field; 2],
    in_blindings:     [Field; 2],
    in_paths:         [[Field; TREE_DEPTH]; 2],
    in_path_indices:  [[u1; TREE_DEPTH]; 2],
    out_values:       [Field; 2],
    out_pubkeys:      [Field; 2],
    out_blindings:    [Field; 2],
)
```
Three invariants you MUST assert:
1. membership of each input commitment against `root`
2. `nullifiers[i] == note.nullifier()` and `nullifiers[0] != nullifiers[1]`
3. `sum(in_values) == sum(out_values)`

### withdraw
```rust
pub fn main(
    root:         pub Field,
    nullifier:    pub Field,
    value:        pub Field,
    recipient:    pub Field,   // payout address, bound in-proof (anti-front-run)
    owner_secret: Field,
    blinding:     Field,
    path:         [Field; TREE_DEPTH],
    path_indices: [u1; TREE_DEPTH],
)
```

## Invariant checklist (the math you do NOT skip)

- [ ] `blinding` is sampled from a field distribution with ~254 bits entropy
- [ ] nullifier is a *function of the commitment*, never of `value` alone
- [ ] value conservation asserted as field equality (no silent wrap — flag)
- [ ] recipient bound into the withdraw proof (else relayer front-runs payout)
- [ ] distinct-nullifier check in transfer (no same-note double-spend in-tx)

## Known simplifications — decide these as you go (don't silently drop)

- `ISSUE-1` owner key = `Poseidon2(secret)` placeholder → BabyJubJub for ECDH memos
- `ISSUE-2` no encrypted note payload (recipient learns of notes out-of-band)
- `ISSUE-3` fixed tree depth 20
- `ISSUE-4` no range proofs (value ≤ 2^128 enforced on contract side, not circuit)

---

**How to use this:** fill the `TODO(ian)` bodies, keep the signatures exactly as
written (they're already the on-chain integration surface), and tick the
invariant checklist as you go. If you change a signature, say so — it ripples
into the Solidity verifier + SDK.