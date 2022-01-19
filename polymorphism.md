*	General idea: Compile-time-only type arguments that are erased / not monomorphized, just like lifetimes.
*	Syntax: `fn foo<poly T>(…)`. Function is compiled down to a single non-generic ABI / symbol / function.
*	Many applications:
	*	Allows a limited form of generics in dynamic linking.
	*	Can be used in methods of object-safe traits.
	*	Saves monomorphization overhead when it’s not needed.
	*	Allows polymorphic recursion.
	*	Can be used in `Foo: for<poly T> TraitBound<Bar<T>>` bounds.
	*	Polymorphic arguments are allowed to be left under-specified.
*	Many future possibilities:
	*	_TODO…_

*	In a `fn foo<poly T>(…) -> … { … }` function, `T` can only be used in limited ways!
	*	as parameter for other functions with `poly T` type argument
	*	lifetime bounds `T: 'a` are okay, most trait bounds are not allowed (more details below)
	*	as parameters for _types_ with a `poly T` type argument
		*	standard library types with `poly T` type argument:
			*	`&'a T`
			*	`&'a mut T`
			*	`*const T`
			*	`*mut T`
			*	`cell::Ref<'a, poly T>`
			*	`cell::RefMut<'a, poly T>`
			*	`PhantomData<poly T>`
		*	some standard library functions/methods with `T: ?Concrete` type argument:
			*	_TODO…_
*	Structs can have `poly T` type arguments
	*	all usages in fields must restricted like for the usages mentioned above
	*	the `Drop` implementation must have `poly T` arguments, and the restrictions for functions
		apply to the implementation of `fn drop(&mut self)`

*	If a `struct Foo<poly T>` is considered, then `Foo<Bar>` and `Foo<Baz>` have the same
	*	layout
	*	drop implementation

	yet still, ordinary generic functions `fn baz<T>` can treat them differently, hence need to be monomorphized for `baz::<Foo<Bar>>`
	and `baz::<Foo<Baz>>` separately; also you cannot call an ordinary generic function `baz::<Foo<T>>` using the generic polymorphic 
	parameter `T` from inside the definition of a function `fn foo<poly T>`. This means there needs to be a way for a generic function
	to _opt in_ guaranteeing to treat types like `Foo<Bar>` or `Foo<Baz>` the same at run-time. These functions support more than just
	“concrete” types, hence they get a new opt-out trait bound:
*	Syntax: `fn foo<T: ?Concrete>(…)`. Function does not differentiate between different `T`s that only differ in `poly` arguments.

*	With a struct `Foo<poly T>`, in the body of a function `foo<poly T>() { … }`, you can pass `T` to `Foo`, building the type `Foo<T>`;
	but then that type will not be considered to implement `Concrete`, so you can only use it as an argument for functions or types that accept
	a `<S: ?Concrete>` type argument.

*	`?Concrete` types have a known layout, and can be dropped. But not much more beyond that.
*	Functions and structs can, of course, have `T: ?Concrete` type arguments, usual trait resulution rules apply to determine what you can use the parameter for.
	* For structs, their drop implementation – if any is present – must naturally have a `T: ?Concrete` bound, too.
	*	Standard library types with `T: ?Concrete` type argument:
		*	`Box<T: ?Concrete>`
		*	_TODO…_
	*	Some standard library functions/methods with `T: ?Concrete` type argument:
		*	_TODO…_

* Each of `poly T` and `T: ?Concrete` also supports `poly T: ?Sized` and `T: ?Concrete + ?Sized`

<hr>

Overview of how fine-grained monomorphization is for each case

_TODO: THERE’LL BE A TABLE_
