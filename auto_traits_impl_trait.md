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

The meaning of leaving out the `auto<…>` declaration still leaves the implementation implicit/inferred (and thus leaks implementation details w.r.t. the auto trait impl), except
in traits, where it means the same as `auto<Ty>` for some marker type `Ty` that doesn’t implement any auto traits at all. If it's present however, then no leakage occurs.
The potentially stronger auto-trait implementations of the actually-returned type are *not* made available to the user.

The code example above features `&Send` in an `auto<…>` bound. This seems the most convenient option but poses the question of whether (and how) to support elided lifetimes in this position.
If they are supported, then there's also the question of what they mean; do they refer to a different lifetime in the function input, of do they behave more like a HRTB? The HRTB approach
has the benefit of being more generally applicable; the idea would be that e.g. `auto<&Self>` means that the return type's implementation of an auto trait such as e.g. `Send` is now requiring `for<'a> &'a Self: Send`.
Almost all implementations of auto traits will not care about the particular choice of lifetime anyways. On the other hand, something like `<&T>::Associated` might not work this way if `Associated` comes from a trait
bound that only is stated for some concrete lifetime.

Multiple `+ auto<…> + auto<…>` should probably be just prohibited. The notationis also disallowed in all places that are not `impl Trait` return types (or equivalently, TAIT definitions).

When `auto<…>` is combined with an explicit auto trait bound, like `+ auto<T> + Send`, then both bounds count, i.e. for `Send` the plain, unqualified `Send` is stronger, and the return types of the function `Foo<T, …>` will
be implementing `Send` unconditionally, but e.g. `Sync` only if `T: Sync`.

Expressing that some auto trait is not implemented can be done by listing an appropriate marker type. E.g. to express `!Uinpin`, you just add `PhantomPinned`, like `auto<…, PhantomPinned>`.
This might motivate the addition to additional marker traits like `PhantomNotSend`, `PhantomNotSync`

Trait implementations can adjust `auto<…>` annotations to be less restrictive, or drop them to make the concrete implementation leak auto-trait info once again
(for not overly generic users that can resolve to this concrete trait implementation).

There are existing patterns around `async fn` for traits where you might want multiple variants of a trait with different `Send`-ness of its functions. As far as that is concerned, it seems possible to
consider additional type (or `const bool`) arguments to the trait which are then used by a marker type in the `auto<…>`. Though I think this needs further exploration.

### Lifetime captures in the same manner

With this `auto<…>` annotation as precedent, I’m looking at the `+ Captures<'lt>` pattern and thinking that it could benefit from similar keyword-based variadic syntax language-support.

`+ capture<'a, 'b, 'c, …>`

The best precise interaction with `+ 'a` are yet to be figured out (I'd assume that at least some warnings in some cases, like `captures<'x, …> + 'y` without `'x: 'y`, are very reasonable/desired),
and it's also a good question to ask if and how it should apply to type arguments as well.

An explicit `capture` syntax can also help with edition-2024 changes for free-standing `impl Trait`-returning methods. The default can just change, and auto fixes can rewrite
previously implicit cases with some lifetimes left uncaptured into now-explicit syntax without changing meaning. E.g.

```rs
impl MyMapLikeScructure<K, V> {
    fn my_iter(&self, key: &K) -> impl Iterator<Item = V> + '_ { … }
}
```
can automatically get rewritten on edition upgrade to
```rs
impl MyMapLikeScructure<K, V> {
    fn my_iter(&self, key: &K) -> impl Iterator<Item = V> + '_ + captures<'_> { … }
}
```
The simple but common case of `captures<'a> + 'a` where the same single lifetime appears in `captures<'a>` and as a `+ 'a` bound, the latter can probably also be automatically dropped safely by this.

Another interesting question would be whether you can decide *not* to capture any of the lifetimes that appear in other places in the trait bounds. (This would probably need to imply some sort of generalized 

### Compatibility concerns

If `auto<…>` is not specified explicitly, existing behavior continues, as outlined above already. This can also be expressed in a sort-of desugaring of elided `auto<…>` bounds.

For normal functions or methods (i.e. not in trait definitions) the “desugared” version of the signature is simply `+ auto<ActualReturnType>`. It is not an entirely truly a desugaring since 
many use cases of `-> impl Trait` return types involve anonymous types. For methods in trait definitions (not trait implementations though), the desugared version is something like `+ auto<Marker>`
for some marker type that doesn't implement any (relevant) autotraits; i.e. similar to `auto<*const (), PhantomPinned>` with “marker” types today,
or `<PhantomNotSendSync, PhantomPinned>` if we get a more dedicated marker type here for `!Send + !Sync`.

There is one issue here. Say something declares `auto<*const ()>` and then returns something that captures `*mut i32` or `Cell<bool>`. All of these are `!Sync`. But *really* what the compiler wants to
ensure here is that the return type such as `Cell<bool>` fulfills `Cell<bool>: Sync` if `*const (): Sync`. Which isn’t really the case today, arguably. TODO…

WIP (negative reasoning and frozen model impls)

### `async fn`

WIP

### More niche auto traits, and new auto traits

The discussion above only acknowledges `Send`, `Sync`, and `Unpin`. However we also have `UnwindSafe`, `RefUnwindSafe` and unstable `Freeze` in the standard library.

`UnwindSafe` and `RefUnwindSafe` are easy for `async fn`: futures from `async` blocks or `async fn` are always unwind safe (they do poisoning on panic).

And in general, one usually doesn’t want to worry about `UnwindSafe` too much as they’re an overly strong lint, for most purposes. I have other general ideas (not written down) about those
that should perhaps allow downgrading trait mismatch on `UnwindSafe` and/or `RefUnwindSafe` to just a *warning*, anyways, so I don’t want to warry too much about it.

For `Freeze`, there are stability concerns. I have not looked into it too deeply. There are other implied (leaked) properties of types, too, anyways, such as variance (I haven't look into
the question of whether that's leaked through `impl Trait` at all) or what parameters are used without or only with indirection (which is relevant for recursive type definitions; not sure
here either, how `impl Trait` handles it currently, and if this can even be a problem to run into, at least without TAIT).

For `Freeze`, additional properties, and possible future auto trait extensions, I believe the safest approach is to exclude them from `auto`. This means for such properties, the current rules still apply,
they are leaked in ordinary functions, and conservatively assumed absent in trait methods (in generic usage contexts).
