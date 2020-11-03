---
title: "Exploring Rust - Week 1"
date: 2020-10-24T16:20:13-04:00
series: ["Exploring Languages", "Exploring Rust"]
---
I've read through Chapters 1 - 3 of ["The Book"](https://doc.rust-lang.org/book/).

Topics covered:
- Installation with `rustup` and using `cargo`
- Types, control flow, mutability, functions


Some things I like off-the-bat:

- **Variables are immutable by default**

  If you want to mutate a variable, you must use the `mut` keyword. I can see this cutting down on accidental mutation and logic errors.

- **`match` statements/destructuring**

  I've been a fan of pattern matching since I was first exposed to it with OCaml in undergrad. Every time I use `switch` in JavaScript, I think of what could've been 😩.
