- Feature Name: N/A
- Start Date: 2022-01-19
- RFC PR: [rust-lang/rfcs#3373](https://github.com/rust-lang/rfcs/pull/3373)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Starting in Rust 2024, stop allowing items inside functions to implement
methods or traits that are visible outside the function.

# Motivation
[motivation]: #motivation

Currently, tools cross-referencing uses and definitions (such as IDEs) must
either search inside all function bodies to find potential definitions
corresponding to uses within a function, or not cross-reference those
definitions at all.

Humans cross-referencing such uses and definitions may find themselves
similarly baffled.

With this change, both humans and tools can limit the scope of their search and
avoid looking for definitions inside other functions, without missing any
relevant definitions.

# Explanation
[explanation]: #explanation

Starting in the Rust 2024 edition:
- An item nested inside a function or closure (through any level of nesting)
  may not define an `impl Type` block unless the `Type` is also nested inside
  the same function or closure.
- An item nested inside a function or closure (through any level of nesting)
  may not define an `impl Trait for Type` unless either the `Trait` or the
  `Type` is also nested inside the same function or closure.
- An item nested inside a function or closure (through any level of nesting)
  may not define an exported macro visible outside the function or closure
  (e.g. using `#[macro_export]`).

Rust 2015, 2018, and 2021 continue to permit this, but will produce a
warn-by-default lint.

No other language features provide a means of defining a name inside a function
and referencing that name outside the function.

# Drawbacks
[drawbacks]: #drawbacks

Some existing code makes use of this pattern, and would need to migrate to a
different pattern. In particular, this pattern may occur in macro-generated
code, or in code generated by tools like rustdoc. Making this change would
require such code and tools to restructure to meet this requirement.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

We'll need a crater run to look at how widespread this pattern is in existing
code.

# Future possibilities
[future-possibilities]: #future-possibilities

If in the future Rust provides a "standalone `derive`" mechanism (e.g. `derive
Trait for Type` as a standalone definition separate from `Type`), the `impl`
produced by that mechanism would be subject to the same requirements.