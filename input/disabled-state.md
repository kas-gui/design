Disabled state
==============

-   Pull request: [#2](https://github.com/kas-gui/design/pull/2)

**Scope:** document what it means for a widget to be disabled and how this is
implemented.

**Status:** implemented (excluding aspects requiring *input data*)


Introduction
------------

Some apps *disable* parts of their UI without removing it. Such disabled items
should:

-   Remain visible so that layout does not change and confused users do not
    need to search for a "missing" feature
-   Be visually de-focused relative to enabled controls
-   Not be interactive so that users cannot enter irrelevant information

### Requirements

The app should specify what is disabled.

It seems sensible to apply disabled status recursively (all children of a
disabled widget are automatically disabled).

### Examples

The gallery chooses to disable tab pages without disabling the tabs
themselves or the scrollbars / scroll-regions. *This is a choice.*
It may achieve this by disabling the root widget inside the tab's scroll region.


Effects
-------

-   Input controls like buttons and sliders should not be interactive.
-   Text selection should *probably* be disabled, though this is debatable.
    In any case `EditField` supports read-only mode (though this won't work if
    a parent is disabled and does not pass events).
-   Scrolling and scroll-bars should *probably* be disabled. I thought
    differently about this in the past due to the behaviour implemented in the
    gallery, *but this should be a choice of the app*.
-   Responding to input-data updates should probably still happen.
-   Animations such as momentum scrolling or radio-box de-selection should
    probably happen; unclear.


Design
------

Disabled status is tracked by `EventState` with the following public API:

-   `fn is_disabled(&self, id: &WidgetId) -> bool` (true when `id` or any ancestor is disabled)
-   `fn set_disabled(&mut self, id: WidgetId, state: bool)`

Event-sending will only deliver a few events to disabled widgets, controlled by
`Event::pass_when_disabled(&self) -> bool`. Other events may still be received
by ancestors during unwinding.

Prior to the introduction of input data, the Gallery example uses a trait to
selectively en/dis-able children. Once input data is available, that should be
used for en/dis-abling widgets.

Possibly add a wrapping widget to en/dis-able its child:
```rust
let ui = Disable::new(child, |data| data.disable);
```
