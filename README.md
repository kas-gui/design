Kas Design
==========

This is a repository of Kas design documentation. Like the Rust RFCs repo, its
intended purpose is to document design goals and to allow open discussion.

Current status: I (Diggory Hardy) have many unpublished notes. I plan to publish
at least some of these here to clarify design intention.


Format
------

This repository will contain a series of design documents.
See [the template](template.md).


Process
-------

Unlike Rust RFCs, there is no formal acceptance procedure in place. Documents
may be added without a comment period. In case you have a concern regarding a
document without an open Pull Request or Issue, please open a new issue here.


Organisation
------------

The following categories currently exist for organisation purposes.
Do not create design documents under the project root.

-   [draw](draw): drawing APIs and technologies (e.g. WGPU, OpenGL, Piet)
-   [theme](theme): visual design and theme API
-   [text](text): fonts, shaping, emojis, formatting (note: see also input category)
-   [accessibility](accessibility): translation, accessibility tools, etc.
-   [widget](widget): widget traits, state, construction
-   [input](input): user input and event handling
-   [layout](layout): sizing and positioning of widgets
-   [windowing](windowing): commonly applicable aspects of window construction, placement, focus, scaling
-   [platform](platform): platform-specific stuff (e.g. Android integration)
-   [resouces](resouces): resouce management
-   [other](other): documents which do not fit under other categories
-   [rejected](rejected): proposals which have been considered and later rejected


Licence
-------

This repository is published under the terms of the Apache License, Version 2.0.
You may obtain a copy of this licence from the [LICENSE](LICENSE) file or on
the following webpage: <https://www.apache.org/licenses/LICENSE-2.0>
