Introspection
=============

-   Pull request: TODO

**Scope:** ability to walk through / inspect / call into a widget tree beyond the top-most node.

**Status:** implemented, under review


Introduction
------------

Kas (like many toolkits) models a User Interface (UI) as a tree of widgets.
Should it be possible to walk through an existing tree to inspect or mutate
nodes or even to add, remove and replace nodes?

### Existing user-interface

The `WidgetChildren` trait provides this interface (a sub-set of methods):
```rust
pub trait WidgetChildren: WidgetCore {
    fn num_children(&self) -> usize;

    // The following expect index < self.num_children():
    fn get_child(&self, index: usize) -> Option<&dyn Widget>;
    fn get_child_mut(&mut self, index: usize) -> Option<&mut dyn Widget>;
}
```

### Motivation

Existing uses of this interface:

-   (mut) Drivers for widget configuration, event sending and keyboard
    navigation (may be moved inside `Widget`'s dyn-safe API)
-   (mut) Event manager code associated with keyboard navigation
-   (mut) Implementation for pending configures
-   (const and mut) `RootWidget` for pop-up management (likely to be replaced
    with a system not requiring introspection in the future)
-   (const) `WidgetHeirarchy` debugging facility
-   (const) `ComboBox`, just to get the `WidgetId` of an entry (could use an
    alternative approach)

In summary, having introspection allows:

-   Aspects of Kas's internals to be implemented externally of the `Widget` trait
-   The addition of features *without* justifying additional functionality on
    the `Widget` trait itself. For example, `WidgetHeirarchy` can be used to
    print a summary of any part of the widget tree for debugging purposes;
    without introspection it is doubtful that it would exist.

### User-defined widgets with children

Note that any user-defined widget with children must have an implementation for
event routing (among other methods). Early versions of Kas did require
user-defined implementations of event routing for some complex widgets, but
these tended to be hard to understand and error-prone.

Introspection allows the *sending* part of event-routing to be implemented
externally. It also allows tweaking aspects of event-routing with minimal
effect on user-defined widget code.

Alternatively, event-routing could use macro-generated code for widgets with
fixed children (using the existing `#[widget]` attribute on fields), but this
would not support more complex cases such as `List`, `TabStack` or `ListView`.
(To be clear, these container widgets could still be implemented within the Kas
library.)

If we didn't have introspection, some of the above would have to be replaced
with additional methods on `Widget` (some of which is planned anyway), some
would require re-writing a feature to work differently, and some features may
just be removed.


Design
------

Uses of introspection sometimes access a single (targetted) node, and sometimes
perform a depth-first traversal of the tree. The implementation of
<kbd>Tab</kbd>-navigation performs a complex partial traversal of the tree.

The current API is the simplest possible: `len`, `get` and `get_mut`.
Some convenience methods for searching the tree for a particular `WidgetId` are
provided over this API.

### Mutable introspection?

The most important use-cases (in Kas's core) require mutable introspection.

No particular need for mutable introspection outside of core Kas code has been
observed, hence the mutable introspection API (`get_child_mut`) could be hidden
and not supported for external usage — except that some user-defined widgets
must implement `get_child_mut`, hence it cannot truely be hidden.

### Constant introspection?

Non-mut introspection is sufficient observed non-core functionality: for the
implementation of `WidgetHeirarchy` debugging utility and for the usage within
`ComboBox`.

Could we simplify by reducing to a `mut`-only API? Yes; the [#387 draft] does
this. But should we? This forces the `WidgetHeirarchy` helper to require a `mut`
widget. The primary motivator for making the change in this draft is to avoid
the need for a non-`mut` variant of the `Node` API, but this is not a strong
motivator (especially given that this variant would have a much smaller API).


Conclusion
----------

We have a good rationale for keeping introspection.

Two changes to the existing design are proposed above:

-   Hide and/or do not support usage of the `mut` introspection API outside of
    Kas core code: not viable since we rely on users to implement it sometimes;
    this also lacks real motivation
-   Reduce to a single `mut`-only design: a small simplification but with
    limitations and insufficient motivation

[#387 draft]: https://github.com/kas-gui/kas/pull/387
