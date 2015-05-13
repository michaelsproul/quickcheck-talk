% Testing Rust Code

Michael Sproul

May 19, 2015

# Overview

* Rust's built-in testing framework.
* Testing with invariant checking.
* Quickcheck.
* Benchmarking.

# How to Write Tests

Mark tests with `#[test]`.

If the test function doesn't panic, it passes.

```
#[test]
fn this_passes() {}

#[test]
fn this_fails() {
    panic!();
}

#[test]
fn this_also_fails() {
    assert!(true); // fine
    assert_eq!(2, 5); // uh oh
}
```

# Compiling a Test Harness

Test binaries (harnesses) are compiled via Rust's [conditional
compilation][cond-comp] facilities. Minimal magic!

```shell
$ rustc --cfg test test.rs
```

Generally you just let Cargo take care of this.

```
$ cargo test
```

# Test Output

```small
$ cargo test
running 3 tests
test this_fails ... FAILED
test this_also_fails ... FAILED
test this_passes ... ok

failures:

---- this_fails stdout ----
	thread 'this_fails' panicked at 'explicit panic', test.rs:6

---- this_also_fails stdout ----
	thread 'this_also_fails' panicked at 'assertion failed: `(left == right)` (left: `2`, right: `5`)', test.rs:12

failures:
    this_also_fails
    this_fails

test result: FAILED. 1 passed; 2 failed; 0 ignored; 0 measured
```

# Standard Testing Idioms

* Take advantage of conditional compilation at the module level.
* Put tests in a separate file when they become numerous.

```
#[cfg(test)]
mod test {
    use super::*;

    fn make_something() -> Thing { ... }

    #[test]
    fn test_one() {
        let x = make_something();
        ...
    }

    ...
}
```

# What to Test?

Rust provides some pretty strong guarantees about memory safety, but there are
plenty of other ways to go wrong.

[[ example using Option types here, with Python equiv ]]

[cond-comp]: http://doc.rust-lang.org/reference.html#conditional-compilation
