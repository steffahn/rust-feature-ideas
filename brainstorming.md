Quick and unfinished ideas, most of them just me brainstorming :-D
*	Have a trait `DropEarly`
	*	(Ideally / at least conceptually) supertrait of `Copy`,
		ideally an auto trait although that would make it quite the
		breaking change, if it’s not auto but supertrait of `Copy` that would be breaking too.

		(Update: perhaps scrub the relationship to `Copy`, it’s more about Drops).
		
		(Update on the auto trait idea: You could (should) by-default exclude pointers; those would probably be needed
		for something that implements e.g. a scoped lock. Also Editions might somehow avoid breaking change.)

	*	Makes the compiler drop a value of this type not at the end of it’s static scope but as
		early as possible. (Or perhaps just allows dropping at any point from earliest possible
		to end of scope?

	*	Possible problems are: E.g. locks that rely on being held for the scope for logically correct behavior,
		and extra latency introduced into code by clean-up happening before the very end. The second point might be not an actual
		problem since Rust doesn’t seem to care about such latency questions very much anyways. When there’s something like a block
		of code where latency is relevant, one could defer all cleanup possible, maybe only allow functions marked non-blocking, or even
		worst-case constant time or something like that.

	*	Hence some (or most) drop flags would become unnecessary; furthermore some moves can be eliminated,
		(suppose, `String` implements `DropEarly`) for example:
		```rs
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
		```rs
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
		```rs
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
		
		This should also dramatically help with supporting tail call optimization implicitly. (An explicit version is still desirable
		of course, to prevent bugs.) Currently, with a tail call, any local variable not passed to the call that has `Drop` glue
		could not be dropped before that call, resulting in the situation that additional stack space is still needed. With the
		`DropEarly` feature, everything could be dropped unless its lifetime is restricted by some argument passed to the tail call or
		it does not implement `DropEarly`.
		
		The explicit tail-call would have similar constraints, but without the `DropEarly` problem.

	*	The exact (or perhaps earliest possible) point of drop is the end of the smalles possible lifetime of the variable.

	*	There should be `impl`s such as for example:
		```rs
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
	([Playground](https://play.rust-lang.org/?version=nightly&mode=debug&edition=2018&gist=b56bf6de50f4a69d33914d4dbac1af34))

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
		This could make `mut` annotations for functions optional, but generate a warning if missing, to make it non-breaking somehow.
		This sounds like some kind of global type-inference but not a full-blown one, so it might just be thinkable.

	*	Lastly, unsafe code is allowed to break these properties. This may allow properly `const`-typed functions executed at
		compile-type to mess with the compiler - not so sure what to do about it... I don’t _really_ feel okay with UB at
		compile-time... Maybe marking functions as `internally unsafe` and disallowing them, where the standard library
		and some “officially” approved code as well as all the crates that you mark explicitly as trustworthy for during-compilation
		unsafety in your crate/project configuration can then _be_ allowed again. However - using “untrusted” code/libs is something you
		shouldn’t to at all in general, so maybe this is not so smart after all?

	*	A possible extension of the `mut` syntax could include additional 

*	Anonymous `trait`-associated-types by allowing `impl` return values. Allow referencing a functions return type via
	`...::return`, i.e. `path::to::function::<'a,'b,A,B,C>::return` or `function::return`. Consequently allow
	`Trait<method::return = A>`.

*	`sealed trait` for traits that only allow `impls` inside the current crate. Also `sealed impl` for impls with `default fn`s inside
	where you don’t want any non-crate-local specializations. Finally `sealed partial impl` might behave similarly; in this case
	`sealed trait` is just syntax for an empty `sealed partial impl`. This also allows for negative implementations
	like for example `sealed partial impl<'a> DerefMut for &'a {}` prevents anyone from implementing `DerefMut` on immutable
	references (see: problems with unsoundness of `Pin`).

*	Allow update syntax for non-`Drop` implementing structs even if some fields are private. _(TODO: how could this **actually** be
	desugared?)_

*	Somehow provide default that partially initializes a struct and calls for specifying the rest manually, furthermore
	allows overwriting already-initialized public fields and, as a bonus, would even allow, somehow, and maybe not
	for all fields, that a provided-and-defaulted public field skips the creation of the default value alltogether.
	
	On the last point, maybe somehow flag types that can be constructed and destructed again without _actual_ side
	effect except for, perhaps, some allocations (which is _the point_ of the whole idea) that can be skipped.

*	Have types that disallow default/implicit drop, as perhaps an extension to the current `must_use` warning. Of course `leak`
	would still work, but it’s a useful strong lint to thinks that you can only “properly” get rid of in a more complex manner,
	especially if that manner needs some extra arguments.

*	Borrowing parts of something: The basic problem is that you cannot model fields with methods currently. This also
	means that proper “properties” syntax is not really possible in a way where they behave the same as fields do. One problem is
	that fields can borrow only that part of the struct whereas methods called on the struct must keep the whole thing occupied.
	There could be syntax for a reference to the whole struct but with access to certain fields only. Furthermore there is fan-out:
	Certain parts could be borrowed for longer than others, go into different result types, or only be used inside the method itself.
	
	*	Would need to be able to borrow fields. Would be nice to have: multiple fields, named (abstract) groups of fields, support
		array indices and ranges if indices (no need for split_mut). For this, some kind of `const` functionality for overlap-tests is
		needed. For instance an associated (usually enum) type allowing indices, sets and ranged, with implementations for
		overlap-checks could do the trick.
	
	*	Another thing that fields do is destructuring. This seems more complicated to abstract upon. You would perhaps need a way to
		disjointly union areas, and get the complement, and finally pass the result as a const parameter to the destructor that also
		only gets a reference on that area. And this is only for the static stuff... IIRC there are probably drop-check flags on
		individual fields, too, and this kind of compatibility shall not be lost either. Finally, with indexing, indices are often
		only available at run-time. (Damn it, `split_mut` probably stays...) It could be possible to do some overlap-ckecks at runtime,
		however what to do if these fail? Panics are somewhat surprising; so possibly something with `Option`...

*	Go towards linear-ish types with something that is not like `must_use` bound to functions but instead to types in a sense that they
	must be either destructured or passed to `drop` in a place where all the fields are visible and not `must_use`. Perhaps even allow
	implicit drops, and drops inside generics somehow if that’s possible. A light version might also allow as an opt-in for a type to
	allow explicit `drop` in general.

*	Improve usage of `Sized`. In particular in view of the possibility of future similar types that are bound by default for backwards
	compatibility. The ideas for improvements include:
	*	A warning if a bound `<T>` could be generalized to `<T: ?Sized>` but isn’t. (The warning would require either an explicit
		`<T: Sized>` or committing to the generealization `<T: ?Sized>` to go away.
	*	Regarding this, it is not quite clear how this is supposed to work around traits, since the generalization of an `impl`
		technically adds new `impl`s in a potentially breaking way. Thinking about it, the _real_ breakage only occurs if a conflicting
		`impl` already exists for some `!Sized` type.
	*	Perhaps automatically infer `Sized` constraints on non-public things. The main reason against inference of such stuff is
		two-fold: Explicit signatures may make development easier since less stuff is implicitly assumed. In the particular case of
		`Sized` there are some times where the need for a `Sized` bound is already clear from the rest of the signature, while in other
		cases an explicit `Sized` constraint would actually be the most readable thing. The second reason however is to avoid breaking
		changes in interface. If type information is implicit, then types can change without the signature changing. In particular,
		breaking changes from removal of capabilities that were never meant to exist are problematic. The problem of breakage however
		really only occurs if the provider and the user of an API are different people -- and in particular if they don’t even know
		each other -- non-public things will never be part of such an API, so inference might be okay there. On a side-note, Rust
		actually (unfortunately) already has, in some cases, type information that is not explicit in a signuature. This is the case
		for functions returning `impl Future` or `impl Fn…` with respect to some `auto trait`s like `Send`. A generic function
		`fn f<T>(x: T) -> impl FnOnce()` may put the `T` inside the closure which is thus only `Send`, `Sync`, etc, if `T` is.

*	Fix contraints in type definitions. Remove the same quirk that
	[Haskell used to have](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/glasgow_exts.html#data-type-contexts).

*	Type based macros
	*	A collection of traits `Macro`, `MacroMut`, `MacroOnce`, and `PrimitiveMacroOnce`, where the hierarchy is:
		Every `PrimitiveMacroOnce` is a `MacroOnce`, every `Macro` is a `MacroMut`, every `MacroMut` is a `MacroOnce`.
	*	every _ordinary_ macro is a zero_sized type that implemets `Macro` and `PrimitiveMacroOnce` and `Copy`.
	*	`Macro` has an associated type `MacroType: PrimitiveMacro`, and we have `trait PrimitiveMacroOnce: MacroOnce<MacroType = Self>`.
	*	There is a way to create _macro closures_.
		```rs
		// TODO: make this fit with the changed PrimitiveMacroOnce
		fn foo() -> impl PrimitiveMacro {
			let x = 1;
			let y = 2;
			let my_macro = macro_closure!(move [ref x, ref y], {
				(foobar +-+ $e:expr) => {println!("{}", x + y + $e};
			});
			my_macro
		}
		fn bar() {
			let my_macro = foo();
			my_macro!(foobar +-+ 3*3);
		}
		```
		which could, roughly, be thought of like
		```rs
		macro_rules! foo_macro_closure_001 {
			($x:ident $y:ident foobar +-+ $e:expr) => {println!("{}", (*$x) + (*$y) + $e};
		}
		fn foo() -> impl PrimitiveMacro {
			let x = 1;
			let y = 2;
			struct foo_macro_closure_001_t {
				x: i32,
				y: i32,
			}
			#[implemented_by(foo_macro_closure_001)]
			impl PrimitiveMacro for foo_macro_closure_001_t {}
			let my_macro = foo_macro_closure_001_t { x, y };
			my_macro
		}
		fn bar() {
			let my_macro = foo();
			let foo_macro_closure_001_t { ref __x, ref __y } = my_macro;
			foo_macro_closure_001!(__x __y foobar +-+ 3*3);
		}
		```
	*	Notably, we need `move` like for closures. We need to explicitly specify which variables are captured and whether directly, by `ref`, or be `ref mut`.
	*	In expression position the `let ..._clocure_..._t { ... } = ...` part is in a new block.
	*	We might _actually_ want those `__x` and `__y` to be more like temporaries and live until the end of the expression.
	*	Macro closures either implement `PrimitiveMacroOnce` and `MacroOnce` directly, or (when they are callable multiple times)
		they implement `Macro` or `MacroMut` and have a (hidden) extra newtype around a
		a reference or mut ref to themselves implement `PrimitiveMacroOnce`.
		TODO: Think about optimizations around small types (smaller than references) that are `Copy` in this context, too.
	*	This feature gives us:
		*	Better import/export of macros, referencing of other macros, even private ones (just include them into the closure, it’s still going to stay
			zero-sized when it has zero-sized fields... well, at least if we’re smart about including zero-sized `Copy` types (or in general
			smaller-than-or-equal-to-pointer-sized	`Copy` types by value instead of by shared static reference.. actually, also every compile-time-known static reference
			could be replaced by a zero-sized type anyways))
		*	Macros in method position: just do things like `object.macro_method()!(special ## syntax +-+ whatever)`
		*	We might want to think about a way to write this like `object.macro_method!(special ## syntax +-+ whatever)`
		*	Features like named arguments as macros.

*	Here’s an easy one: Make `std::mem::needs_drop::<T>()` return `false` even if `T` implements `Drop` as long as the `drop` implementation does not do anything.
	In particular, this should be made to work if the code in `fn drop` impl is wrapped in an `if` with a `const` evaluable condition.
