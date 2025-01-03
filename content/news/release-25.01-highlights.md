+++
title = "Release 25.01 Highlights"
date = 2025-01-03T00:01:00Z
type = "post"
description = "Highlights of the 25.01 release."
in_search_index = true
+++

The new year brings in a new Helix release! Say hello to 25.01. This is an
especially large release comprising changes from a whopping 171 contributors.
_Thank you_ to everyone who made this release possible!

New to Helix?
Helix is a modal text editor with built-in support for multiple selections,
Language Server Protocol (LSP), tree-sitter, and experimental support for Debug
Adapter Protocol (DAP).

This large release has a bunch of big improvements so let's jump right in to
the highlights.

## Completions

Completions have seen two big updates in 25.01. First, completions now support
paths:

{{ asciinema(id="path-completion", width="94", height="25") }}

Helix now detects a possible path when inserting text and suggests either
absolute paths or relative paths based on the current working directory. This
is especially useful for some languages like Nix where paths are commonplace.

Path completion is especially exciting because it's the first completion
feature not driven by LSP. The change to add path completion refactored parts
of the codebase that assumed that completions could only ever come from a
language server and this should make it easier to add new completion sources
in the future.

The other big change comes to the behavior of snippet completions - currently
only send by language servers - as 25.01 adds support for snippet tabstops.

{{ asciinema(id="snippet-tabstops", width="94", height="25") }}

In the recording above, rust-analyzer suggests a snippet completion for `add`
that includes some placeholders for the function arguments. In the past Helix
would clear these placeholders when accepting the completion with the Enter
key. Now the Enter key jumps the cursor to the first tabstop. The Tab key can
be used to switch to the next tabstop. Typing in any other character replaces
the placeholder text.

## Diagnostics

Historically, LSP diagnostics have appeared right-aligned in the top-right
corner of the editor. While this display is straightforward, it can become hard
to read on small terminal sizes or when a language server sends a long
diagnostic message. 25.01 adds a new way to render diagnostics _inline_.

{{ asciinema(id="inline-diagnostics", width="94", height="25") }}

Inline diagnostics leverage the internal virtual text system to render
diagnostics at the diagnostic's range in the document. Diagnostic messages
appear after the end of the line or in between lines of the appropriate code.

Inline diagnostics are currently disabled by default as we tune the display and
iron out bugs and can be enabled with a config like so:

```toml
# ~/.config/helix/config.toml
[editor]
# Minimum severity to show a diagnostic after the end of a line:
end-of-line-diagnostics = "hint"

[editor.inline-diagnostics]
# Minimum severity to show a diagnostic on the primary cursor's line.
# Note that `cursor-line` diagnostics are hidden in insert mode.
cursor-line = "error"
# Minimum severity to show a diagnostic on other lines:
# other-lines = "error"
```

Setting `end-of-line-diagnostics`, `cursor-line` or `other-lines` to anything
other than `"disable"` enables inline diagnostics.

## Tabular pickers

The picker UI component is central in Helix: it's an efficient and readable way
to jump between files and locations of interest like LSP diagnostics and
symbols. In 25.01 the picker component has been overhauled significantly so
that items are laid out as a table.

{{ asciinema(id="tabular-pickers", width="94", height="25") }}

Simple pickers such as the file picker (`<space>f`) which only need one column
are unchanged. Pickers with multiple fields of information though now show
column names at the top of the the results pane. Rows still correspond to
individual items to pick but there are now columns that horizontally align
the same kinds of content between items. The diagnostics picker for example now
has three columns - "severity", "code" and "message" - while the LSP symbol
picker has two - "kind" and "name".

Columns are individually filterable: one column is filtered by default and
others can be filtered with a query syntax: `%<column name> filter text`. For
example the diagnostics picker can now be filtered down to only errors by
searching for `%severity error`. Filter text is fuzzy matched and column names
may be specified by a prefix, so a search for `%s e` will behave the same in
the diagnostics picker as a search for `%severity error`.

{{ asciinema(id="interactive-global-search", width="94", height="25") }}

Alongside this change, the `global_search` command (`<space>/`) has been
refactored to update with the picker's query dynamically. This allows searching
a codebase for a regex interactively. Filenames can then be filtered separately
with a query like `%path filename`.

## Macro keybindings

Initial support for keybindings written as macros has landed in 25.01. Macro
keybindings can be written by prefixing a string of inputs with `@`.

```toml
# Change the `<space>y` keybinding to yank to the `a` register instead of the
# system clipboard:
[keys.normal]
space.y = "@\"ay"
```

Macro keybindings use the same infrastructure as macros recorded with `q` and
executed with `Q`, so the above behaves as if you typed

1. `"`: open the register select popup
2. `a`: select the 'a' register
3. `y`: copy the selection contents to the register

Macros cover a gap in keybindings: inputs into components like the register
select popup are not exposed as commands, so operations like selecting a
register would not otherwise be possible to bind.

The initial support for macro keybinding limits the bindings to be unusable
in a command sequence: you can't yet combine macros with regular commands. For
example you could swap out the `y` above for its command `yank` in a sequence
like `["@\"a", "yank"]`, but this is not yet supported.

## Commenting

25.01 includes improvements to commenting that make for a smoother experience
when writing line comments like inline documentation.

{{ asciinema(id="25.01-comment-improvements", width="94", height="25") }}

Line comments are now continued by default: pressing the Enter key while in
insert mode or `o`/`O` in normal mode while the cursor is on a line comment now
extends that comment, inserting the current comment token prefix and a trailing
space character. Trailing whitespace is now consistently stripped when pressing
Enter, so `<ret><ret>` can be used to insert a break between two paragraphs in
inline markdown documentation for example.

The `join_selections` and `join_selections_space` commands (`J` and `A-J`) have
also been improved to now strip the comment token when combining line comments.
This makes it easier to edit existing comments or fix spacing between
paragraphs.

## Wrapping up

As always, this is just the highlights from the 25.01 release. Check out the
full [changelog] for the details.

Come chat about usage and development questions in the [Matrix space][matrix]
and follow along with Helix's development in the [GitHub repository][helix-git].

[changelog]: https://github.com/helix-editor/helix/blob/master/CHANGELOG.md#2501-2025-01-03
[helix-git]: https://github.com/helix-editor/helix/
[matrix]: https://matrix.to/#/#helix-community:matrix.org
