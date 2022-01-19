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

<hr>

```rs
use std::{
    alloc::{alloc, dealloc, Layout},
    ptr::{self, NonNull},
};

struct PolyBox<poly T> {
    ptr: NonNull<T>,
    metadata: NonNull<Metadata<T>>,
}

impl<T: ?Concrete> PolyBox<T> {
    fn new(x: T) -> Self {
        let meta: &Metadata<T> = StaticMetadata::META;
        let ptr = unsafe { NonNull::new_unchecked(alloc(meta.layout)) };
        Self {
            ptr: ptr.cast(),
            metadata: NonNull::from(meta),
        }
    }
}

impl<poly T> Drop for PolyBox<T> {
    fn drop(&mut self) {
        let meta: &Metadata<T> = unsafe { self.metadata.as_ref() };
        struct DeallocOnDrop(*mut u8, Layout);
        let _guard = DeallocOnDrop(self.ptr.as_ptr().cast(), meta.layout);
        unsafe { (meta.drop_in_place)(self.ptr.as_mut()) };
        impl Drop for DeallocOnDrop {
            fn drop(&mut self) {
                unsafe { dealloc(self.0, self.1) };
            }
        }
    }
}

struct Metadata<poly T> {
    layout: Layout,
    drop_in_place: unsafe fn(*mut T),
}
impl<T: ?Concrete> Metadata<T> {
    const fn new() -> Self {
        Self {
            layout: Layout::new::<T>(),
            drop_in_place: ptr::drop_in_place::<T>,
        }
    }
}

trait StaticMetadata<'a>: ?Concrete + Sized + 'a {
    const META: &'a Metadata<Self>;
}

impl<'a, T: ?Concrete + 'a> StaticMetadata<'a> for T {
    const META: &'a Metadata<Self> = &Metadata::new();
}

struct Tree<poly T>(Count<T>);

enum Count<poly T> {
    Once(PolyBox<T>),
    Twice(Box<Count<[PolyBox<T>; 2]>>),
}
use Count::*;

impl<poly T> Tree<T> {
    // link 2 trees of equal size
    fn link(self, other: Tree<T>) -> Option<Tree<T>> {
        fn recurse<poly T>(left: Count<T>, right: Count<T>) -> Option<Count<T>> {
            Some(match (left, right) {
                (Once(l), Once(r)) => Twice(Box::new(Once(PolyBox::new([l, r])))),
                (Twice(l), Twice(r)) => Twice(Box::new(recurse(*l, *r)?)),
                _ => None?,
            })
        }
        Some(Tree(recurse(self.0, other.0)?))
    }
}
```
