Access keys
===========

-   Pull request: [#1](https://github.com/kas-gui/design/pull/1)

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
-   Map upper/lower-case: e.g. `&Load` and `Re&load` both bind to `L`.
-   Be driven by localisation (translations), not hard-coded names/keys.
-   Display an underline on configured letters in mnemonics.
-   Ideally, only usable mnemonics should be underlined: e.g. if two menu items
    bind `B` only the usable one should be underlined.
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
even be considered outdated.

An alternative approach to keyboard-assisted menu navigation is search: e.g. a
keyboard shortcut to open a list of commands which may be filtered by typing,
or the approach used by Android's settings (a choice of navigable menus or using
a search box).


Existing design
---------------

The current design used in Kas is as follows:

-   Mnemonics are assigned to "layers". Typically one base layer is used for the
    main app and a new one for each menu. One layer is active, but mnemonics
    from lower layers may be used (with the side-effect of closing any popup
    associated with the higher layers).
-   The `AccessString` type parses mnemonic keys during construction and
    assignment.
-   Widgets register valid mnemonics at configuration time.
-   Pressing <kbd>Alt</kbd> triggers a redraw; the `AccessLabel` widget draws
    underlines only when `Alt` is pressed.
-   `fn start_key_event` handles matching mnemonics from layers.
-   A widget with an active mnemonic is considered "depressed", allowing e.g.
    calculator buttons to move visually with the corresponding keyboard keys.

This is a piecemeal design which is functional with limitations:

-   The visually-depressed state is currently broken for button-label mnemonics
    since the *label* is considered depressed, not the button.
-   Widgets must register themselves during configuration. If used through a
    view-list it might result in a very large number of registrations. Further,
    all children of stacks, lists etc. are considered part of the same "layer",
    potentially resulting in many conflicts and hidden widgets being activated.
-   Conflicting mnemonics still get a visible underline.
-   Mnemonics are not tied to visibility excepting where a layer is used
    explicitly (mostly for menus). This can have surprising results, especially
    when mnemonics conflict.


Improved designs
----------------

Objectives are to fix the limitations above.

### Pass physical key identifier

This is a very minor modification of the existing design. `AccessLabel` pushes a
`kas::message::Activate` when a mnemonic is used; the key identifier (e.g.
scancode) may be attached as a payload thus allowing the button consuming this
message to register itself as depressed until this key is released.

This fixes only the visually-depressed state issue.

### Event broadcast

A larger modification is to remove configuration "layers" and registration of
mnemonics entirely. Instead, an event notifying of the mnemonic key is broadcast
to all visible widgets on press; the broadcast may be terminated once any widget
consumes the event.

This requires two new capabilities: event broadcast, and obtaining a list/range
of visible children (already implemented).

This fixes two limitations: the need to register mnemonics and tying mnemonics
to visibility.

Caveat: parent menu mnemonic keys will be active and take priority over submenus
unless additionally broadcast is limited to the last popup (possibly closing
that and repeating on no match).

### Mnemonic activation registration

An alternative to removing registration entirely is to perform this temporarily
when <kbd>Alt</kbd> is pressed and when state changes until <kbd>Alt</kbd> is
released.

Registration could happen from `Events::update`, though this is insufficient if
e.g. `Stack` uses sub-tree update when changing page (full-tree update would be
needed to remove mnemonics from the old page).

This allows fixing the final limitation regarding underline of conflicting
mnemonics by providing feedback during this registration action, but only if
registrations are added on a first-come-first-served basis (may not be ideal if
parent menus may have active mnemonics when a sub-menu is opened).

Although it uses registration, it does not suffer from the issues of the
existing design since the registration list is limited to "visible" widgets and
is not long lived. The catch is that the definition of "visible" used by
`Events::update` does not correspond exactly with what is visible on the screen
(though it may be a good thing if scrolling does not hide mnemonics).

### Mnemonic pages

Similar to the prior design, register all mnemonics to pages, but unlike the
prior design the "popup" should be the root of the page (not its parent).
This does not fix any of the above issues.

Use of additional sub-pages (e.g. for `Stack` pages) partially addresses the
visibility issue, but requires some new mechanism to track active `Stack` pages.
