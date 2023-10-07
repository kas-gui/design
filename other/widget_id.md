WidgetId
========

-   Pull request: [#10](https://github.com/kas-gui/design/pull/10)
-   Other relevant links here

**Scope:** widget identifiers (paths)

**Status:** implemented; some alternatives considered


Introduction
------------

A UI consists of a widget tree (plus various state and interfaces). It is frequently useful to be able to store an identifier for some widget, for example to allow event-handling state to "remember" which widget has keyboard focus. This is central to Kas's event-handling model, allowing events to be sent to a specific widget instead of merely broadcast.

For this we require an "identifier" type, `WidgetId`, which is:

-   Small
-   Unparameterised (especially lifetime parameter)
-   Supports `Copy` or at least `Clone`
-   Supports `Eq` or at least `PartialEq`
-   Default-constructible to some invalid value (allowing identifier assignment after construction and operations to panic on invalid values)
-   Reproducibly generable from any widget in the tree
-   Robust against resizing and scrolling (so cannot simply use coordinates)

Complication: view widgets (see e.g. `ListView`) may be reassigned while scrolling the view. A `WidgetId` should identify the virtual (data-mapped) widget not the (reusable) backing widget so that, for example, focussing one entry in a list then scrolling away does not result in some other list item
gaining focus. In other words, the `WidgetId` should be reproducibly generable for the path through the UI to the data item.

With this in mind, our `WidgetId` type should ideally also:

-   Be robustly reproducible when elements of the UI are added/removed/re-arranged
-   Be reproducible for some "path" through the UI tree, including where path elements are derived from some data index (or other identifier)
-   Support a fast "is x an ancestor of y" test to allow efficient traversals of the widget tree
-   Support `Hash` and `Ord`, allowing usage as map keys
-   Be "non-zero", allowing `Option<WidgetId>` to have the same size as `WidgetId`


Design
------

This is documentation of the existing (implemented) design of `WidgetId`, which meets most of the above requirements.

To allow robust representation of identity, we identify widgets via a path where:

-   The first component is the window number (1-based index)
-   Usually, other components are the child index as used by `Layout::get_child`, but...
-   Widgets may use a custom mapping (e.g. `List` does to persist the child-component relationship given insertion and removal of other children)
-   One (or multiple) path components may be derived from a data identifier as used by a view widget

Clearly, such paths cannot always be stored in a small, fixed-size type, however it turns out that many (even the vast majority) can. We use a hybrid design with three variants:

-   Invalid (the default-constructible value)
-   Inline
-   Pointer to allocation

### Inline variant

Most widgets in Kas UIs have a small, fixed number of children, thus most path components are some very small number. Inspired by UTF-8, the inline variant uses variable-length encoding of path components bit-packed into 4-bit segments of a `u64`:

-   Four bits are used for flags (identify invalid / inline / pointer variants while avoiding zero)
-   Four bits are used to encode the length
-   The remaining fourteen 4-bit segments are used to encode path components

Each 4-bit segment may represent a value in `0..=7` directly in its last three bits, or may set its first bit to indicate a continuation to the next 4-bit segment.

This enables paths with length up to 14 to be stored inline, depending on the value of each component. Paths which do not fit use the allocated variant instead. In practice it appears that the vast majority of identifiers fit the inline variant, though this is be application-dependent.

### Allocated variant

This variant stores the path as a heap-allocated `[usize]` with attached reference-count.

The allocation is reference-counted, allowing cheap `Clone` support.

The pointer may be stored in the same `u64` used for the inline variant. Note that a `[usize]` must be at least 4-byte aligned, thus the last two bits of the pointer must be zero, allowing these bits of the backing `u64` to be used for flags.

### Ancestor test

Since a `WidgetId` stores the full path, it is trivial to test whether one is an ancestor of another. Further, path components may be extracted, though since the meaning of path components is arbitrary the parent widget must be used to convert component(s) to a child index.

### Display implementation

We display the inline variant as a `#` followed by a hex dump of the used 4-bit segments. Allocated variants are displayed using the same system. Some examples:

-   `"#" = []`
-   `"#1" = [1]`
-   `"#197 = [1, 15]`
-   `"#123 = [1, 2, 3]`
-   `"#d81" = [321]`
-   `"#59195 = [5, 9, 13]`

### Drawbacks and discussion

To allow correct memory management of allocated variants, this `WidgetId` type requires non-trivial `Clone` and `Drop` implementations. In particular, this means that it is not `Copy`. Since the implementation uses reference-counting, `Clone` and `Drop` are suitably fast, but the requirement that users `.clone()` values is annoying. Alternative designs could allow a `Copy` type, but with other compromises (see below).

Using bit-packing to compress path components is not free, however given how modern CPUs are fast relative to memory access times and how this design allows many identifiers to avoid allocation altogether it is likely a worthwhile trade-off. This has not been investigated in detail (nor has compression of allocated variants been investigated).


Alternatives
------------

### A counter or random number

The simplest forms of unique identifiers are generated on demand (in unspecified order) using a counter or random number generator. Such identifiers lack support for:

-   Reproducibly identifying virtual (data-mapped) widgets
-   A fast ancestor check

#### Depth-first enumeration

Early versions of Kas assigned widgets a unique identifier during configuration (depth-first tree traversal) via a counter. Compared to the simple variant above:

-   Identifier ordering matches that of a depth-first visit of elements. This allows widgets to implement a form of fast ancestor check and efficient traversal to an identifier.
-   Widget identifiers are not stable: adding/removing/re-arranging items in the UI may change widget identifiers. Attempting to make this stable for view widgets would require an assumption about the size of the sub-UI for each view widget.

#### Sub-maps

Use a counter, but let mutable widgets like `List` assign from a "sub-map" (a new counter). The full widget identifier is thus `(submap, index)`. This potentially fixes the stability issue, however implementing an ancestor check without full tree traversal now requires storing a map of submap paths; memory management for this map gets complicated if a view widget may use a submap.

### Variations on path identifiers

#### `u128`

Kas could provide a crate feature to use `u128` instead of `u64` as the backing type, probably allowing inline storage of up to 30 components.

Caveat: the little-used `fn WidgetId::as_u64` method would no longer work so well (at the least, equality tests on such values may have some false positives).

#### Copy (leaking or GC) variants

A very simple approach to making `WidgetId: Copy` would be to simply never free any allocated variants. This makes `Clone` and `Drop` trivial while keeping dereference safe since it is known (within the lifespan of the program) that the pointed memory is never de-allocated.

One could hypothetically use some form of garbage collection to scan for unused allocations to avoid memory leaks, however Rust does not provide a suitable run-time for (user-defined) garbage collection, nor would this be in the spirit of the Rust language.

It is worth noting that if this design were ported to a language or run-time with garbage-collection (while also sufficiently low-level to support this design), the type could easily support the equivalent of `Copy`.

#### Implicit clone

In some ways it would be nice if Rust supported implicit clone operations, allowing cheaply clonable types like `Rc<_>` and `WidgetId` to *appear* to use copy semantics while still performing reference-counting operations under the hood.

For better or worse, Rust decided against this option. Maybe we should just accept having to write `.clone()` everywhere?

#### Owning / copy variant

A different approach to making "ordinary identifiers" support `Copy` is to use two variants, `OwnedId` (`Clone`) and `Id` (`Copy`).

Functionally, an `Id` is identical to an `OwnedId` except that liveness must be proved before pointer dereference. We are thus trading the need to `Clone` and `Drop` identifiers for the need to perform some liveness lookup in a static or thread-local database, plus the cost of maintaining (and garbage-collecting) that database (likely not a good trade).

Further, though this would allow users to freely copy `Id` values around (no need to clone or pass by reference), users have to accept *two* types of identifier in the public API. Binary methods on identifiers such as `is_ancestor_of` and `next_key_after` would now be expected to work with any combination of `Id` and `OwnedId`, requiring that these be moved to traits. Overall, the size of the public API around identifier types will likely be more than double what it is with a single identifier type.

One could try to get around this by making an `OwnedId` dereference to an `Id` (thus requiring all operations on an `OwnedId` to unnecessarily perform liveness checks) or an `Id` derefence to an `OwnedId` (performing the liveness check in the dereference, with panic on failure), or both dereference to a third type (effectively the same as one of the prior options).

Note that this is *not* akin to `std::rc::Weak` which does not support `Copy`.


Potential changes
-----------------

The name `WidgetId` is cumbersome; potentially this could be renamed to `Id`. The name `Path` is in some ways more accurate, however this may be confused with `std::path::Path`.

Currently, many methods take a `WidgetId` by reference. We could prefer passing the type by value, which in some cases requires extra reference-count operations but otherwise requires one less memory dereference. It would allow removal of the `Layout::id_ref` method (replacing with `LayoutExt::id`), simplifying the API slightly.
