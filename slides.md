% Testing and QuickCheck

Rust 1.0 Testing Techniques

Michael Sproul

May 19, 2015

# Overview

* Rust's built-in testing framework.
* Generating tests with QuickCheck.

# How to Write Tests

Mark tests with the `#[test]` [attribute][attributes].

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

[attributes]: http://doc.rust-lang.org/reference.html#attributes

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

[cond-comp]: http://doc.rust-lang.org/reference.html#conditional-compilation

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
}
```

# What to Test?

Rust provides some pretty strong guarantees about memory safety, but there are
plenty of other ways to go wrong.

* Code inside `unsafe` blocks can violate memory safety.
* Panicking crashes the whole program (via `unwrap`, array indexing, etc).
* **Logic Errors**.

# Preventing Logic Errors

Programming is all about invariants.

**Testing allows us to check invariants that can't be expressed in the type system.**

# Example: Lottery Scheduling

**Basic idea:** Assign processes "tickets" in a lottery and schedule processes
based on the outcome of the lottery. Reward good behaviour and penalise bad
behaviour.

Here we consider only the basic data structures.

```
pub struct Lottery {
    /// The total number of tickets.
    pub total_tickets: usize,
    /// The number of tickets each process has, such that process i's
    /// ticket count is tickets[i].
    pub tickets: Vec<usize>
}

pub enum Transaction {
    /// Increase the given player's ticket count by 1.
    Reward(usize),
    /// Decrease the given player's ticket count by 1.
    Penalise(usize)
}
```

# Lottery Scheduling Invariants

To keep scheduling somewhat fair we place an upper and lower limit on the
number of tickets any process can have.

```
pub const MIN_TICKETS: usize = 1;
pub const MAX_TICKETS: usize = 50;
```

* ∀t ∈ `tickets` : t ≥ `MIN_TICKETS` and t ≤ `MAX_TICKETS`.
* `total_tickets` = sum(`tickets`).

We'd like to verify that these invariants are maintained for arbitrary
sequences of transactions.

# QuickCheck

QuickCheck is a library that can generate random input to a function **based on
its type signature**.

```
/// Claim that all vectors have even length...
fn prop_even_length(v: Vec<u32>) -> bool {
    v.len() % 2 == 0
}
```

Given a "property function", QuickCheck will generate input for it and check that it succeeds.

If the function fails, QuickCheck can **shrink** the failing input to produce a minimal test case!

In the case above, QuickCheck finds `[0]`.

# Testing the Lottery Scheduler with QuickCheck

Let's use QuickCheck to test the lottery scheduler.

**Plan of attack**: Generate random vectors of transactions and apply them.
After the application of each transaction, check the integrity of the lottery
using an invariant checking function.

```
/// Invariant checking function.
impl Lottery {
    fn is_valid(&self) -> bool {
        // check sum and min/max bounds here.
    }
}

```

# Testing the Lottery Scheduler with QuickCheck

```
/// QuickCheck property function.
fn lottery_check(transactions: Vec<Transaction>) -> bool {
    let mut lottery = Lottery::test_lottery(5);

    for t in transactions {
        lottery.apply_transaction(t);

        if !lottery.is_valid() {
            return false;
        }
    }

    true
}
```

# Generating Transactions

How does QuickCheck know how to generate a `Vec<Transaction>`? We have to tell it!

The `Arbitrary` trait describes types which can be randomly generated given a
random number generator.

```
impl Arbitrary for Transaction {
    fn arbitrary<G: Gen>(g: &mut G) -> Transaction {
        let reward = g.gen();
        let player = g.gen::<usize>() % TEST_NUM_PLAYERS;

        if reward {
            Reward(player)
        } else {
            Penalise(player)
        }
    }
}
```

QuickCheck can handle `Vec<A: Arbitrary>` out of the box.

# Running QuickCheck

QuickCheck integrates seamlessly with Rust's testing framework!

```
#[test]
fn test_transactions() {
    /// QuickCheck property fn from before.
    fn lottery_check(transactions: Vec<Transaction>) -> bool { ... }

    quickcheck(lottery_check as fn(Vec<Transaction>) -> bool);
}
```

Note: The function type cast should be unnecessary in a future version of QuickCheck.

# It Works!

In a lottery where every player has 5 tickets initially...

**This is what happens if we forget to enforce the lower bound:**

```
[quickcheck] TEST FAILED. Arguments: ([Penalise(0), Penalise(0), Penalise(0), Penalise(0), Penalise(0)])
```

**This is what happens if we forget to enforce the upper bound:**

```
[quickcheck] TEST FAILED. Arguments: ([Reward(4), ... x46 ])
```

In both cases QuickCheck finds a minimal test case!

# Advanced QuickCheck

* By default QuickCheck only checks relatively small inputs. To find the bug
  with the vector of length 46 I had to tweak some settings.

```
let gen = StdGen::new(rand::thread_rng(), 10_000);
let mut qc = QuickCheck::new().gen(gen);
```

* It's possible to filter the randomly generated input.

# Further Reading

The full code for the lottery scheduler example is up on Github:

https://github.com/michaelsproul/quickcheck-example

See also:

* [QuickCheck Source and Documentation][quickcheck]
* [Testing | The Rust Programming Language][trpl]

[quickcheck]: https://github.com/BurntSushi/quickcheck
[trpl]: http://doc.rust-lang.org/book/testing.html

# Thanks!
