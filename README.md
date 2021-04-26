[<img alt="github" src="https://img.shields.io/badge/github-self__cell-8da0cb?style=for-the-badge&logo=github" height="20">](https://github.com/Voultapher/self_cell)
[<img alt="crates.io" src="https://img.shields.io/badge/dynamic/json?color=fc8d62&label=crates.io&query=%24.crate.max_version&url=https%3A%2F%2Fcrates.io%2Fapi%2Fv1%2Fcrates%2Fself_cell&style=for-the-badge&logo=rust" height="20">](https://crates.io/crates/self_cell)
[<img alt="docs.rs" src="https://img.shields.io/badge/docs.rs-self__cell-66c2a5?style=for-the-badge&logoColor=white&logo=data:image/svg+xml;base64,PHN2ZyByb2xlPSJpbWciIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyIgdmlld0JveD0iMCAwIDUxMiA1MTIiPjxwYXRoIGZpbGw9IiNmNWY1ZjUiIGQ9Ik00ODguNiAyNTAuMkwzOTIgMjE0VjEwNS41YzAtMTUtOS4zLTI4LjQtMjMuNC0zMy43bC0xMDAtMzcuNWMtOC4xLTMuMS0xNy4xLTMuMS0yNS4zIDBsLTEwMCAzNy41Yy0xNC4xIDUuMy0yMy40IDE4LjctMjMuNCAzMy43VjIxNGwtOTYuNiAzNi4yQzkuMyAyNTUuNSAwIDI2OC45IDAgMjgzLjlWMzk0YzAgMTMuNiA3LjcgMjYuMSAxOS45IDMyLjJsMTAwIDUwYzEwLjEgNS4xIDIyLjEgNS4xIDMyLjIgMGwxMDMuOS01MiAxMDMuOSA1MmMxMC4xIDUuMSAyMi4xIDUuMSAzMi4yIDBsMTAwLTUwYzEyLjItNi4xIDE5LjktMTguNiAxOS45LTMyLjJWMjgzLjljMC0xNS05LjMtMjguNC0yMy40LTMzLjd6TTM1OCAyMTQuOGwtODUgMzEuOXYtNjguMmw4NS0zN3Y3My4zek0xNTQgMTA0LjFsMTAyLTM4LjIgMTAyIDM4LjJ2LjZsLTEwMiA0MS40LTEwMi00MS40di0uNnptODQgMjkxLjFsLTg1IDQyLjV2LTc5LjFsODUtMzguOHY3NS40em0wLTExMmwtMTAyIDQxLjQtMTAyLTQxLjR2LS42bDEwMi0zOC4yIDEwMiAzOC4ydi42em0yNDAgMTEybC04NSA0Mi41di03OS4xbDg1LTM4Ljh2NzUuNHptMC0xMTJsLTEwMiA0MS40LTEwMi00MS40di0uNmwxMDItMzguMiAxMDIgMzguMnYuNnoiPjwvcGF0aD48L3N2Zz4K" height="20">](https://docs.rs/self_cell)

# `self_cell`

`self_cell` provides a macro-rules macro: `self_cell`. With this macro you can
create self-referential structs that are safe-to-use in stable Rust, without
leaking the struct internal lifetime.

In a nutshell, the API looks *roughly* like this:

```rust
// User code:

self_cell!(
    struct NewStructName {
        #[from]
        owner: Owner,

        #[covariant]
        dependent: Dependent,
    }
    
    impl {Debug}
);

// Generated by macro:

struct NewStructName(...);

impl NewStructName {
    fn new(owner: Owner) -> NewStructName { ... }
    fn borrow_owner<'a>(&'a self) -> &'a Owner { ... }
    fn borrow_dependent<'a>(&'a self) -> &'a Dependent<'a> { ... }
}

impl Debug for NewStructName { ... }
```

Self-referential structs are currently not supported with safe vanilla Rust. The
only reasonable safe alternative is to expect the user to juggle 2 separate data
structures which is a mess. The library solution ouroboros is really expensive
to compile due to its use of procedural macros.

This alternative is `no_std`, uses no proc-macros, some self contained unsafe
and works on stable Rust, and is miri tested. With a total of less than 300
lines of implementation code, which consists mostly of type and trait
implementations, this crate aims to be a good minimal solution to the problem of
self-referential structs.

It has undergone [community code review](https://users.rust-lang.org/t/experimental-safe-to-use-proc-macro-free-self-referential-structs-in-stable-rust/52775)
from experienced Rust users.

### Fast compile times

```
$ rm -rf target && cargo +nightly build -Z timings

Compiling self_cell v0.7.0
Completed self_cell v0.7.0 in 0.2s
```

Because it does **not** use proc-macros, and has 0 dependencies compile-times
are fast.

Measurements done on a slow laptop.

### A motivating use case

```rust
use self_cell::self_cell;

#[derive(Debug, Eq, PartialEq)]
struct Ast<'a>(pub Vec<&'a str>);

impl<'a> From<&'a String> for Ast<'a> {
    fn from(code: &'a String) -> Self {
        // Placeholder for expensive parsing.
        Ast(code.split(' ').filter(|word| word.len() > 1).collect())
    }
}

self_cell!(
    struct AstCell {
        #[from]
        owner: String,

        #[covariant]
        dependent: Ast,
    }

    impl {Clone, Debug, Eq, PartialEq}
);

fn build_ast_cell(code: &str) -> AstCell {
    // Create owning String on stack.
    let pre_processed_code = code.trim().to_string();

    // Move String into AstCell, build Ast by calling pre_processed_code.into()
    // and then return the AstCell.
    AstCell::new(pre_processed_code)
}

fn main() {
    let ast_cell = build_ast_cell("fox = cat + dog");
    dbg!(&ast_cell);
    dbg!(ast_cell.borrow_owner());
    dbg!(ast_cell.borrow_dependent().0[1]);
}
```

```
$ cargo run

[src/main.rs:36] &ast_cell = AstCell { owner: "fox = cat + dog", dependent: Ast(["fox", "cat", "dog"]) }
[src/main.rs:37] ast_cell.borrow_owner() = "fox = cat + dog"
[src/main.rs:38] ast_cell.borrow_dependent().0[1] = "cat"
```

There is no way in safe Rust to have an API like `build_ast_cell`, as soon as
`Ast` depends on stack variables like `pre_processed_code` you can't return the
value out of the function anymore. You could move the pre-processing into the
caller but that gets ugly quickly because you can't encapsulate things anymore.
Note this is a somewhat niche use case, self-referential structs should only be
used when there is no good alternative.

Under the hood, it heap allocates a struct which it initializes first by moving
the owner value to it and then using the reference to this now Pin/Immovable
owner to construct the dependent inplace next to it. This makes it safe to move
the generated SelfCell but you have to pay for the heap allocation.

See the documentation for a more in-depth API overview and advanced examples:
https://docs.rs/self_cell

### Installing

[See cargo docs](https://doc.rust-lang.org/cargo/guide/).

## Running the tests

```
cargo test

cargo miri test
```

### Related projects

[rental](https://github.com/jpernst/rental)

[Schroedinger](https://github.com/dureuill/sc)

[owning_ref](https://github.com/Kimundi/owning-ref-rs)

[ouroboros](https://github.com/joshua-maros/ouroboros)

[ghost-cell](https://github.com/matthieu-m/ghost-cell)

## Contributing

Please respect the [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) when contributing.

## Versioning

We use [SemVer](http://semver.org/) for versioning. For the versions available,
see the [tags on this repository](https://github.com/Voultapher/self_cell/tags).

## Authors

* **Lukas Bergdoll** - *Initial work* - [Voultapher](https://github.com/Voultapher)

See also the list of [contributors](https://github.com/Voultapher/self_cell/contributors)
who participated in this project.

## License

This project is licensed under the Apache License, Version 2.0 -
see the [LICENSE.md](LICENSE.md) file for details.
