Quick and unfinished ideas, most of them just me brainstorming :-D
*	Have a trait `DropEarly`
	*	(Ideally / at least conceptually) supertrait of `Copy`,
		ideally an auto trait although that would make it quite the
		breaking change, if it’s not auto but supertrait of `Copy` that would be breaking too.
		(Update: perhaps scrub the relationship to `Copy`, it’s more about Drops).
	*	Makes the compiler drop a value of this type not at the end of it’s static scope but as
		early as possible. (Or perhaps just allows dropping at any point from earliest possible
		to end of scope?)
	*	Hence some (or most) drop flags would become unnecessary; furthermore some moves can be eliminated,
		(suppose, `String` implements `DropEarly`) for example:
		```rust
		fn main() {
			let a = String::from("foo");
			if EXPR {
				// do something that moves a like
				drop(a)
				// do something else
			} else {
				// don’t use a
			}
			// some other code
		}
		```
		currently becomes something like
		```rust
		fn main() {
			let a = String::from("foo");
			let a_dropped = false;
			if EXPR {
				// do something that moves a like
				drop(a)
				a_dropped = true;
				// do something else
			} else {
				// don’t use a
			}
			// some other code
			if !a_dropped { drop(a) } // of course not legal rust, since a isn’t usable here 
		}
		```
		But it could instead become:
		```rust
		fn main() {
			let a = String::from("foo");
			if EXPR {
				// do something that moves a like
				drop(a)
				// do something else
			} else {
				drop(a);
				// don’t use a
			}
		}
		```

		About avoiding moves:
		TODO: find that example where `Drop` enforces a copy in one of the books again.

		Definitely helps at least save some stack space in some cases. (Should be obvious.)

		Solves problems with memory leaks in a naive REPL (i.e. REPL interface with the approach:
		what you write constitures the body of a very long `main()` function. There rerunning commands
		shadowing earlier bindings will allow drops of `DropEarly` values (if no reference, etc. is retained either).
	*	The exact (or perhaps earliest possible) point of drop is the end of the smalles possible lifetime of the variable.
	*	There should be `impl`s such as for example:
		```rust
		impl<T: ?Sized + DropEarly> DropEarly for Box<T> {}
		impl<T: ?Sized + DropEarly> DropEarly for Vec<T> {}
		```
		Note, that this already has effect on types like `Vec<u8>`, etc

		And at least every type that doesn’t (directly or inderectly) need any custom drop should automatically
		implement `DropEarly`. This might introduce a breaking change in library updates if they change their type.
		This is bad if we don’t allow disabling the feature for such types.
		Maybe a good approach to prevent something like that is something like only allow use of `DropEarly`
		constraints in `DropEarly` impls.

		As stated in the beginning, implementation as an auto trait can break current stuff. However, maybe structs that
		contain only `DropEarly` types and that themselves don’t implement `Drop` (so they don’t do anything on drop that’s
		not not already explicitly marked _okay for executing earlier than lexical end of scope_ by their component
		anyways) can get auto `impl`’d.

*	Some type system feature that allows back-wards reasoning to refine a type with certain defaults into slower but more
	capable variants. The example in mind would be a data structure using `Rc`, but that upgrades to `Arc` to become `Sync`
	(or maybe also to become `Send`). Questions arise on how far this decision gets fixed, i.e. in an Application where some
	variables (in some places) need `Sync` and others don’t, on what granularity will types be fixed to the `Arc` version around those
	variables?

	Reasoning: It’s inconvenient to implement and even more inconvenient to use rc-based data

	Problems: Lots. For example even: Concrete `Rc` and `Ac` has special properties like method dispatching - how does that generalize?
	Also, while we’re at it, we should provide ways to unify `&` and `&mut`, and perhaps there, too, allow chains of type-inference to
	pick the right one.

*	Shouldn't references to zero-sized types be zero-sized and references to empty types be empty?

	Shouldn't immutable references to (non-UnsafeCell-containing) small (i.e. sub pointer-size) types be actually passed
	by value? Although this is dangerous if it can kill optimizations such as `Option<&usize>` being small enough. But
	at least for types strictly smaller than `usize`, we can pass value instead, saving dereferencing operations.

	Although... this actually introduces dereferencing to `&mut` to `&` conversions.... aaaand makes `&` to `*const`
	conversions kind-of _wrong_
*	This type:
	([Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=91f673f3ced8398d01f44372d56c41cf))
*	Mutability generics, and allowing pure functions (by improving `const`).
	*	_[TODO: did I already mention sth. like this above?]_ Marking non-`const` functions `mut` instead. I.e. `mut fn f(a: A) -> B`.

	*	Treat `mut` in references like lifetimes in some regards: Allow generic functions over those, as in some syntax like
		for example: `fn f<'M>(some_struct: &mut?'M Struct) -> &mut?'M Field`. We’d have reserved mutability names `'Mut` and `'Ref`
		and the explicit `&'a mut` would mean `&'a mut?'Mut`, and `&'a` would mean `&'a mut?'Ref`. There would be the option to have
		implicit mutability constraints just like for lifetimes, syntax being for example
		`fn f(some_struct: &mut? Struct) -> &mut? Field` (meaning the same as previous explicit example), with similar
		or identical rules as for lifetimes. There would be a subtyping	relation `'Ref : 'Mut`
		_[TODO: is currently `&` vs `&mut` subtyping or is it implicit conversion? How bad might a change be?]_	and covariance of
		refs in their mutability. Accordingly, `'M + 'N` is `'Mut` if and only if `'M` and `'N` are _both_ `'Mut`. Also
		`for<...>` notation should be included.

	*	You can then remove lots of duplication. For example `FnMut` vs `Fn`. `BorrowMut` vs. `Borrow`. (Note, for unsafe code, for
		example in `RefCell` it should be useful to get the value of a mutability parameter at run-time.)

	*	Back to the first point: We can then have _[TODO: how does actually this compare to current `const` RFCs?]_ a function like
		`fn mutate_field(&mut struct: Struct)` and another function `mut fn mutate_global_state()`. I think this compares very well.
		A problem is something like `RefCell`. It might, at a first approximation, mutate global state to write into the inner
		mutability.	But it also has read-only access start rely on global state, i.e. both `borrow` and `borrow_mut` being `mut fn`s.

	*	I'd like to introduce a distiction between references with interior mutability and ones without. Maybe `&const` for
		pure references. So have `'Const : 'Ref` and `'Const : 'Mut`. The idea being, that `RefCell`’s `borrow` can still
		be `fn borrow(&self) -> Ref<T>`. But for example normal field access, like `fn f(some_struct: &mut? Struct) -> &mut? Field`,
		would allow for usage via `(some_struct: &const Struct) -> &const Field`.

	*	Perhaps, if I can think of a good way, there might be a possibility to infer all the `mut` vs. non-`mut fn`’s information
		as well as `'Ref` vs `'Const` automatically. (I.e. make `&` mean both `&const` and “`&ref`” at the depending on inference.)
		This could make `mut` annotations for functions optional, but generate a warning if missing to make it non-breaking somehow.
		This sounds like some kind of global type-inference but not a full-blown one, so it might just be thinkable.

	*	Lastly, unsafe code is allowed to break these properties. This may allow properly `const`-typed functions executed at
		compile-type to mess with the compiler - not so sure what to do about it... I don’t _really_ feel okay with UB at
		compile-time... Maybe marking functions as `internally unsafe` and disallowing them, where the standard library
		and some “officially” approved code as well as all the crates that you mark explicitly as trustworthy for during-compilation
		unsafety in your crate/project configuration can then _be_ allowed again. However - using “untrusted” is something you
		shouldn’t to at all in general, so maybe this is not so smart after all?

	*	A possible extension of the `mut` syntax could include additional 

*	Anonymous `trait`-associated-types by allowing `impl` return values. Allow referencing a functions return type via
	`...::return`, i.e. `path::to::function::<'a,'b,A,B,C>::return` or `function::return`. Consequently allow
	`Trait<method::return = A>`.
