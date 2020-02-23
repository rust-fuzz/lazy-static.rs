resettable-lazy-static.rs
=========================

This is a "resettable" version of [`lazy_static`](https://github.com/rust-lang-nursery/lazy-static.rs), a macro for declaring lazily evaluated statics in Rust.

This version adds a function `reset` that puts all `lazy_static` variables back in their original, uninitialized states. Such a function facilitates fuzzing with [AFL](http://lcamtuf.coredump.cx/afl/) in persistent mode.

**WARNING**: The caller of `reset` is responsible for ensuring that there are no outstanding references to any `lazy_static` variables. Similarly, the caller must not try to initialize a `lazy_static` variable concurrently with a call to `reset`, as this presents a data race.

# Motivation

In persistent mode, AFL does not launch a new process for each fuzzing run. Rather, the process [loops](https://github.com/rust-fuzz/afl.rs/blob/4e2b8a79a89df9adc9722c954ebc5c9529dfebef/src/lib.rs#L153-L175), repeatedly executing the function under test with new fuzzing inputs. This can cause problems for programs that use `lazy_static` variables in at least two ways:

1. AFL can give incorrectly low stability reports. Roughly speaking, stability is the fraction of instructions that are executed the same number of times across different runs of the program with the same input. Because a `lazy_static` variable's initializer is executed only on the variable's first use, the initializer's instructions are counted against stability, causing it to be low.

2. AFL can fail to report timeouts. Suppose that a fuzzing input X causes a program to use a `lazy_static` variable with an expensive initializer. If the variable was used for an earlier fuzzing input, then the variable's intializer will not be run on input X, causing the program to take less time than it otherwise would on input X.

Returning each `lazy_static` variable to its uninitialized state at the end of each fuzzing iteration addresses these problems. This causes each `lazy_static` variable's initializer to be executed for each fuzzing input that uses that variable. Thus, the initializer's instructions are not counted against stability. On the other hand, the time taken to execute the initializer is counted against each input that uses that variable.

# How it works

Conceptually, this version of `lazy_static` differs from the original in two ways:

* It collects all initialized `lazy_static` variables in a linked list.

* It wraps each `lazy_static` variable's `Once` instance in a `Cell`.

The function `reset` walks the linked list in order to set the initialized `lazy_static` variables back to their uninitialized states, assigning `ONCE_INIT` to each `Cell<Once>` in the process.

# Example

```rust
use lazy_static::lazy_static;
use std::sync::Mutex;

lazy_static! {
    static ref M: Mutex<String> = Mutex::new("foo".to_string());
}

fn main() {
    {
        let mut s = M.lock().unwrap();

        assert_eq!(*s, "foo".to_string());

        *s = String::from("bar");

        assert_eq!(*s, "bar".to_string());
    }

    assert!(M.try_lock().is_ok());

    unsafe {
        lazy_static::lazy::reset();
    }

    {
        let s = M.lock().unwrap();

        assert_eq!(*s, "foo".to_string());
    }
}
```

## License

Licensed under either of

 * Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
 * MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any
additional terms or conditions.
