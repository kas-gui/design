Keyboard accessibility
======================

- Pull request: https://github.com/kas-gui/design/pull/4

**Scope:** keyboard navigation, focus, shortcuts

**Status:** partial implementation

Introduction
------------

This is a specific sub-set of accessibility, including:

- <kbd>Tab</bkd>-key navigation
- Focus-specific key handling (e.g. use of arrow keys to move a slider)
- Mnemonics (e.g. pressing <kbd>Alt+F</kbd> to open the File menu)
- Context-sensitive shortcuts (e.g. <kbd>Ctrl+C</kbd> to copy the focussed/selected item)
- Global shortcuts (e.g. <kbd>Ctrl+P</kbd> to print a page)
- Following platform conventions (e.g. <kbd>Ctrl+C</kbd> on Linux/Windows, <kbd>Command+C</kbd> on MacOS)

Excluded:

- Screen reader and use of other external accessibility tools (this is a separate issue)
- Focus indication: this comes under theme design

This document discusses what interaction should be supported and the implementation status, with minimal discussion of how.


Tab-key navigation
------------------

### Requirements

- That there is a *keyboard focus* (implemented: `EventState::nav_focus`)
- That widgets can define whether they are a valid navigation target (implemented: `navigation` method/property)
- Pressing <kbd>Tab</bkd> advances to the next valid widget, and <kbd>Shift+Tab</bkd> does the reverse (implemented)
- That container widgets can specify the navigation order of their children (implemented: `Widget::nav_next` method)
- That special UIs (e.g. a small `calculator` or a spreadsheet) can disable <kbd>Tab</kbd>-key navigation (implemented)
- That when a widget receives navigation focus it is notified (implemented: `Event::NavFocus`) and made visible (implemented: special handling of `Event::NavFocus` to scroll as required; is a bit hacky)

### Large containers

If e.g. a 1000-element list has navigable items, this interacts poorly with <kbd>Tab</kbd>-key navigation. Applicable Kas examples are `data-list`, `data-list-view` and `times-tables`. Some apps (e.g. spreadsheets) will provide their own navigation for such cases, but we should aim to provide a decent experience by default.

The status quo:

- The `List` widget (used in `data-list`) is not intended for very long lists and interacts with <kbd>Tab</kbd>-navigation like other containers
- The `ListView` and `MatrixView` widgets (used in `data-list-view` and `times-tables`) have several special features:
  - <kbd>Tab</kbd>-navigation where the prior focus is not a child jumps to the first mapped item (i.e. the first visible item, or only just outside the visible range).
  - <kbd>Tab</kbd>-navigation where the prior focus is a child scrolls as necessary to focus the next item, until the end of the list (the "obvious" behaviour).
  - The navigation keys <kbd>Home</kbd>, <kbd>End</kbd>, arrow-keys, <kbd>PageUp</kbd>, <kbd>PageDown</kbd> may be used to navigate the list

### Interaction with mouse/touch

Clicking on / touching a widget will assign navigation focus to that widget.


Focus-specific key handling
---------------------------

It is expected that e.g. a slider with navigation focus may be adjusted with the arrow keys and that <kbd>Home</kbd> may be used to navigate a list.

This is implemented, roughly as follows:

- If any widget (usually an `EditField`) has claimed keyboard input focus, key events go there; otherwise...
- A sub-set of keyboard keys map to a special enum, [`Command`](https://docs.rs/kas/latest/kas/event/enum.Command.html)
- A `Command` is sent to the widget with navigation focus (e.g. if <kbd>Home</kbd> is pressed and the focus is a `Slider`, that widget handles the event)
- If unused, the [event handling model](https://docs.rs/kas/latest/kas/event/index.html#event-handling-model) gives each ancestor a chance to use the event during unwinding (so e.g. if <kbd>Home</kbd> is pressed but the focussed widget does not use it, but is part of a `ListView`, then that may navigate to the start of the list)


Mnemonics
---------

See [#8: Access Keys](https://github.com/kas-gui/design/pull/8).


Shortcuts
---------

Context-local shortcuts are implemented by using a platform-specific map to convert the key-sequence to a [`Command`](https://docs.rs/kas/latest/kas/event/enum.Command.html), then sending that `Command` to the keyboard focus (see above).

This system is under-developed, especially regarding custom (app-specific) shortcuts and global shortcuts.


Alternatives
------------

Probably <kbd>Tab</kbd>-navigation on a `ListView` should jump to the first visible item, not the first mapped item (which may be just outside the visible range). This is a small bug/TODO.

### Alternate Tab-navigation style

A short press of <kbd>Tab</kbd> or <kbd>Shift+Tab</kbd> behaves like usual. Holding <kbd>Tab</kbd> instead activates a special "navigation mode":

- While <kbd>Tab</kbd> is held, focus is indicated via an extra box/highlight. Focus does not move (no key repeat).
- <kbd>Tab+Esc</kbd> moves focus to the parent (or none)
- <kbd>Tab</kbd> + arrow keys moves focus in the appropriate direction (requires a new method, `spatial_nav`)
- <kbd>Tab</kbd> + <kbd>Home/kbd> or <kbd>End</kbd> moves focus to the first/last child of its container (possibly using event-handling's unwinding, i.e. the first ancestor to handle `TabHome` moves focus).
- <kbd>Tab</kbd> + <kbd>PageUp/kbd> or <kbd>PageDown</kbd> moves focus within a container

Advantage: if well implemented this might be intuitive and much easier to use, at least in some cases.

Disadvantage: non-standard.

Disadvantage: not trivial to implement (but should be viable).

Disadvantage: it is possible that some widgets would be hard or impossible reach. A few cases might cause problems, e.g. floats (widgets on top of one another).
