# Specification: Property-Based Testing for Monads in Rust

## 1. Introduction to Monads and Their Properties

A monad is a design pattern that allows for sequencing operations. In Rust, this
pattern is often seen with types like `Option<T>` and `Result<T, E>`, and can be
generalized to any type that supports a "bind" operation (often called
`and_then` or `flat_map`).

For a type to be considered a lawful monad, it must satisfy three specific laws:

1. **Left Identity:** `return(a).bind(f) == f(a)`
   - If you take a value `a`, put it into a default monadic context (`return`),
     and then apply a function `f` to it, the result should be the same as just
     applying `f` to `a`.

2. **Right Identity:** `m.bind(return) == m`
   - If you have a monadic value `m` and you apply the `return` function to it,
     the result should be the original monadic value `m`.

3. **Associativity:** `m.bind(f).bind(g) == m.bind(|x| f(x).bind(g))`
   - When you have a chain of two functions, it shouldn't matter how they are
     nested.

In the context of `proptest`, `return` is analogous to `Just`, and `bind` is
`prop_flat_map`.

## 2. Testing Monadic Laws with `proptest`

We can use `proptest` to generate arbitrary instances of our monadic type and
arbitrary functions to verify that these laws hold.

### 2.1. Project Setup

Ensure `proptest` is a dev-dependency in your `Cargo.toml`:

```toml
[dev-dependencies]
proptest = "1.9.0"
```

### 2.2. Example: Testing `Option<T>`

Let's verify the monadic laws for Rust's `Option<T>`.

#### 2.2.1. Defining Strategies

First, we need strategies to generate `Option<u32>` values and functions that
can be used in our tests.

```rust
use proptest::prelude::*;

// Strategy for generating an Option<u32>
fn arb_option_u32() -> impl Strategy<Value = Option<u32>> {
    prop_oneof![
        Just(None),
        any::<u32>().prop_map(Some),
    ]
}

// A function from u32 to Option<u32> for testing
// Note: proptest doesn't have a built-in way to generate random functions,
// so we create a few simple, deterministic ones to stand in.
fn f(x: u32) -> Option<u32> {
    Some(x.wrapping_add(1))
}

fn g(x: u32) -> Option<u32> {
    if x % 2 == 0 {
        Some(x.wrapping_mul(2))
    } else {
        None
    }
}
```

#### 2.2.2. Writing the Property Tests

Now, we can write the tests for the three monad laws.

```rust
use proptest::prelude::*;

// (Strategies and functions f, g from above)

proptest! {
    #[test]
    fn test_left_identity(a in any::<u32>()) {
        let return_a = Some(a); // `return` for Option is `Some`
        prop_assert_eq!(return_a.and_then(f), f(a));
    }

    #[test]
    fn test_right_identity(m in arb_option_u32()) {
        // `return` for Option is `Some`
        prop_assert_eq!(m.and_then(Some), m);
    }

    #[test]
    fn test_associativity(m in arb_option_u32()) {
        let lhs = m.and_then(f).and_then(g);
        let rhs = m.and_then(|x| f(x).and_then(g));
        prop_assert_eq!(lhs, rhs);
    }
}
```

## 3. The Cost of Monadic Composition in PBT

As detailed in the article "Demystifying monads in Rust through property-based
testing," the use of monadic composition (`prop_flat_map`) in `proptest` has a
significant performance drawback during the **shrinking** phase.

- **`prop_map` (Non-Monadic):** When a value is shrunk, it is simply passed
  through the mapping function. The structure of the value generation is
  preserved, and shrinking is efficient.

- **`prop_flat_map` (Monadic):** When a value is shrunk, the `flat_map` function
  is re-executed, generating a _completely new strategy_ and value tree. The
  shrinker has to start over from scratch within this new tree, leading to an
  exponential increase in the number of shrink attempts.

### 3.1. Practical Implications

- **Slow Feedback Loop:** Tests that rely heavily on `prop_flat_map` can take
  orders of magnitude longer to shrink a failing test case, slowing down the
  debugging process considerably.
- **Incomplete Shrinking:** Due to internal limits, `proptest` might give up on
  shrinking a complex monadic strategy before finding the minimal failing
  example.

### 3.2. Recommendations

1. **Avoid `prop_flat_map` When Possible:** Always look for a non-monadic way to
   structure your strategies. Combinators like `prop_map`, `prop_filter`,
   `prop_recursive`, and generating tuples are highly preferred.

2. **Isolate Monadic Composition:** If `prop_flat_map` is unavoidable, try to
   isolate it to the smallest possible part of your strategy. The less of the
   value tree that needs to be regenerated, the better.

3. **Example: Rewriting a Monadic Strategy**

   **Monadic (Slow):** Generate a number `n`, then generate a vector of that
   length.

   ```rust
   // This will be very slow to shrink.
   fn monadic_vec_strategy() -> impl Strategy<Value = Vec<u8>> {
       (0..100u8).prop_flat_map(|size| {
           proptest::collection::vec(any::<u8>(), size as usize)
       })
   }
   ```

   **Non-Monadic (Fast):** Generate a vector with a size range directly.

   ```rust
   // This is the idiomatic and efficient way.
   fn non_monadic_vec_strategy() -> impl Strategy<Value = Vec<u8>> {
       proptest::collection::vec(any::<u8>(), 0..100)
   }
   ```

## 4. Conclusion

Property-based testing is an excellent tool for verifying the correctness of
monadic structures by testing their fundamental laws. However, developers must
be acutely aware of the performance implications of monadic composition
(`prop_flat_map`) within the `proptest` framework. By favoring non-monadic
alternatives, we can ensure that our property tests remain fast and effective,
providing minimal failing cases that are easy to debug.
