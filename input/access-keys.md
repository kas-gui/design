Access keys
===========

-   Pull request: [#8](https://github.com/kas-gui/design/pull/8)

**Scope:** keyboard shortcuts as shown by underline

**Status:** implemented (simple); possible improvements


Introduction
------------

Several traditional UI systems support mnemonics, also known as "access keys".
Primarily these are used on menus:

-   Hold <kbd>Alt</kbd> to underline "mnemonic" keys in labels
-   Press e.g. <kbd>Alt + F</kbd> to open the `File` menu

Such labels are often configured using simple markup: e.g. `&New` may be the
title of a menu item with a mnemonic bound to <kbd>Alt + N</kbd>.

One common limitation of such systems is that, should mnemonic keys happen to
clash, only one such item is reachable through the mnemonic key though others
may still be underlined.

### Requirements

Mnemonics should:

-   Be configurable via simple markup: `&File`, `&Reload`, `Reload &All` etc.
-   Map upper/lower-case: e.g. `&Load` and `Re&load` both bind to key <kbd>L</kbd>.
-   Be driven by localisation (translations), not hard-coded names/keys.
-   Display an underline on configured letters in mnemonics. Ideally, only
    underline usable mnemonics, not clashing keys or those from a different
    "layer" (e.g. a parent menu).
-   Optionally, we could support multiple mnemonics per label with the toolkit
    automatically selecting one to use, or even automatically selecting a letter
    in labels without a configured mnemonic.

Note: theoretically, labels could support further markup, allowing e.g. bold
items or superscript (though Unicode already has significant support for this).
In this case we would want a (custom) DSL not a general markup language like
Markdown.

### Questions

Concerning internationalisation:

-   Are mnemonics applicable to CJK languages whose alphabets are much larger
    than the number of keys on a keyboard?
-   Are there issues with Arabic where letter form may alter substantially?
-   There likely will be issues where a ligature is used, e.g. `Æ`, `ĳ`.
-   Are there issues with accented keys entered via deadkey? Should, say,
    `&Écran` be activated by <kbd>E</kbd>? French keyboards do typically have an
    <kbd>É</kbd> key, but on a Swiss-German layout this letter is only available
    via a third layer (<kbd>AltGr</kbd>) and on a Romanian layout only via a
    dead-key sequence.

### Alternatives

Some apps such as basic calculators will want to bind keyboard keys without the
need to press <kbd>Alt</kbd> to activate mnemonics. This could use a different
approach entirely, but Kas currently uses mnemonics with a special "alt bypass"
mode.

The toolkit does not have to support mnemonics: not all UIs use them. They could
even be considered outdated. There might be conflicts with external
accessibility tools (unknown).

An alternative approach to keyboard-assisted menu navigation is search: e.g. a
keyboard shortcut to open a list of commands which may be filtered by typing,
or the approach used by Android's settings (a choice of navigable menus or using
a search box).


Existing design
---------------

The current design used in Kas is as follows:

-   Mnemonics are assigned to "layers". One base layer is used for the main app
    while each popup has its own layer, whose keys are only active when that
    popup is open.
-   The `AccessString` type parses mnemonic keys during construction and
    assignment.
-   Widgets register valid mnemonics at configuration time.
-   Pressing <kbd>Alt</kbd> triggers a redraw; the `AccessLabel` widget draws
    underlines only when `Alt` is pressed.
-   `fn start_key_event` handles matching mnemonics from layers.
-   A widget with an active mnemonic is considered "depressed", allowing e.g.
    calculator buttons to move visually with the corresponding keyboard keys.

This is a piecemeal design which is functional with limitations:

-   Widgets must register themselves during configuration. This registration
    system is mostly static and may not fully track the state of the UI.
-   Mnemonics are not tied to visibility with the exception of popups. This
    can result in conflicts and in hidden widgets (e.g. from another page of a
    `TabStack` widget) having active access keys.
-   Mnemonics are underlined in labels even when a key conflict makes them
    unusable and when their configuration layer is not active.


Improved designs
----------------

Objectives are to fix the limitations above.

### Mnemonic pages

Currently only the `Window` and `Popup` widgets have access-key layers.

Additional layers could be used such that, for example, a `Stack` widget
could allocate a new layer for each page, thus avoiding having active access
keys on hidden pages (partially fixing the visibility limitation).

This would require some new mechanism to track active `Stack` pages.

### Event broadcast

A larger modification is to remove configuration "layers" and registration of
mnemonics entirely. Instead, an event notifying of the mnemonic key is broadcast
to all visible widgets on press; the broadcast may be terminated once any widget
consumes the event.

This requires two new capabilities: event broadcast, and obtaining a list/range
of visible children (we could probably use `Events::recurse_range` for the latter).

This fixes two limitations: the need to register mnemonics and tying mnemonics
to visibility.

Caveat: parent menu mnemonic keys will be active and take priority over submenus,
unless additionally broadcast is limited to the last popup (possibly closing
that and repeating on no match).

This is likely the best option, but requires some adjustment to core widget
methods.

### Mnemonic activation registration

An alternative to removing registration entirely is to perform this temporarily
when <kbd>Alt</kbd> is pressed and when state changes until <kbd>Alt</kbd> is
released.

Registration could happen from `Events::update`. Caveat: `update` is supposed
to be very fast, though likely this is a non-issue. Caveat: widgets sometimes
update only part of the tree, e.g. when adding a child to a `List` or when
switching the page of a `Stack`. We would need to either add a mechanism to
*unregister* sub-tree registrations or to disallow partial-tree updates.

This design would allow fixing the final limitation regarding underline of
conflicting mnemonics by providing feedback during this registration action,
but only if registrations are added on a first-come-first-served basis (which,
as with the above design, would require restriction to the top-most popup).

### Shadow tree

We could copy the model used by AccessKit: build a shadow model of the widget
tree (limited to nodes which are visible and have either children or access
keys). This shadow tree would need to update any node whose list of visible
children changes.

The caveat of this approach is that it does a lot of work just to build a
limited copy of the widget tree we already have. The advantage is that it would
work very similarly to AccessKit, and might even be swapped out at run-time
(with the implication that access keys are not available when using an external
accessibility tool).
