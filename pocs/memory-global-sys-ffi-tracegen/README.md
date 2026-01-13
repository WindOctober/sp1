# PoC: MemoryGlobal `sys` FFI tracegen divergence

This PoC demonstrates that `feature="sys"`'s C++ row writer for `MemoryGlobal` does **not** populate
the auxiliary columns that Rust tracegen populates (`lt_cols`, `is_next_comp`, `is_prev_addr_zero`,
`is_first_comp`, `is_last_addr`), producing a trace that differs from Rust tracegen.

## Run

End-to-end (AIR constraint check: Rust trace satisfies AIR, FFI trace fails; expected to exit non-zero):

```bash
cargo test -p sp1-core-machine --features sys \
  test_sys_memory_global_ffi_trace_violates_air -- --ignored --nocapture
```

## Code

- Test: `crates/core/machine/src/memory/global.rs`
- C++ row writer: `crates/core/machine/include/memory_global.hpp`
- FFI export: `crates/core/machine/cpp/extern.cpp`

## C++ Row Writer (Excerpt)

The `sys` C++ helper only fills the “base fields” (`addr/addr_bits/shard/timestamp/value/is_real`) and
does not set the derived/aux columns that Rust tracegen sets (`lt_cols`, `is_next_comp`,
`is_prev_addr_zero`, `is_first_comp`, `is_last_addr`):

```cpp
// crates/core/machine/include/memory_global.hpp
namespace sp1_core_machine_sys::memory_global {
    template<class F, class EF7>
    __SP1_HOSTDEV__ void event_to_row(const MemoryInitializeFinalizeEvent* event, const bool is_receive, MemoryInitCols<F>* cols) {
        cols->addr = F::from_canonical_u32(event->addr);
        for(uintptr_t i = 0 ; i < 32 ; i++) {
            cols->addr_bits.bits[i] = F::from_canonical_u32(((event->addr) >> i) & 1);
        }
        cols->shard = F::from_canonical_u32(event->shard);
        cols->timestamp = F::from_canonical_u32(event->timestamp);
        for(uintptr_t i = 0 ; i < 32 ; i++) {
            cols->value[i] = F::from_canonical_u32(((event->value) >> i) & 1);
        }
        cols->is_real = F::one();
    }
}
```

## PoC Test Code

This PoC is an end-to-end constraint failure: it shows that the sys FFI row-writer can produce a
matrix-shaped trace, but that trace fails the `MemoryGlobal` AIR constraints (so proving with it is
impossible).

```rust
#[cfg(feature = "sys")]
#[test]
#[ignore = "expected to fail: demonstrates sys FFI trace violates MemoryGlobal AIR constraints"]
fn test_sys_memory_global_ffi_trace_violates_air() {
    // Generates a valid Rust trace and checks it satisfies the `MemoryGlobal` AIR,
    // then generates a trace via `crate::sys::memory_global_event_to_row_babybear` and
    // shows it fails `sp1_stark::debug_constraints`.
}
```
