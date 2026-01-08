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

