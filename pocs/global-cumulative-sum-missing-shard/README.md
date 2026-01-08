# PoC: Global cumulative sum detects missing shard

This PoC demonstrates that the core proof verifier enforces global bus closure by checking that:

`vk.initial_global_cumulative_sum + Σ shard_proof.global_cumulative_sum == 0`.

Dropping a shard makes this sum non-zero and verification fails with
`NonZeroCumulativeSum(InteractionScope::Global, ...)`.

## Run

```bash
cargo test -p sp1-prover test_global_cumulative_sum_detects_missing_shard -- --nocapture
```

## Code

- Test: `crates/prover/src/lib.rs`
- Global sum check: `crates/stark/src/machine.rs`

