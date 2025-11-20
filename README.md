# Lume Compiler Development Guide

This book is an attempt to explain how the [Lume compiler](https://github.com/lume-lang/lume) works for
new contributors, as well as being a reference for compiler debugging, testing and general compiler documentation.

[The current version can be read here.](https://lume-lang.github.io/compiler-book/)

### Building the book

The book uses [`mdbook`](https://github.com/rust-lang/mdBook), which can be installed using Cargo:
```sh
cargo install mdbook
```

Afterwards, the book can be built by executing the following command in the repository root:
```sh
mdbook build --open
```
