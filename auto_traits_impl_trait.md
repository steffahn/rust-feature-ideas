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

WIP

### Lifetime captures in the same manner

WIP

### Compatibility concerns

WIP (elided “defaults”; negative reasoning and frozen model impls)

### `async fn`

WIP
