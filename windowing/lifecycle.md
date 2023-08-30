Window state and lifecycle
==========================

-   Pull request: [#7](https://github.com/kas-gui/design/pull/7)

**Scope:** window and widget state lifecycle

**Status:** documentation of status quo with proposed modifications to support
suspended / hidden windows. "Secondary windows" secion is incomplete.


Introduction
------------

Kas has so far been designed and tested only on desktop platforms. As such:

-   Primary windows are constructed on application start
-   Additional windows may be constructed from event handlers
-   There is no mechanism to "re-open" a closed window
-   In contrast, menus currently use widgets owned by the parent window which
    may be opened as a child layer
-   The application exits once all windows have been closed

Limitations:

-   This does not match the required application lifecycle on mobile platforms
-   There is no proper support for modal windows or menus on desktop platforms


Primary windows
---------------

Definition: a *primary window* is an application window without a parent.

All GUI apps require at least one primary window, though it is not always
created on application start. Some platforms may only support one primary
window.

A primary window consists of the following "state":

-   A `kas::Window` (which contains a `Box<dyn Widget<Data = AppData>>`)
-   A `kas::WindowId`
-   An instance of `EventState`

It also contains some "cache":

-   A `kas::layout::SolveCache`
-   A theme window
-   Animation timers

Finally, it includes "window components":

-   A `winit::window::Window`
-   Potentially a clipboard handle
-   A drawable surface

The user is responsible for explicitly creating primary windows, usually on
application start, though potentially later from event handlers.

-   All "state" is constructed at or before creation of the window and persists
    until application exit or the window is explicitly and irreversably closed
-   "Window components" are created on Winit's `Event::Resumed` and destroyed on
    `Event::Suspended`
-   "Cache" components may use either of the above models.


### Hidden windows

A primary window may be hidden. In this case, no new "window components" will be
created on `Event::Resumed`. A hidden window will never be sized or drawn or
receive events.

Any primary window may be hidden or unhidden from its own event handlers or via
a `WindowId` from another window or proxy. When hidden, the widget will be
suspended and the window components destroyed; when unhidden, window components
will be created and the widget configured and sized.

**Note:** it may be desirable that a hidden window can be configured and receive
both data updates (`fn update)` and timer updates (`Event::TimerUpdate`). Such
functionality is left to a future design document.


Popups
------

A popup (e.g. a menu or tooltip) is driven by a widget within a parent window.
This implies that its state exists as long as the parent window does and that
its input data is filtered through the parent window's UI tree.

### Emulated vs using the windowing system

All desktop platforms should have some form of support for such surfaces, e.g.
Wayland has `wl_shell_surface::set_popup`. Unforunately Winit does not yet
support this (issue #403).

Mobile platforms may not support such surfaces at all.

As a result we should support emulating such surfaces (with the restriction that
the surface may not escape the bounds of the parent), but use window manager
surfaces when available.

Since only emulated popups are currently supported, it is yet to be determined
what other changes will be required for window-manager provided popups.

### Construction and state

It is desirable by the current stateful widget model that a popup surface may
be driven by an arbitrary widget in the tree of the parent window (using a
widget excluded from layout).

State: a popup has a `WindowId` only when open. A popup does not have its own
`EventState`, `SolveCache` or theme state. It will need its own draw surface.

Configuration and updates: this may happen along with the parent widget or only
on usage. This may be affected by the state-retention model.

Layout: sizing must happen no earlier than configuration and before first usage;
likely it should happen when first opened. It could be repeated (and this might
be required if configuration is repeated). The position should be set on each
usage (at least if anything changes).


Modal windows
-------------

Dialog boxes such as "Open File" are a form of modal window: one that on
desktop platforms usually appears as an independent window, but which should
block access to the parent window while open (for example, the user must
choose a file to open or cancel this task before being able to interact with
the parent window again).

Some aspects of modal windows may be different to popups:

-   The modal window could potentially appear on a different screen with a
    different scale factor, thus may need independent theme window state.
-   Some aspects of `EventState` are tied to the scale factor; others relate to
    popups. A modal window should therefore have its own `EventState`.
-   The modal window will need its own surface.

Thus, in many ways a modal window is similar to a primary window.
It is not yet determined whether a modal window should have state embedded
within a parent window via a widget, though likely not.


Widget lifecycle
----------------

The status quo:

-   Are constructed by the user
-   Methods `configure` and then `update` are called initially (before sizing).
    External resources (e.g. images) and transient children (view widgets) are
    created by one of these methods.
-   Method `update` is called again after input data (may) have changed
-   Widgets are sized via `size_rules` and `set_rect` before drawing or event-handling
-   Sizing is repeated as required (potentially `set_rect` only) when the widget
    is resized, moved or the scale factor changes
-   After the above, drawing and event handling may happen at any time

New additions:

-   Method `Events::suspend` is called on all widgets when the window is
    suspended. This method may (but is not required to) free resources and
    destroy transient children to save memory. Suspended widgets may not be
    sized, drawn, or have their event handlers invoked. Potentially (but not
    necessarily) popups could be suspended when closed and `Stack` pages when
    hidden; if so additional hints regarding resource usage may be desired.
-   A widget may be resumed by configuring (`configure`, `update`) and sizing
    (at least `set_rect`). After this, drawing and event-handling may happen.
