## On the problem of auto traits and `impl Trait`

The problem is that functions with `-> impl Trait` return types leak auto trait bounds of the real type instantiating the opaque type.
For ordinary functions, this is “merely” a semver footgun. For *`impl Trait` in `trait`* however, this is a real expressivity issue.

Listing auto traits explicitly like
```rs
trait Foo {
    fn bar<T>(arg: T) -> impl Baz + Send;
}
```
has the issue of being too simplistic. Many use cases would probably prefer being able to declare
that the return type of `bar`, `type Ty<S, T> = <S as Foo>::bar::<T>::return;` only fulfills `Send` conditionally,
with an implementation of the form
```rs
impl<S, T> Send for Ty<S, T>
where
    S: Send,
    T: Send,
{}
```
though the precise form that’s desired may vary. For instance, a trait might envision an intended use case that only needs
with an implementation of the form
```rs
impl<S, T> Send for Ty<S, T>
where
    T: Send,
{}
```
or perhaps even a use-case (imagine if a `&self` argument was added and supposed to be allowed to be “captured”)
```rs
impl<S, T> Send for Ty<S, T>
where
    S: Sync, // !!
    T: Send,
{}
```

A “natural” conclusion would be the desire for some sort of quantified trait bounds. E.g. if the thing has a name as a TAIT, this could be a custom attribute like
```rs
#[auto_traits(
    for<S: Sync, T: Send> Ty<S, T>: Send,
    for<S: Sync, T: Sync> Ty<S, T>: Sync,
)]
type Ty<S, T> = impl Baz;
```
or so. Alternatively, one could consider a sort of conditional trait bounds syntax. E.g.
```rs
trait Foo {
    fn bar<T>(arg: T) -> impl Baz + if<Self: Sync, T: Send> Send + if<Self: Sync, T: Sync> Sync;
}
```

All of this is very verbose however.

### A leaner alternative, inspired by `PhantomData`

Auto traits are notable for that we usually do *not* write their implementations manually.
Another personal idea&opinion of mine is that this might even be a good idea to consider in cases where currently they *are* explicit.

E.g. if I were to implement my own personal … let’s see … iterator over `&mut [T]`, then the type contains raw pointers
```rs
struct IterMut<'a, T> {
    current: *mut T,
    end: *mut T,
    marker: PhantomData<&'a mut T>, // marker mostly for using the lifetime
}
```
and then, on top of that, I need to write `Send` and `Sync` implementations
```rs
unsafe impl<T: Sync> Sync for IterMut<'_, T> {}
unsafe impl<T: Send> Send for IterMut<'_, T> {}
```
making sure to mirror the right implementation pattern from `&mut T` (or equivalently `&mut [T]`).

But why the effort? I could just use `PhantomData` to make this “follow the impls of `&mut T`” idea explicit!
Well, almost; `*mut T: !Send + !Sync` kills this idea… [if only there was a general opt-out for those conservative implementations](https://github.com/rust-lang/libs-team/issues/322):[^1]
```rs
struct IterMut<'a, T> {
    current: AssertThreadSafe<*mut T>,
    end: AssertThreadSafe<*mut T>,
    marker: PhantomData<&'a mut T>, // marker now used for correct Send+Sync, too
}
```

Returning from this detour, let’s now just declare auto traits by-example ✨
```rs
trait Foo {
    fn bar<T>(arg: T) -> impl Baz + auto<&Self, T>;
}
```

The syntax `impl Trait + auto<T₁, …, Tₙ>` takes an arbitrary number of parameters, and the `impl Trait` type then implements all auto traits exactly if (and only if) the types `T₁`, …, `Tₙ` do. Imagine some
```rs
impl<…> Sync for impl_trait_type
where
    T₁: Sync,
     …
    Tₙ: Sync,
{}
```
and so on, for all the auto traits.

[^1]: No more `unsafe` in this code example!? That’s unsound – or is it? No, not really, since doing anything that might break thread safety would need to dereference those pointers `current` and/or `end` and ***that’s*** still `unsafe`. 

WIP

### Lifetime captures in the same manner

WIP

### Compatibility concerns

WIP (elided “defaults”; negative reasoning and frozen model impls)

### `async fn`

WIP
