Initial notes: (TODO, cleanup/rewrite)

> Without function abstraction: DOWNGRADE – Mutable, later immutable reference can be taken, maybe just with `&mut`-expression.
> These can be projected to fields and through Box and `&mut` indirections. UPGRADE, take reference without really taking a reference,
> upgrade to immutable/mutable reference later. Also uses `&mut`. Related to two-stage-borrow. The preliminary reference can be projected
> to references to fields, no indirections. While they exist, the underlying value cannot be moved (although &mut-interactions and take_mut
> is still okay; actually – maybe moving out and then back in locally can be okay, too?) UPGRADE2: Take mutable reference later, initially just shared.
> The shared reference means we CAN go through indirections. Basically downgrade in reverse. More complex multi-down/upgrade paths are possible, too.
> DOWNGRADE2, upgrade(1) in reverse, allows re-upgrading. Solves https://internals.rust-lang.org/t/local-closures-borrowing-ergonomics/15129
>
> Towards function abstractions: Mutability-generic functions could support downgrading. Most things to need extensions of the type system.

<hr>

Thoughts:
*   A function like <code>[std]::[cell]::[RefCell]::[get_mut]</code> cannot support downgrading <code>[&mut] T</code> to <code>[&]T</code>
    because there’s also <code>[RefCell]::[borrow_mut]</code>.
    *   ⟹ So we only want to support “[`mut`]-generic” use-cases after all?
*   Light-weight generics syntax, reuse existing lifetime-markers to figure out connections. Upgrading existing methods.
    *   E.g. unify
        ```rs
        pub const fn as_ref(&self) -> Option<&T> { … }
        ```
        with
        ```rs
        pub fn as_mut(&mut self) -> Option<&mut T>
        ```
        to something like
        ```rs
        pub fn as_ref(&[mut] self) -> Option<&[mut] T>
        ```
        or
        ```rs
        pub fn as_ref(&mut? self) -> Option<&mut? T>
        ```
        This method would then support downgrading.
    *   On the topic of [`RefCell`], don’t unify <code>[RefCell]::[borrow_mut]</code> and <code>[RefCell]::[borrow]</code> since they have significantly different
        run-time behavior.
    *   Needs systematic coverage of all kinds useful borrowing settings, e.g.
        *   with or without support for downgrading to “actually **n**ot **b**orrowed at **a**ll” (nba) kind of state
        *   with [`&mut`] or [`&`] or `&nba`-level requirements for the duration of the function call
        *   with in [`&mut`]-to-`&nba` generic case, support for working with or without an intermediate [`&`]-state.
            *   The example of <code>[RefCell]::[get_mut]</code> could support up-/downgrading between [`&mut`] and `&nba`, just no intermediate [`&`]-state.
                *   Actually, <code>[RefCell]::[get_mut]</code> can also support working with `&nba`-level access for the duration of the function call itself,
                    i.e. two-staged-borrow-esque interaction.
            *   Is there any existing API that would become unsound with an [`&`]-state-less up-/downgrade?
    *   Maybe syntax suggesting “lazy” borrowing for supporting the not-borrowed-at-all state during the call? My Haskell mind says using `~`, so
        something like `~&T` and `~&mut T` and `~&[mut] T`? Or `&~T` and `&mut ~T` and `&[mut] ~T`? Or `&~T` and `&~mut T` and `&~[mut] T`?
        Where does the lifetime go for the last ones?
*   Probably still also needs support for “explicit” generics – also [compare `brainstorming.md`], I guess.
    *   In this case, the implicit version leaning on lifetime relations might only be a “lifetime elision”-like convenience feature, desugaring to some
        “fully featured” mutability generics? 
*   Another use case is “deferred” execution at the end of scope through descturctors. `&nba` borrows that only materialize for the destructor call would be
    needed. This assumes some way of annotating upgrading-requirements for destructors.

[std]: https://doc.rust-lang.org/std/index.html
[cell]: https://doc.rust-lang.org/std/cell/index.html
[RefCell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[`RefCell`]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[borrow]: https://doc.rust-lang.org/std/cell/struct.RefCell.html#method.borrow_mut
[borrow_mut]: https://doc.rust-lang.org/std/cell/struct.RefCell.html#method.borrow
[get_mut]: https://doc.rust-lang.org/std/cell/struct.RefCell.html#method.get_mut
[&mut]: https://doc.rust-lang.org/std/primitive.reference.html
[`&mut`]: https://doc.rust-lang.org/std/primitive.reference.html
[&]: https://doc.rust-lang.org/std/primitive.reference.html
[`&`]: https://doc.rust-lang.org/std/primitive.reference.html
[`mut`]: https://doc.rust-lang.org/std/keyword.mut.html
[compare `brainstorming.md`]: ./brainstorming.md#:~:text=Mutability%20generics
