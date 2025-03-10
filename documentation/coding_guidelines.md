---
id: coding-guidelines
title: Coding Guidelines
---

# Libra Core coding guidelines

This document describes the coding guidelines for the Libra Core Rust codebase.  

## Code formatting

All code formatting is enforced with [rustfmt](https://github.com/rust-lang/rustfmt) with a project specific configuration.  Below is an example command to adhere to the project conventions.

```
libra/libra$ cargo fmt
```

## Code analysis

[Clippy](https://github.com/rust-lang/rust-clippy) is used to catch common mistakes and is run as a part of continuous integration.  Before submitting your code for review, you can run clippy with our configuration: 

```
libra$ ../setup_scripts/clippy.sh 
```

In general, we follow the recommendations from [rust-lang-nursery](https://rust-lang-nursery.github.io/api-guidelines/about.html).  The remainder of the guide is to provide more detailed guidance on specific areas to provide as much uniformity as possible.

 
## Code documentation
 
Any public fields, functions, and methods should be documented with [Rustdoc](https://doc.rust-lang.org/book/ch14-02-publishing-to-crates-io.html#making-useful-documentation-comments).
 
 Please follow the conventions as detailed below for modules, structs, enums, and functions.  The *single line* is used as a preview when navigating Rustdoc.  As an example, see the 'Structs' and 'Enums' sections in the [collections](https://doc.rust-lang.org/std/collections/index.html) Rustdoc.
 
 ```
 /// [Single line] One line summary description
 ///
 /// [Longer description] Multiple lines, inline code 
 /// examples, invariants, purpose, usage, etc. 
 [Attributes] If attributes exist, add after Rustdoc
 ```

Example below:

```
/// Represents (x, y) of a 2-dimensional grid
///
/// A line is defined by 2 instances.
/// A plane is defined by 3 instances.
#[repr(C)]
struct Point {
    x: i32,
    y: i32,
}
```

### Constants and fields

Describe the purpose and definition of this data.

### Functions and methods

Document the following for each function:

* The action the method performs - “This method *adds* a new transaction to the mempool.” Use *active voice* and *present tense* (i.e. adds/creates/checks/updates/deletes).
* Describe how and why to use this method.
* Any condition that must be met _before_ calling the method.
* State conditions under which the function will `panic!()` or returns an `Error`
* Brief description of return values.
* Any special behavior that is not obvious
 
### README.md for top-level directories and other major components

Each major component of the system needs to have a `README.md` file. Major components are:
* top-level directories (e.g. `libra/network`, `libra/language`)
* the most important crates in the system (e.g. `vm_runtime`)

This file should contain:
 
 * The *conceptual* *documentation* of the component.
 * A link to the external API documentation for the component.
 * A link to the master license of the project.
 * A link to the master contributing guide for the project. 

A template for readmes:

```
# Component Name

[Summary line: Start with one sentence about this component.]

## Overview

* Describe its purpose and how the code in this directory works.
* Describe the interaction of code in this directory with other components.
* Describe the security model and assumptions about the crates in this directory.  Examples of how to describe the security assumptions will be added in the future.

## Implementation Details

* Describe how the component is modeled. For example, why is the code organized the way it is?
* Other relevant implementation details.

## API Documentation

For the external API of this crate refer to [Link to rustdoc API].

[For a top-level directory, link to the most important APIs within.] 

## Contributing

Refer to the Libra Project contributing guide [LINK].

## License

Refer to the Libra Project License [LINK].
```

A good example of README.md is `libra/network/README.md` that describes the networking crate.

## Code suggestions

Below are suggested best practices for a uniform codebase that will evolve and improve over time.  We will be investigating what can be clippy enforced and will update this section in the future.

### Attributes

Make sure to use the appropriate attributes for handling dead code:

```
// For code that is intended for production usage in the future
#[allow(dead_code)]
// For code that is only intended for testing and 
// has no intended production use
#[cfg(test)]
```

### Avoid Deref polymorphism

Don't abuse the Deref trait to emulate inheritance between structs, and thus reuse methods.  For more information, read [here](https://github.com/rust-unofficial/patterns/blob/master/anti_patterns/deref.md).

### Comments

Prefer `//` and `///` comments rather than block comments `/* ... */` for uniformity and simpler grepping.  

### Cloning

If `x` is reference counted, prefer [`Arc::clone(x)`](https://doc.rust-lang.org/std/sync/struct.Arc.html) rather than `x.clone()`, as it is more explicit that we are cloning `x` in the former case. This avoids confusion about whether we are performing an expensive clone of a `struct`, `enum` or other type or a cheap reference copy.

Also, if you are passing around [`Arc<T>`](https://doc.rust-lang.org/std/sync/struct.Arc.html) types, consider using a newtype wrapper:

```
#[derive(Clone, Debug)]
pub struct Foo(Arc<FooInner>);
```

### Concurrent types

Concurrent types such as [`CHashMap`](https://docs.rs/crate/chashmap), [`AtomicUsize`](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicUsize.html), etc. have an immutable borrow on self i.e. `fn foo_mut(&self,...)` in order to support concurrent access on interior mutating methods. Good practices (such as those in the examples mentioned) avoid exposing synchronization primitives externally (e.g. `Mutex`, `RwLock`) and document the method semantics and invariants clearly.

*When to use channels vs concurrent types?*

Below are high-level suggestions for the distinction based on experience.

* Channels are for ownership transfer, decoupling of types, and coarse-grained messages.  They fit well for transferring ownership of data, distributing units of work, and communicating async results.  Furthermore, they help break circular dependencies (e.g. `struct Foo` contains an `Arc<Bar>` and `struct Bar` contains an `Arc<Foo>` that leads to complex initialization).

* Concurrent types (e.g. such as [`CHashMap`](https://docs.rs/crate/chashmap) or structs that have interior mutability building on [`Mutex`](https://doc.rust-lang.org/std/sync/struct.Mutex.html), [`RwLock`](https://doc.rust-lang.org/std/sync/struct.RwLock.html), etc.) are better suited for caches and states.

### Error handling

Error handling suggestions follow the [Rust book guidance](https://doc.rust-lang.org/book/ch09-00-error-handling.html).  Rust groups errors into two major categories: recoverable and unrecoverable errors, where recoverable errors should be handled with [Result](https://doc.rust-lang.org/std/result/).  For our suggestions on unrecoverable errors, see below:

*Panic*

* `panic!()` - Runtime panic! should only be used when the resulting state cannot be processed going forward.  It should not be used for any recoverable errors.
* `unwrap()` - Unwrap should only be used for mutexes (e.g. `lock().unwrap()`) and test code.  For all other use cases, prefer `expect()`. The only exception is if the error message is custom-generated, in which case use `.unwrap_or_else(|| panic!("error: {}", foo))`
* `expect()` - Expect should be invoked when a system invariant is expected to be preserved.  It is preferred over unwrap() and should have a detailed error message on failure in most cases.
* `assert!()` - This macro is kept in both debug/release and should be used to protect invariants of the system as necessary
* `unreachable!()` - This macro is will panic on code that should not be reached (violating an invariant) and can be used as appropriate.

### Generics

Generics allow dynamic behavior (similar to [`trait`](https://doc.rust-lang.org/book/ch10-02-traits.html) methods) with static dispatch.  Consider that as the number of generic type parameters increase, the difficulty of using the type/method also increases (e.g. what combination of trait bounds is required for this type, duplicate trait bounds on related types, etc.).  In order to avoid this complexity, we generally try to avoid using a large number of generic type parameters.  We have found that converting code with a large number of generic objects to trait objects with dynamic dispatch often simplifies our code.

### Getters/setters

Excluding test code, set field visibility to private as much as possible. Private fields allow constructors to enforce internal invariants. Implement getters for data that consumers may need, but avoid setters unless a mutable state is necessary.

Public fields are most appropriate for [`struct`](https://doc.rust-lang.org/book/ch05-00-structs.html) types in the C spirit: compound, passive data structures without internal invariants.  Naming suggestions follow the guidance [here](https://rust-lang-nursery.github.io/api-guidelines/naming.html#getter-names-follow-rust-convention-c-getter) as shown below.

```
struct Foo {
    size: usize,
    key_to_value: HashMap<u32, u32>
}

impl Foo {
    /// Return a copy when inexpensive
    fn size(&self) -> usize {
        self.size
    }

    /// Borrow for expensive copies
    fn key_to_value(&self) -> &HashMap<u32, u32> {
        &self.key_to_value
    }

    /// Setter follows set_xxx pattern
    fn set_foo(&mut self, size: usize){
        self.size = size;
    }

    /// For a more complex getter, using get_XXX is acceptable 
    /// (similar to HashMap) with well-defined and 
    /// commented semantics
    fn get_value(&self, key: u32) -> Option<&u32> {
        self.key_to_value.get(&key)
    }
}
```

### Logging

We currently use [slog](https://docs.rs/slog/) for logging.

* [error!](https://docs.rs/slog/2.4.1/slog/macro.error.html) - Error-level messages have the highest urgency in [slog](https://docs.rs/slog/).  An unexpected error has occurred (e.g. exceeded the maximum number of retries to complete an RPC or inability to store data to local storage).
* [warn!](https://docs.rs/slog/2.4.1/slog/macro.warn.html) - Warn-level messages help notify admins about automatically handled issues (e.g. retrying a failed network connection or receiving the same message multiple times, etc.).
* [info!](https://docs.rs/slog/2.4.1/slog/macro.info.html) - Info-level messages are well suited for "one-time" events (such as logging state on one-time startup and shutdown) or periodic events that are not frequently occurring - e.g. changing the validator set every day.  
* [debug!](https://docs.rs/slog/2.4.1/slog/macro.debug.html) - Debug-level messages are frequently occurring (i.e. potentially > 1 message per second) and are not typically expected to be enabled in production.
* [trace!](https://docs.rs/slog/2.4.1/slog/macro.trace.html) - Trace-level logging is typically only used for function entry/exit. 

### Testing

*Unit tests*

Ideally, all codes will be unit tested.  Unit test files should exist in the same directory as `mod.rs` and their file names should end in `_test.rs`.  A module to be tested should have the test modules annotated with `#[cfg(test)]`.  For example, if in a crate there is a db module, the expected directory structure is as follows:

```
src/db                        -> directory of db module
src/db/mod.rs                 -> code of db module
src/db/read_test.rs           -> db test 1
src/db/write_test.rs          -> db test 2
src/db/access/mod.rs          -> directory of access submodule
src/db/access/access_test.rs  -> test of access submodule
```

*Property-based tests*

Libra contains [property-based tests](https://blog.jessitron.com/2013/04/25/property-based-testing-what-is-it/) written in Rust using the [`proptest` framework](https://github.com/AltSysrq/proptest). Property-based tests generate random test cases and assert that invariants, also called *properties*, hold about the code under test.

Some examples of properties tested in Libra:

* Every serializer and deserializer pair is tested for correctness with random inputs to the serializer. Any pair of functions that are inverses of each other can be tested this way.
* The results of executing common transactions through the VM are tested using randomly generated scenarios, and a simplified model as an *oracle*.

A tutorial for `proptest` can be found in the [`proptest` book](https://altsysrq.github.io/proptest-book/proptest/getting-started.html).

References:

* [What is Property-Based Testing?](https://hypothesis.works/articles/what-is-property-based-testing/) (includes a comparison with fuzzing)
* [An introduction to property-based testing](https://fsharpforfunandprofit.com/posts/property-based-testing/)
* [Choosing properties for property-based testing](https://fsharpforfunandprofit.com/posts/property-based-testing-2/)

*Fuzzing*

Libra contains harnesses for fuzzing crash-prone code like deserializers, using [`libFuzzer`](https://llvm.org/docs/LibFuzzer.html) through [`cargo fuzz`](https://rust-fuzz.github.io/book/cargo-fuzz.html). For more, see the `testsuite/libra_fuzzer` directory.
