Explicit syntax for declaring whether or not a trait is object safe.

## Motivation

The main issue with the current system is that it’s easy to miss changes of object safety to a trait. This has even happened to `std` once.
This basic idea is also proposed in [an open RFC pull request here](https://github.com/rust-lang/rfcs/pull/3022).

The forms this can take are various. On one hand, there’s the question on syntax. Syntax can be attribute-based or using custom syntax.
E.g. to mark a trait as object safe, one could imagine
```rs
#[object_safe]
trait Foo {
    fn foo(&self);
}
```
```rs
dyn trait Foo {
    fn foo(&self);
}
```

The `dyn trait Foo` syntax could maybe be related to [RFC 3323 “Restrictions”](https://rust-lang.github.io/rfcs/3323-restrictions.html).
Visibility could perhaps be specified in general RFC 3323 fashion as a restriction on using the trait object type
(or think, *visibility* of the trait object type), as `dyn(self) trait Foo` or `dyn(crate) trait Foo`,
at least for object-safe traits that are not supposed to expose this object safety. This could even facilitate
niche use-cases such as traits that are currently object-safe, whose trait objects are used internally in a crate, but
externally one wants to keep that object safety a non-commitment.

## Defaults & Editions

Apart from the choice of syntax there’s also the choice of defaults. Related is the choice of whether a `!dyn` / `#[not_object_safe]` syntax
is possible and/or necessary. The linked RFC pull request has proposed `dyn trait Foo` only as a *test*, i.e. a static assertion that object
safety is given. This can be contrasted with approaches that make object safety entirely opt-in (over an edition). And another way is
to make object safety inferred but opt-out, and optionally tested.

The last general approach of testing only that a trait is object safe and not that it *isn’t* object safe has advantages in that
*	it minimizes the things you need to annotate (i.e. instead of writing `dyn` or `!dyn`, you only need to write something for the `dyn` case)
*	you can get consistent rules for public and private items, and just not annotate the private ones (because semver doesn’t matter there)
	*	“private” shall include public-in-private here… linting rules should be adapted to handle those cases (at least mostly) correctly
   		in order to support consistently linting against unannotated object safe public traits.
*	it prevents accidental “object safe ==>> not object safe” transitions. The opposite transition is not possible in a trait without an (obvious)
	breaking change (changing the signature of an existing trait method), anyway.

The “big” disadvantage is that it still allows for issues with “meta breakage”. If Rustc wants to further liberalize the supported set of signatures
that can do object safety, then that *can* be a kind of “not object safe ==>> object safe” (though in this case, the arrow goes up in compiler version,
not crate version) which can then consequently turn a crate-level “not object safe ==>> not object safe” transition into either
of “not object safe => object safe” or “object safe => not object safe”, neither is particularly nice, the latter even semver breakage.

The solution could be to limit such upgrades to edition boundaries, so that the inferred object-safety rules only change over editions. Before the switch,
lints would point out traits that need to add an opt-out in order to stay *not* object safe. Principles of edition migrations then also prescribe that this
opt-out must work cross-edition, which gives an argument for the opt-out to be something that works *even* for traits that weren’t object safe to begin with.
(Consequently, approaches of “annotate everything” are supported again!?)

In the future, `DispatchFromDyn` could even be made available for custom types, which brings this kind of breakage potential exists on a library level,
and editions can no longer help.

The always-opt-in style approach has the *huge* disadvantage that *private* traits don’t care about the semver. In order to apply *different* rules
between private and public traits, one would however need a solid concept of “private vs. public”, where any proper such notion should consider
“public in privat” as “private”, but well… do we *want* a simple `pub use` somewhere change the properties of a trait!?

Approach of annotating *every* public trait have the advantage that you never prefer object safety or non-object safety purely based on the “easier syntax”,
and also that the declaration then matches the documentation (since rustdoc should *definitely* properly document object safety of a trait).

## Bigger context

Semver concerns with inferred properties is not unique to object safety of traits. Other examples include properties of structs (like variance,
auto traits, generics liveness during `Drop`, immediately vs. indirectly contained types for recursive structs, unsizing support), or properties of
opaque types (`impl Trait` return types, or futures of `async fn` – particularly the auto traits `Send`/`Sync` for either of these).

Explicit annotations, and the question whether those are tests or prescriptive, is relevant in all such cases, and could benefit from a neat
& universally applicable solution.
