# PoC: MemoryGlobal `sys` FFI tracegen divergence

This PoC demonstrates that `feature="sys"`'s C++ row writer for `MemoryGlobal` does **not** populate
the auxiliary columns that Rust tracegen populates (`lt_cols`, `is_next_comp`, `is_prev_addr_zero`,
`is_first_comp`, `is_last_addr`), producing a trace that differs from Rust tracegen.

## Run

```bash
cargo test -p sp1-core-machine --features sys \
  test_memory_global_event_to_row_ffi_is_incomplete_vs_rust_tracegen -- --nocapture
```

## Code

- Test: `crates/core/machine/src/memory/global.rs`
- C++ row writer: `crates/core/machine/include/memory_global.hpp`
- FFI export: `crates/core/machine/cpp/extern.cpp`

## PoC Test Code

This PoC is a *differential* test: it asserts that the FFI row-writer output does **not** match
Rust tracegen (so the test *passing* means the mismatch exists).

```rust
#[cfg(feature = "sys")]
#[test]
fn test_memory_global_event_to_row_ffi_is_incomplete_vs_rust_tracegen() {
    use std::borrow::BorrowMut;

    use rand::{thread_rng, Rng};
    use sp1_core_executor::{events::MemoryInitializeFinalizeEvent, ExecutionRecord};

    fn generate_trace_ffi(
        events_sorted_by_addr: &[MemoryInitializeFinalizeEvent],
        padded_nb_rows: usize,
        is_receive: bool,
    ) -> RowMajorMatrix<BabyBear> {
        let mut values = vec![BabyBear::zero(); padded_nb_rows * NUM_MEMORY_INIT_COLS];
        for (row_idx, event) in events_sorted_by_addr.iter().enumerate() {
            let row = &mut values[row_idx * NUM_MEMORY_INIT_COLS..(row_idx + 1) * NUM_MEMORY_INIT_COLS];
            let cols: &mut MemoryInitCols<BabyBear> = row.borrow_mut();
            unsafe {
                crate::sys::memory_global_event_to_row_babybear(event, is_receive, cols);
            }
        }
        RowMajorMatrix::new(values, NUM_MEMORY_INIT_COLS)
    }

    let mut events: Vec<MemoryInitializeFinalizeEvent> = (0..8)
        .map(|_| MemoryInitializeFinalizeEvent {
            addr: thread_rng().gen_range(0..BabyBear::ORDER_U32),
            value: thread_rng().gen(),
            shard: thread_rng().gen_range(0..BabyBear::ORDER_U32),
            timestamp: thread_rng().gen_range(0..BabyBear::ORDER_U32),
        })
        .collect();
    events.sort_by_key(|e| e.addr);
    events.dedup_by_key(|e| e.addr);
    assert!(events.len() >= 2);

    let mut record = ExecutionRecord::default();
    record.global_memory_initialize_events = events.clone();
    record.global_memory_finalize_events = events.clone();

    for (kind, is_receive) in [(MemoryChipType::Initialize, false), (MemoryChipType::Finalize, true)] {
        let chip = MemoryGlobalChip::new(kind);
        let rust_trace: RowMajorMatrix<BabyBear> =
            chip.generate_trace(&record, &mut ExecutionRecord::default());

        let events_sorted_by_addr = events.as_slice();
        let ffi_trace = generate_trace_ffi(events_sorted_by_addr, rust_trace.height(), is_receive);

        // Passing this assertion means: FFI output differs from Rust (i.e. the mismatch exists).
        assert_ne!(ffi_trace, rust_trace);
    }
}
```
