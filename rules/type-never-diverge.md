# type-never-diverge

> Use `!` (never type) for functions that never return

## Why It Matters

The never type `!` indicates a function will never return normally—it either loops forever, panics, or exits the process. This helps the compiler understand control flow and enables `!` to coerce to any type, making it useful in match arms and expressions.

## Bad

```rust
// Return type doesn't indicate non-returning
fn infinite_loop() {
    loop {
        process_events();
    }
    // Implicit () return type, but never returns
}

// Using Option when it always panics
fn unreachable_code() -> Option<()> {
    panic!("This should never be called");
}
```

## Good

```rust
// ! indicates function never returns
fn infinite_loop() -> ! {
    loop {
        process_events();
    }
}

fn abort_with_error(msg: &str) -> ! {
    eprintln!("Fatal error: {}", msg);
    std::process::exit(1);
}

fn panic_handler() -> ! {
    panic!("Unexpected state");
}
```

## Coercion to Any Type

```rust
// ! coerces to any type
fn get_value(opt: Option<i32>) -> i32 {
    match opt {
        Some(v) => v,
        None => panic!("No value"),  // panic! returns !, coerces to i32
    }
}

// Useful in Result handling
fn must_get_config() -> Config {
    match load_config() {
        Ok(c) => c,
        Err(e) => {
            log_error(&e);
            std::process::exit(1)  // Returns !, coerces to Config
        }
    }
}
```

## Standard Library Examples

```rust
// std::process::exit
pub fn exit(code: i32) -> !

// panic! macro
// Expands to an expression of type !

// std::hint::unreachable_unchecked
pub unsafe fn unreachable_unchecked() -> !

// loop {} with no break
fn forever() -> ! {
    loop {}
}
```

## In Match Expressions

```rust
enum State {
    Running,
    Stopped,
    Error,
}

fn get_status(state: &State) -> &str {
    match state {
        State::Running => "running",
        State::Stopped => "stopped",
        State::Error => unreachable!(),  // ! coerces to &str
    }
}

// With Result
fn process(r: Result<Data, Error>) -> Data {
    match r {
        Ok(d) => d,
        Err(e) => panic!("Unexpected error: {}", e),  // ! coerces to Data
    }
}
```

## Diverging Closures

```rust
// Closures that never return
let handler: fn() -> ! = || {
    panic!("Handler called");
};

// In thread spawn
std::thread::spawn(|| -> ! {
    loop {
        process_work();
    }
});
```

## Stability Notes

`fn f() -> !` (never type as a return type) has been stable since Rust 1.0.

Since **Rust 1.100** the never type is fully stable: `!` can be used anywhere a type can — as a type argument, in a `let` annotation, in a struct field. Use `Result<T, !>` for an operation that structurally cannot fail; because the `Err` variant is uninhabited, the pattern `Ok(v)` is irrefutable and no `unwrap()` is needed:

```rust
// Stable since 1.100: ! as a type argument
fn parse_lenient(s: &str) -> Result<usize, !> {
    Ok(s.trim().len())
}

fn use_it() -> usize {
    let Ok(n) = parse_lenient(" abc ");  // irrefutable — Err(!) can't exist
    n
}
```

`std::convert::Infallible` is now a type alias for `!`, so existing code that uses `Infallible` (e.g. `impl FromStr` with `type Err = Infallible`) keeps working unchanged. In new code, write `!` directly; keep `Infallible` only if your MSRV is below 1.100.

**Compatibility note (1.100):** never-type fallback now resolves to `!` on all editions (previously `()` before the 2024 edition). Code where an unconstrained diverging expression silently became `()` — e.g. `let x = panic!(); x.method()` or `?`/`return` feeding into a generic call — may now fail to compile; add an explicit type annotation.

## See Also

- [err-result-over-panic](./err-result-over-panic.md) - When to panic vs return Result
- [type-result-fallible](./type-result-fallible.md) - Result for errors
- [opt-cold-unlikely](./opt-cold-unlikely.md) - Marking unlikely paths
