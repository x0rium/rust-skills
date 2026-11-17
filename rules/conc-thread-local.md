# conc-thread-local

> Prefer `thread_local!` with `Cell`/`RefCell` over `static mut`

## Why It Matters

`static mut` requires `unsafe` at every access and is undefined behavior if any two threads touch it simultaneously — the compiler cannot rule that out. In Rust 2024, taking a shared or mutable reference to a `static mut` is rejected by the deny-by-default `static_mut_refs` lint. `thread_local!` gives each thread its own independent copy of the value, accessed through safe APIs via `Cell` (for `Copy` types) or `RefCell` (for anything else), with no synchronization overhead and no unsafe code needed.

## Bad

```rust
// Rust 2024: referencing static mut is denied by default (static_mut_refs lint)
static mut BUFFER: Vec<u8> = Vec::new();

fn append_to_buffer(data: &[u8]) {
    // UB if called from multiple threads; `static_mut_refs` denies this in 2024
    unsafe {
        BUFFER.extend_from_slice(data);
    }
}

fn flush_buffer() -> Vec<u8> {
    unsafe {
        std::mem::take(&mut BUFFER) // still requires unsafe
    }
}
```

## Good

```rust
use std::cell::RefCell;

thread_local! {
    static BUFFER: RefCell<Vec<u8>> = RefCell::new(Vec::with_capacity(4096));
}

fn append_to_buffer(data: &[u8]) {
    BUFFER.with_borrow_mut(|buf| buf.extend_from_slice(data));
}

fn flush_buffer() -> Vec<u8> {
    BUFFER.with_borrow_mut(|buf| std::mem::take(buf))
}
```

For `Copy` types, `Cell` is simpler (no borrow overhead):

```rust
use std::cell::Cell;

thread_local! {
    // `const { }` initializer: no lazy-init check on each access
    static CALL_COUNT: Cell<u32> = const { Cell::new(0) };
}

fn record_call() {
    CALL_COUNT.update(|c| c + 1); // LocalKey<Cell<T>>::update, stable since 1.99
}

fn get_call_count() -> u32 {
    CALL_COUNT.get()
}
```

## Key Points

- `with_borrow` / `with_borrow_mut` are stable convenience methods (since 1.73) on `LocalKey<RefCell<T>>` — prefer them over the longer `with(|v| v.borrow_mut())` form.
- `LocalKey<Cell<T>>` has direct `get` / `set` / `take` / `replace` (since 1.73) and `update` (since 1.99) — no `with(|c| ...)` closure needed for simple counters and flags.
- Thread-local destructors run when the thread exits, so cleanup happens automatically.
- Avoid storing thread-locals in types that are sent across threads; the value is strictly per-thread and is not visible to other threads.
- For read-only global constants shared across threads, use `static` with an immutable type instead.

## When to Use

- Scratch buffers, caches, or accumulators that are logically per-thread and don't need to be shared
- Reusing allocations across calls within a single thread (e.g., formatting buffers, temporary vecs)
- Replacing `unsafe static mut` with a safe alternative

## See Also

- [own-refcell-interior](own-refcell-interior.md) - interior mutability for single-threaded code
- [own-mutex-interior](own-mutex-interior.md) - shared mutable state across threads requires `Mutex`
