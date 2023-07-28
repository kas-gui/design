Widget input data
=================

-   Pull request: https://github.com/kas-gui/design/pull/6

**Scope:** ability to provide "input data" do widgets

**Status:** merged in [kas#396](https://github.com/kas-gui/kas/pull/396)


Motivation
----------

Introduction of "input data" to widgets has a huge impact on building Kas UIs.
There are several aspects to this motivation.

### Xilem

In [his blog introducing Xilem](https://raphlinus.github.io/rust/gui/2022/05/07/ui-architecture.html),
Raph Levien gives us a taste of how it could enable a very concise counter application:
```rust
fn app(count: &mut u32) -> impl View<u32> {
    v_stack((
        format!("Count: {}", count),
        button("Increment", |count| *count += 1),
    ))
}
```

Both Druid and Xilem build a UI description over custom app data, accessed via
mutable reference. This has some drawbacks (especially determining which parts
of the UI need updating when), but is a very powerful tool.

### Kas's shared data models

Kas already has models for shared data — but using these feels like using a
completely different UI framework. Not being built into the core framework they
also feel clunky, requiring their own "freshness" testing mechanism (version
numbers), a broadcast event (not really used for anything else) and a "driver"
library shadowing many core widgets.

Can we replace both standard and shared-data widget models with a new unified
model?

### Single specification

Kas's old widget model requires widgets with mutating content be specified both
with an initialiser and and update step. Extracted from the old counter example:
```rust
#[derive(Clone, Debug)]
struct Increment(i32);

impl_scope! {
    #[widget{
        layout = column! [
            align!(center, self.display),
            row! [
                TextButton::new_msg("−", Increment(-1)),
                TextButton::new_msg("+", Increment(1)),
            ],
        ];
    }]
    struct Counter {
        core: widget_core!(),
        #[widget]
        display: Label<String>,
        count: i32,
    }
    impl Self {
        fn new(count: i32) -> Self {
            Counter {
                core: Default::default(),
                display: Label::from(count.to_string()),
                count,
            }
        }
    }
    impl Events for Self {
        fn handle_message(&mut self, mgr: &mut EventMgr) {
            if let Some(Increment(incr)) = mgr.try_pop() {
                self.count += incr;
                *mgr |= self.display.set_string(self.count.to_string());
            }
        }
    }
}
```
The `display` element is mentioned four times here: the type field, an
initialiser, and an explicit update in `handle_message`, and position within
the layout. In contrast, the increment buttons are each mentioned only once;
the need to explicitly update `display` is the only reason it needs to be typed
explicitly (as a field) and given a separate constructor.

### Iced

Curiously enough, the final source of inspiration comes from Iced where a UI is
re-specified on every change. Both Iced and Kas have widget event-handlers
return messages, allowing another handler to enact their effects (in contrast to
Druid and Xilem where widgets may directly enact their effects).

Kas-using-input-data would therefore look superficially similar to Iced: a data
model with message handlers and a UI description. The latter part, however, will
look different since Iced widgets simply take a value as input while Kas widgets
would have to take some form of map from input data to value (or require prior
mapping of input data, as in Druid's lens system).


Basic premise
-------------

Widgets remain stateful (internal state which is not usually lost) but gain
input data. Input data may be mapped to a different type by any widget node in
the tree, including usage of local data (widget fields).

Unlike Xilem, input data is read-only. Like Iced, the only way a widget may act
(other than adjusting its own state) is by passing a message; as in prior Kas
versions any ancestor may handle the message.

Like both Iced and Xilem, the user should not normally need to call methods on
child widgets after construction; state management happens through
input data, input events and output messages.

### Exceptions: cached values and messages to siblings

There are some exceptions not covered by the above premise:

-   Sizing and drawing operations will not have access to "input data", hence
    sometimes values must be cached within widget state.
-   Communication with siblings is sometimes required, e.g. to set a filter on
    a list from an input field. This design document does not provide a direct
    method for this, though it can be achieved by copying the message to the
    state of a mutual ancestor, then passing as input data. Methods used by
    prior versions of Kas remain available, e.g. `Rc<RefCell<..>>` and calling
    methods on children of a custom widget.

Xilem's model (data passed by mutable reference) has an advantage here. The cost
is requiring a complex strategy to determine which parts of the UI must be
updated after an event handler is run. Druid stores and compares a copy of the
user data while Xilem uses an interesting (complex) tree of `Rc` values.
This direction of UI design has some merit and is perhaps the largest design
difference between Kas's and Xilem's widget models.

### Synchronising the UI with data

To keep widgets in sync with input data, we will add a new method,
`Widget::update`, and a requirement that this is called at least once between
any change to input data and calls to draw, resize or use event handlers on the
widget. For performance reasons, it is not required to call `update` on any
hidden children, provided `update` is called when those children become visible
(the `Stack` widget uses this approach).

We do not permit omitting a call to `update` where the data is `()` or is
unchanged from the last call. Reasons: (1) widgets may derive state from the
`ConfigMgr` instead, which may use `update` as a notification of change, and
(2) some input widgets allow direct manipulation of their state before sending
a message; a handler may determine that the input does not change any state and
call `update` with the prior state to re-set all dependent widgets.

Any widget may use its own state as input data so long as it calls `update` on
change to that data. For convenience, we provide a new widget, `Adapt`, which
both stores state and accepts user-defined message handlers, calling `update`
whenever any message is handled.


Input data how?
---------------

Focussing on the new `Widget::update` method noted in the requirements above,
there are a few possibilities for passing "input data" down the widget tree:

1.  Pass a simple fixed payload such as `u64`. This is a capability that the
    old `EventMgr::update_all` included and can be used to directly encode a
    little information (e.g a count or a pointer) in a known context. Throughout
    the whole widget tree this approach is inappropriate for anything beyond the
    simplest of UIs.
2.  Pass a value according to type parameter `T: 'static`. This allows custom
    data to be passed as needed throughout the widget tree, but has a couple of
    limitations: (a) data size is variable and may be large (expensive to copy)
    and (b) safely passing references requires `Arc`, `Rc` or similar.
3.  Pass a `&T` where `T: Sized`.
4.  Pass a `&T` where `T: ?Sized`. This implies that `&T` may be a fat pointer
    (e.g. `&str` is a pointer-to-data + length). This could be useful if Rust
    supported custom fat pointers (functionality that is likely far into the
    future if ever), but is fairly limited in utility otherwise.
5.  Pass a GAT, `T<'a>`.
6.  Pass a GAT by reference: `&T` where `T<'a>: Sized`.
7.  Pass variadic values (`Any`). A stack of such values could be passed via an
    attached context. The problems are three: (a) there is no attached lifetime
    (`Any: 'static`) hence references cannot be passed, (b) there is no
    compile-time enforcement of types, (c) downcasting everywhere is annoying.
8.  Pass some type multiplexing over ownership such as `MaybeOwned<'a, T>`.
    The only real utility over `&T` is that expensive-to-clone types could
    potentially be produced once and consumed once; the limitations are that
    matching values is now more complex and that widgets with multiple children
    need to convert `Owned(v)` to `Borrowed(&v)` or clone.

Options 1, 7 and 8 do not deserve further consideration. The other options
prompt two important questions:

-   Do we want an attached lifetime (to be able to pass references to temporary
    data)? The answer is quite obviously yes (e.g. `&T`), but leads to two
    sub-questions:

    -   Do we want to be able to pass multiple references, e.g. to combine
        temporary data from some outer context with inner state from an `Adapt`
        node? The answer is likely yes. There are two paths to this capability:
        "fat pointers" encoding multiple custom pointers (theoretically possible
        but well beyond todays stable or even nightly Rust) and Generic
        Associated Types (GATs) which are stable today; unfortunately
        object-safe GATs are not (thus `dyn Widget` is not available if
        `Widget::Data` is a GAT).
    -   Do we want to be able to construct temporaries in-line and pass
        references to children? This is a useful ability when passing references
        but means that methods like `Adapt`'s map function and
        `Widget::get_child` cannot return values but must accept a callback.
        In other words, this may be necessary depending on the model chosen and
        makes a number of APIs a little worse.

-   Should the size of the "data" payload be fixed? E.g. `u64` and `&T`
    where `T: Sized` are fixed while `T` and `&T` where `T: ?Sized` are not
    fixed. There are three components to this:

    -   Unfixed-size data payloads might be large. This is a potential concern
        with `T` and `T<'a>` but not an issue so long as the user has some
        option to pass larger values by reference.
    -   The more optimal `unsafe` design for the `Node` API (see
        [Introspection](#introspection)
        works by type-casting a widget object reference and a data reference.
        In effect, `Node` is a fat pointer. This does not work if data does not
        have fixed size (though the less efficient implementation does).
    -   Unfixed-size payloads (with GATs) would allow passing something in by
        value; e.g. `Text` could fix its input data type to `String` instead of
        using a closure to map data to a `String`. This comes with a major
        caveat: data is passed in to many methods without knowing which ones
        need to actually consume it, hence would frequently result in values
        being generated but not used. We do not consider this further.

    There is some interaction with other aspects of the design: e.g. if we
    wanted to use fat pointers, we could pass a reference to those (`&&T`
    where `T: ?Sized`); if we use GATs we could pass `&T` where `T<'a>: Sized`.

The only options meeting all our requirements and anywhere close to stable Rust
is to pass `T` or `&T` where `T` is a GAT. In today's Rust that should be
possible if we didn't need `dyn Widget`
([traits with GATs are not object safe](https://blog.rust-lang.org/2022/10/28/gats-stabilization.html#traits-with-gats-are-not-object-safe)),
however we do (both for `Window` and for layout macros, though hopefully Rust
will eventually get sufficient support for singleton structs with inferred types
to make this unnecessary for layout macros).
An incomplete nightly feature including this functionality exists,
[`generic_associated_types_extended`](https://github.com/rust-lang/rust/issues/95451),
but does not appear close to completion. We may choose to adopt this approach in
the future once the mentioned feature is stable.

The next best option is to pass `&T` where `T` is a type parameter or associated
type. In this case we could allow `T: ?Sized` but doing so has only a little
advantage (e.g. a temporary `str` could be passed as data instead of a `String`
but no custom fat pointers except through limited under-specified
[hacks](https://github.com/whentze/fat_pointer_hack)). Considering that
variable-size data payloads would limit the `Node` design (see
[Introspection](#introspection)) we will restrict `T: Sized`.

There is some contention in whether `T` should be a type parameter of the trait
or an associated type (sometimes bound to a type parameter of the struct):

-   A type parameter would let `Label: Widget<T>` for any `T` without
    requiring `Label<T>`. This sounds nice, but in practice it's not a big
    deal since `fn label(..) -> MapAny<A, Label>` is a viable alternative
    (and layout macros can support string literals as `Label`s either way).
-   An associated type allows e.g. `List<D: Directional, W: Widget>` instead of
    `List<A, D: Directional, W: Widget<A>>`. Not very significant.
-   An associated type is the only option for GATs and thus closer to what we
    may wish to use in the future.

The latter is the most important point here, so we will use an associated type.


Data models
-----------

Part of the motivation for input data is to unify "normal" widgets with data model
widgets like `ListView`. We can do this by requiring that the data passed into
`ListView` supports `ListData`, then passing `ListData::Item` to its children.

We wish to support a few forms of data models:

-   Single values shared by multiple widgets (`SingleView`): this will now be
    supported by "normal" widgets using input data
-   Simple static indexed data like `[&str]`
-   Generated data like in `examples/times-tables.rs`
-   Large data values from a DB cache passed by reference

The "simple" method of supporting this is for the `Item` type to be a GAT. Our
data traits can work with this, but view widgets must match the bound
`W: Widget<Data = A::Item>` where `A: ListData` which would require
that `Widget::Data` is also a GAT: more evidence that this is ultimately the
best option, but we can't do that without Rust supporting object-safe GATs.

The indirect method is to use an unparameterised `Item` associated type, but
have the getter return something else which can dereference/borrow to `Item`.
This is the current/old method. It works, but is not compatible with being
passed as input data *and* having `fn get_child` return a reference.


Introspection
-------------

A key part of Kas's design is that the widget tree supports introspection.
This allows, for example, `WidgetHierarchy` to print a representation of the
tree for debugging purposes without being implemented as part of the widget
model. It also allows much of the internals such as event sending to be
implemented via generic code instead of macro definitions. (Most internal uses
reduce to calling methods which are part of the `Widget` vtable anyway, but a
few do not: getting the `translation` of some descendant by `id`, drawing a
descendant by `id`, finding the `Rect` and cumulative `translation` of a
descendant by `id` (all used for pop-ups embedded in the widget tree), set
key-navigation focus to a child.) A further potential use-case is unit testing.

Introspection is still not considered an *essential* design element of Kas, but
it is sufficiently useful to keep if reasonably viable.
Introspection over only `Layout` methods would not require any adaptation to
input data, however this would exclude many of the internal uses. We thus choose
between:

1.  Restricting introspection to only `Layout` methods (which do not use input
    data). All recursive widget methods such as event-sending would be unable to
    use this API. Any widgets needing to provide their own implementation of
    `get_child` would still need to provide impls for "return a `dyn Layout`" as
    well as providing an introspection-capable or macro-syntax-only impl for
    use by event-sending implementations etc.
2.  A new introspection API combining a widget reference and data. We call this
    `Node` (and `NodeMut` using a mutable widget reference).

Since widgets like `List` which require a custom impl of `get_child` would need
to specify how to map data to children anyway, and since there are no other
major blockers, we choose the latter route.

Essentially, our `Node` is a custom fat pointer: a (mut) reference to a
`dyn Widget` (itself a fat pointer) plus a data reference (or value).
If only Rust natively supported such things... but it doesn't (yet).
Instead we can just use a struct:
```rust
pub struct Node<'a, T>(&'a dyn Widget<Data = T>, &'a T);
impl<'a, T> Node<'a, T> {
    // wrapper fns around the widget API here
}
```

Problem: any part of the widget tree which changes the data type (`Adapt`,
`MapAny`) would be unable to implement `get_child` returning the expected
`Node` type. We need to remove that parameter `T` from `Node`.

This is actually quite easy, if `T: Sized` and one is prepared to make certain
assumptions about the way Rust's vtables work: the vtable for
`dyn Widget<Data = T>` does not use `T` *except* as `&T` and we have a `&T`.
This `&T` has a fixed size (since `T: Sized`) and we have the appropriate value
handy. Simply casting `&dyn Widget<Data = T>` to `&dyn Widget<Data = ()>` and
the associated `&T` to `&()` should do what we want. Yes, this is `unsafe`. Yes,
this depends on under-specified parts of Rust. But it does appear to work and
can be offered as an option (currently feature-gated behind `unsafe_node`).

If we don't accept the above `unsafe` approach we can still implement `Node`,
but with significantly more complexity: introduce a new (internal only) trait
for use by `Node` methods and implement over the tuple
`(&'a dyn Widget<Data = T>, &'a T)`. We then need to store this tuple inside
`Node` and type-erase to `dyn NodeT + 'a`; since the latter is unsized we could
implement this as:

1.  `Node<'a>(Box<dyn NodeT + 'a>)`: simple but requires an extra allocation
    and dereference
2.  `Node<'a>(stack_dst::Value<dyn NodeT + 'a, stack_dst::buffers::_>)`:
    avoids the indirection and overhead of boxing at the cost of `unsafe`
    (but at least over better specified parts of the Rust langauge)
3.  `Node<'a>(&'a dyn NodeT)`: apparently simple but requires that `Node::new`
    take a tuple instead of two parameters, further requiring that
    `Widget::as_node` cannot return a `Node` (must use a callback or be removed)

For now, option (1) appears the most sensible: low disruption and achieving
the goal of avoiding `unsafe`. Possibly (2) could be offered as an option but
it makes little sense when the prior `unsafe_node` is already an option (unless
that is determined to be unsound).
