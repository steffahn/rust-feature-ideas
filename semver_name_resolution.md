*   Solve semver hazards from introduing traits (breaking method resolution) or defining items (ambiguity with glob-imports).
*   Add a way to annotate the precise minor version when a trait implementation or an item was defined.
    *   Either as an annotation on the `impl` / the item itself, (probably better)
    *   or as a separate file, automatically generated.
*   Either way, support for automatic generation is necessary.
    *   Cargo publish should check that those are present / generate them (but also manual generation is possible).
    *   Should affect every public item that’s visible. This is somewhat nontrivial.
    *   Automatic generation with external files would allows retroactively generating the information: You walk through your git tags chronologially and
        accumulate the required information into an untracked file, then add that file.
    *   In-code annotations have the advantage of following around with public-in-private re-exported items.
    *   Separate files go out-of-the-way easier, they don’t have that appeal of “changing the language” too much.
*   Goal: Remove semver hazards. Additional Goal: Simplify avoiding dependences on things that aren’t present in your claimed minimal supported version.
*   Allows non-magic reproduction of patterns like the `.into_iter` stabilization for arrays. The minimal dependency in `cargo.toml` serves the role of “edition”.
    Updating this version might thus break your code, however minor version updates to dependencies _without_ the MSV-update can **avoid** breakage that
    can currently happen.

*   An item can’t be used through a `::*`-import if it wasn’t included in your minimal supported definition. (And even without a `::*` import, we’ll want
    a warning in these cases.) Furthermore, metainfo-free crates could be made lower-priority for `::*`-item disambiguation, while warnings could/should be issued
    if there are multiple `::*`-imports with at least one of them being metainfo-free.

*   A trait method can’t be used with method-notation if its `impl` wasn’t in your minimal supported definition.

*   Might want to avoid strong coupling between the “edition” role and the actual dependency for ease of upgrading larger code bases… in this case you might want
    to allow more ways of accessing before-“edition”-version-items; without the decoupling, we could outright deny-warn all usage of such items / trait impls.
    However this stronger MSV checking could be extended further with other aspects, i.e. it’s not the main focus here and we should perhaps just not add more
    new warnings than necessary. If the roles are separated, the “edition” versioning could be independent and slower. However…… we’ll want to use this feature
    for `std`, too, that one already has “editions” for other purposes so carrying out the analogy too far as described is probably a bad idea.

*   Problems should not occur anyways, because we’ll want good tooling to auto-fix code on dependency upgrades.

*   A general approach of doing only warnings beyond what’s minimally necessary to avoid the main semver hazard also improves the situation of “not changing the
    language too much”.
