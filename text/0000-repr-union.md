- Feature Name: `repr_union`
- Start Date: 2015-12-29
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Provide native support for C-compatible unions, defined via `#[repr(union)]
struct`.

# Motivation
[motivation]: #motivation

Many FFI interfaces include unions.  Rust does not currently have any native
representation for unions, so users of these FFI interfaces must define
multiple structs and transmute between them via `std::mem::transmute`.  The
resulting FFI code must carefully understand platform-specific size and
alignment requirements for structure fields.  Such code has little in common
with how a C client would invoke the same interfaces.

Introducing native syntax for unions makes many FFI interfaces much simpler and
less error-prone to write, simplifying the creation of bindings to native
libraries, and enriching the Rust/Cargo ecosystem.

The syntax proposed here avoids reserving a new keyword (such as `union`), and
thus will not break any existing code.  To preserve memory safety, accesses to
union fields may only occur in `unsafe` code.

# Detailed design
[design]: #detailed-design

## Declaring a union type

A union declaration uses identical syntax to a `struct` declaration, with the
addition of a `#[repr(union)]` tag.  `#[repr(union)]` implies `#[repr(C)]` as
the default representation, making `#[repr(C,union)]` permissible but
redundant.

```rust
#[repr(union)] struct MyUnion {
    f1: u32,
    f2: f32,
}
```

## Instantiating a union

A union instantiation uses the same syntax as a struct instantiation, except
that it must specify exactly one field:

```rust
let u = MyUnion { f1: 1 };
```

Specifying multiple fields in a union instantiation results in a compiler
error.

Safe code may instantiate a union, as no unsafe behavior can occur until
accessing a field of the union.  Code that wishes to maintain invariants about
the union fields should make the union fields private and provide public
functions that maintain the invariants.

## Reading fields

Unsafe code may read from union fields, using the same dotted syntax as a
struct:

```rust
fn f(u: MyUnion) -> f32 {
    unsafe { u.f2 }
}
```

## Writing fields

Unsafe code may write to fields in a mutable union, using the same syntax as a
struct:

```rust
fn f(u: &mut MyUnion) {
    unsafe {
        u.f1 = 2;
    }
}
```

If a union contains multiple fields of different sizes, assigning to a field
smaller than the entire union must not change the memory of the union outside
that field.

## Pattern matching

Unsafe code may pattern match on union fields, using the same syntax as a
struct, without the requirement to mention every field of the union in a match
or use `..`:

```rust
fn f(u: MyUnion) {
    unsafe {
        match u {
            MyUnion { f1: 10 } => { println!("ten"); }
            MyUnion { f2 } => { println!("float {}", f2); }
        }
    }
}
```

Matching a specific value from a union field makes a refutable pattern; naming
a union field without matching a specific value makes an irrefutable pattern.
Both require unsafe code.

A pattern match may match multiple fields of a union at once.  For rationale,
consider a union using the low bits of an aligned pointer as a tag; a pattern
match may match the tag using one field and a value identified by that tag
using another field.

Note that a pattern match on a union field that has a smaller size than the
entire union must not make any assumptions about the value of the union's
memory outside that field.

## Union and field visibility

The `pub` keyword works on the union and its fields as with a struct.  The
union and its fields default to private.  Using a private field in a union
instantiation, field access, or pattern match, produces an error.

## Uninitialized unions

The compiler should consider a union uninitialized if declared without an
initializer.  However, providing a field during instantiation or assigning to a
field should cause the compiler to treat the entire union as initialized.

## Unions and traits

A union may have trait implementations, using the same syntax as a struct.

The compiler should warn if a union field has a type that implements the `Drop`
trait.

## Transmuting through unions

Rust code may access a different field than the one assigned.  This has the
same semantics as `std::mem::transmute`:

```rust
unsafe fn transmute<A, B>(e: A) -> B {
    #[repr(union)] struct Temp { a: A, b: B };
    (Temp { a: e }).b
}
```

## Union size and alignment

A union must have the same size and alignment as an equivalent C union
declaration for the target platform.  Typically, a union would have the maximum
size of any of its fields, and the maximum alignment of any of its fields.
Note that those maximums may come from different fields; for instance:

```rust
#[repr(union)] struct U {
    f1: u16,
    f2: [u8; 4],
}

#[test]
fn test() {
    assert_eq!(std::mem::size_of<U>(), 4);
    assert_eq!(std::mem::align_of<U>(), 2);
}
```

# Drawbacks
[drawbacks]: #drawbacks

Adding a new type of data structure would increase the complexity of the
language and the compiler implementation, albeit marginally.  However, this
change seems likely to provide a net reduction in the quantity and complexity
of unsafe code.

# Alternatives
[alternatives]: #alternatives

[Discussion on rust-internals](https://internals.rust-lang.org/t/pre-rfc-unsafe-enums-now-including-a-poll/2873)

- Don't do anything, and leave users of FFI unions to continue writing complex
  platform-specific transmute code.
- Create macros to define unions and access their fields.  However, such macros
  make field accesses and pattern matching look more cumbersome and less
  structure-like.  The implementation and use of such macros provides strong
  motivation to seek a better solution, and indeed existing writers and users
  of such macros have specifically requested native syntax in Rust.
- Use a new keyword `union` rather than `#[repr(union)] struct`.  While that
  would make union declarations clearer, reserving `union` as a keyword would
  break existing code, including [multiple functions in the standard
  library](https://doc.rust-lang.org/std/?search=union).  Given that unions
  represent inherently unsafe functionality, intended primarily for use with
  FFI, reserving `union` as a keyword does not seem appropriate.
- The [unsafe enum](https://github.com/rust-lang/rfcs/pull/724) proposal:
  introduce untagged enums, identified with `unsafe enum`.  Pattern-matching
  syntax would make field accesses significantly more verbose than structure
  field syntax.
- The [unsafe enum](https://github.com/rust-lang/rfcs/pull/724) proposal with
  the addition of struct-like field access syntax.  The resulting field access
  syntax would look much like this proposal; however, pairing an enum-style
  definition with struct-style usage seems confusing for users.  An enum-based
  declaration leads users to expect enum-like syntax; a struct-based
  declaration leads users to expect struct-like syntax.

# Unresolved questions
[unresolved]: #unresolved-questions
