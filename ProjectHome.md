# [tj-tools](https://github.com/BellerophonMobile/tj-tools) is now being maintained on [GitHub](https://github.com/BellerophonMobile/tj-tools).  Wiki notes are still here for now. #

tj-tools provides several utility structures for C.  They're intended to be lightweight drop-ins for use in other projects, providing commonly used elements such as expandable buffers and simple parsing tokens.

Current structures and features include:

  * [tj\_buffer](tj_buffer.md): A simple expandable, reusable buffer container.
  * [tj\_template](tj_template.md): Basic variable substitution within a (text) buffer.
  * [tj\_heap](tj_heap.md): Macro-defined customizable heap.
  * [tj\_searchpathlist](tj_searchpathlist.md): Simple support for locating a file within a set of locations.
  * [tj\_solibrary](tj_solibrary.md): A simple container and helpers for dynamically loaded (.so) libraries.

The structures are well documented in the header files as well as in the wiki pages.

tj-tools is released under the MIT license.