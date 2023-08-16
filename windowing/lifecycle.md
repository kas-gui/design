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


Secondary windows
-----------------

All secondary windows have a parent window. They are positioned relative to a
parent window, but may (depending on platform and implementation) exceed the
bounds of the parent window (note: Winit does not yet fully support this).

A secondary window is defined by a widget of the parent window. This widget will
be resumed when the window is opened and suspended when the window is closed.

Multiple forms of secondary widget may be supported:

-   A modal window which blocks input to its parent and may be positioned anywhere
-   A menu surface which is positioned precisely relative to its parent window
-   Emulations of the above which do not create new window surfaces

To be defined: whether secondary windows have their own `EventState`.
Potentially this struct must be split; it appears that at least some parts
should be per secondary window.


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
    sized, drawn, or have their event handlers invoked.
-   A widget may be resumed by configuring (`configure`, `update`) and sizing
    (at least `set_rect`). After this, drawing and event-handling may happen.
