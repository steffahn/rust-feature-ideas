Explicit syntax for declaring whether or not a trait is object safe.

# Motivation

The main issue with the current system is that it’s easy to miss changes of object safety to a trait. This has even happened to `std` once.
This basic idea is also proposed in [an open RFC pull request here](https://github.com/rust-lang/rfcs/pull/3022).

The forms this can take are various. On one hand, there’s the question on syntax. Syntax can be attribute-based or using custom syntax.
E.g. to mark a trait as object safe, one could imagine
*	```rs
	#[object_safe]
	trait Foo {
	    fn foo(&self);
	}
	```
*	```rs
	dyn trait Foo {
	    fn foo(&self);
	}
	```

[INCOMPLETE]
