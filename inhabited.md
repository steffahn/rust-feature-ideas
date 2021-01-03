Addressing this point from [brainstorming.md](brainstorming.md)
>	*	Fix contraints in type definitions. Remove the same quirk that
>		[Haskell used to have](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/glasgow_exts.html#data-type-contexts).

*	Adding special marker traits `Inhabited` and `Contains<T>`. `Inhabited` allows for some forms of negative reasoning.
*	If `S: Contains<T>` then `S: Inhabited` entails `T: Inhabited` and `T: !Inhabited` entails `S: !Inhabited`.
*	We have e.g. `&'a T: Contains<T>`, `Contains` hat to be implemented manually but implementations are checked by the compiler for correctness.
*	Also `!: !Inhabited` and `std::convert::Infallible: !Inhabited`.
*	`Inhabited` can’t be implemented manually for a type. Enums without variants are globally known to be `!Inhabited`. Also `Inhabited`-implications
	analog to how `Contains` works are considered between structs and public fields, or single-variant enums and their fields. This might include a
	rule such that all struct field being public and all fields being `Inhabited` can make the struct known to be `Inhabited`.	Also in general, if
	an (exhaustive) enum has an `!Inhabited` field in every variant, it’s `!Inhabited`; types that are `Contains`-contained in some field of every
	variant are `Inhabited` if the enum is `Inhabited`; and if all fields of some variant are `Inhabited`, the enum is `Inhabited`.
*	Whenever a function has a parameter of some type _`TypeExpr`_, inside of this function, <code>_TypeExpr_: Inhabited</code> is known.
	The information that `T: Inhabited` holds can come up on a more fine-grained grained way than on the level of full function bodies. In the control flow,
	at any point where there is a value of type _`TypeExpr`_, <code>_TypeExpr_: Inhabited</code> hold true. Furthermore, if any control flow that leads
	to the current point is known to have come from a point where a value of type _`TypeExpr`_ _was_ in scope, then still <code>_TypeExpr_: Inhabited</code>
	at that later point. Similarly if the value is not in scope has already been bound to a temporary in the current or previous statements, or if it _could_ have
	been bound but was discarded, for example by a `_` in a pattern.
	*	This last point aims to allow code like
		```rust
		let x: Result<i32, !> = Ok(42);
		match x {
			Ok(y) => y,
			Err(_) => {}, // block returns `!`, because `!: !Inhabited`
			              // and the `!` could have been bound by the `_`
		```
		The rule could be extended so that `_ => {}` works, too, and ultimately such that the whole branch is allowed to be omitted.
*	For a data type with constraints, like e.g.
	```rust
	struct HasEqConstraint<T: Eq>(PhantomData<*const T>);
	```
	using the constructor `HasEqConstraint` requires `T: Eq` to be known. The same applies to unsized coercions, e.g. from `Box<HasEqConstraint<Struct>>`
	to `Box<HasEqConstraint<dyn Trait>>`, assuming a version of `HasEqConstraint` with a `?Sized` parameter in the first place. How unsized coercions interact with
	this in detail remains to be worked out fully.
	
	Then, `HasEqConstraint`’s constraint makes the compiler aware that `HasEqConstraint<T>: Inhabited` entails `T: Eq`.
*	Whenever at some point both <code>_TypeExpr_: Inhabited</code> and <code>_TypeExpr_: !Inhabited</code> holds, it is known that that point is unreachable.
	In particular, in unreachable code an end of a block has an implicit return value of `!` instead of `()`.
*	The compiler has an internal, more precise notion of inhabited vs uninhabited types. Basically, this notion operates as if every legal `Contains`
	implementation would actually exist and all fields are public. This internal reasoning can be used to remove uninhabited variants from enums.
