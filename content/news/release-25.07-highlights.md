+++
title = "Release 25.07 Highlights"
date = 2025-07-15T00:01:00Z
type = "post"
description = "Highlights of the 25.07 release."
in_search_index = true
+++

A long-awaited 25.07 release is finally here. This release saw the replacement of a major, core component of Helix and the addition of plenty of flashy features besides. This release saw changes from 194 contributors. A hearty _thank you_ to everyone who made this release possible.

New to Helix? Helix is a modal text editor with built-in support for multiple selections, Language Server Protocol (LSP), tree-sitter, and experimental support for Debug Adapter Protocol (DAP).

Buckle up; these release notes will get a bit technical as we talk about our new bindings to Tree-sitter, `tree-house`. Before we get into the weeds, let's see check out flashier features.

## File explorer

{{ asciinema(id="file-explorer", width="94", height="25") }}

25.07 adds a file explorer under `<space>e`. The file explorer is a _picker_, a [telescope](https://github.com/nvim-telescope/telescope.nvim)-like UI component central to Helix. Like most other pickers you can fuzzy search within the options. Selecting a directory with Enter opens a new file explorer under that directory and selecting a file opens that file. This is useful for examining a directory as a hierarchy. In contrast the usual file explorer (`<space>f`) opens a picker with the contents of a directory recursively. For sprawling projects, the file explorer can be a more precise tool.

## LSP documentColors

One of the flashier features of the Language Server Protocol (LSP) spec is the [Document Color Request](microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_documentColor). This request allows the client (Helix) to ask a language server like `tailwindcss-language-server` or `vscode-css-language-server` what ranges of the document correspond to RGB colors.

In 25.07 Helix now requests document colors from language servers and displays the swatch (a small box) with the color, inline. This is exactly like the LSP inlay hints feature - which shows types - but for colors.

{{ asciinema(id="lsp-document-colors", width="94", height="25") }}

## New command mode features

Command mode (`:`) is used to execute _typable_ commands. `:write` is a typable command, for example, that takes an optional argument. So is `:quit` - which takes no arguments.

The syntax for command mode is nuanced. It should be simple so that common operations like `:write path/to/doc.md` are easy to type. But it also needs tools for escaping spaces like in `:write 'a b.txt'`. And for some commands it's useful to have a custom or extensible syntax, like `:run-shell-command <complex shell-specific command>`.

25.07 includes a complete rewrite of all of the code used to parse and represent arguments and provide completions for the command line. This fixes a number of bugs with parsing and completion, like trying to complete files with spaces in their names, and introduces two new features, _flags_ and _expansions_.

### Flags

Flags work just like flags you'd pass in a shell command. They're meant to cover cases where you want to execute a command with a minor modification in behavior.

So far flags are only used for a small set of commands: the `:write` family of commands (any command starting with `:write`) and `:sort`.

25.07 removes the `:rsort` command and replaces it with `:sort --reverse`, or `:sort -r` for short. And the `:write` commands now all accept a `--no-format` flag. Typically you want to format the current document(s), if they're configured to auto-format, but occasionally it's useful to write the file exactly as-is. These are great use-cases for flags: you shouldn't need extra typable commands just to tweak small details.

Flags are displayed in the infobox for typable commands and the long versions (like `--reverse`) can be auto-completed.

### Expansions

Expansions introduce a special syntax to interpolate values. These mostly follow [Kakoune](https://github.com/mawww/kakoune)'s concept of expansions with some minor tweaks.

Variables based on the current editor state can be written as `%{variable_name}`. `%{buffer_name}` prints the name of the currently-focused document as it appears in the statusline, and `%{cursor_line}` prints the 1-indexed line number of the primary cursor.

Shell commands can be executed with the `%sh{..}` expansion. Together with the variable expansions above and the new, simple `:echo` command that prints to the statusline, a command like this prints the git blame of the current line on the statusline:

```
:echo %sh{git blame -L %{cursor_line},+1 %{buffer_name}}
```

Both variable names and expansion kinds (like `sh` for shell commands or `u` for Unicode) can be auto-completed.

### Extensible parsing

The initial reason to explore this rewrite was to revisit how the command line was parsed. With the changes in 25.07, command mode parses and completes file names better, allows for flags and expansions, and also enables switching to other methods of parsing part-way through the line.

Typable commands `:set-option` and `:toggle-option` now use [`serde_json`'s streaming deserializer](https://docs.rs/serde_json/latest/serde_json/struct.StreamDeserializer.html) to parse complex config values like lists. Shell commands like `:run-shell-command` and `:pipe` no longer try to parse the command line after the command name. So you don't need to guess how to escape Helix's parsing rules and then your shell's parsing rules - the ultimate quote hell.

## Tree-house

In this release cycle we switched out the crates we use to interact with Tree-sitter, adding new crates built from the ground up and removing the official bindings alongside a lot of old code from Helix.

The rest of this post will discuss details about Tree-sitter and Tree-house. Looking for more details about the changes since Helix 25.01.1? Check out the [changelog] for the full set of code changes.

#### Tree-sitter

Not familiar with [Tree-sitter](https://github.com/tree-sitter/tree-sitter)? At a high level, it's a framework for generating and using fast, error-tolerant parsers. In a `grammar.js` file you can write parser rules via the [Grammar DSL](https://tree-sitter.github.io/tree-sitter/creating-parsers/2-the-grammar-dsl.html) and use then use `tree-sitter` CLI tools to generate and test the parser.

Tools like editors can then use the parser you've defined with the Tree-sitter C library, or language specific bindings, to parse and act on syntax trees. What you do with the syntax tree is up to your imagination! Language servers can use Tree-sitter for their parsers, diff tools like [Difftastic](https://github.com/Wilfred/difftastic) can produce syntax-aware diffs, a language server like [Codebook](https://github.com/blopker/codebook) can do a syntax-aware scan for spell checking. Even [GitHub uses tree-sitter](https://docs.github.com/en/repositories/working-with-files/using-files/navigating-code-on-github) for code navigation and highlighting of some languages.

A very powerful tool for working with parsed trees is _Tree-sitter queries_. Queries are a way to pattern-match against subtrees and _capture_ nodes for future use. For an editor you might use a query, commonly called `highlights.scm`, to capture a tree node like a Rust keyword in order to highlight the node's text according to the current theme.

Like syntax trees, the applications for queries are only limited by your imagination. We currently use queries in Helix for highlighting, indentation and textobjects (recognizing functions, parameters, etc.). In the future, code folding, spell checking and code navigation could use tree-sitter queries as well.

#### History in Helix

Helix depended on Tree-sitter for syntax highlighting even before its initial public release via the official Rust bindings to the C library, the [`tree-sitter`](https://crates.io/crates/tree-sitter) crate. The `tree-sitter` crate wraps the C library and is fairly low level. We also need a highlighter and that is provided by a separate crate: `tree-sitter-highlight`.

[`tree-sitter-highlight`](https://crates.io/crates/tree-sitter-highlight) provides a syntax highlighter which takes the queries for a language and a document's text to highlight and can be iterated to produce highlight events. Helix could then consume highlight iterators while rendering the viewable documents. This works out-of-the-box with `tree-sitter-highlight` and for or simple use-cases like highlighting a document once, `tree-sitter-highlight` is all you need.

The problem with `tree-sitter-highlight` is that it doesn't work incrementally. Creating a new highlight iterator means fully re-parsing the document as well as re-analyzing the queries. This is wasteful since Tree-sitter can reuse queries. Plus parsing in Tree-sitter can work incrementally: you can give the old syntax tree to Tree-sitter and it will parse the new version of the document faster.

So Helix's early highlighter was a fork of `tree-sitter-highlight`, inspired by Tree-sitter's use in the [Atom editor](https://github.com/atom/atom), which factored out the parsed tree (a `Syntax` type) and `tree_sitter::Query`s from the highlighter. With this release we're rewritten the highlighter from scratch in the new `tree-house` crate which fixes a number of longstanding bugs and improves incremental parsing.

#### Injections, a tree of trees

`tree-sitter-highlight` supports a very powerful feature called _injections_. Injection queries capture nodes which should 'switch' to another language. For example in Markdown, you can use a code-fence like so:

    ```rust
    println!("Hello, world!")
    ```

And you would expect the contents of that code-fence to be highlighted as Rust code. How this works is that the full text of this Markdown document is parsed using a Markdown Tree-sitter parser. An `injections.scm` query for Markdown tells Helix that the contents of this code-fence should be treated as Rust code instead. Then Helix runs a Rust Tree-sitter parser for that range of the document and creates a syntax tree.

Then when it comes time to highlight this document, the `highlights.scm` query for Markdown defines highlights for the Markdown parts. And when we get to the Rust _layer_ of the document, the Rust `highlights.scm` take over.

While this is a simple example, injections can work robustly even in complicated cases. This release adds support for Markdown injection within Rust's doc comments for example, leading to deeply nested injections like this:

```rust
/// A type that parses **stuff**
///
/// This is a doc comment, so it should have _Markdown_ highlighting
/// on top of the regular comment highlights.
///
/// # Heading 1
///
/// Know what we can do with Markdown? Inject Rust!
///
///     println!("Hello, world!");
pub struct Parser(/* ... */);
```

In a Rust file like this, the _root_ layer covering the full document is Rust, naturally. Then each doc line comment (`///`) has the content past it parsed as a Markdown document, _combined_ - meaning that the ranges are collectively treated as one Markdown layer. And nested within that, the indented block should act like a code-fence which is another Rust layer.

Internally Tree-house represents this _layer_ concept as a tree. The overall `Syntax` type has a root layer for its file type, and children layers under that for all of its injections. And the children layers can inject other layers themselves, and so on. So the layers form a tree. And each layer is parsed so that it has its own syntax tree, making a kind of _tree of trees_.

#### Incremental injections

Injections were previously discussed way back in the [22.03 release notes](./2022-03-28-release-22.03-highlights.md) which added support for _combined_ injections, like those Markdown comments. Later that year, [22.12](content/news/2022-12-06-release-22.12-highlights.md) brought _incremental injections_. That change reduced the unnecessary work done to re-parse and rerun injections queries for documents with many injections. The switch to Tree-house improves upon incremental injections so that injection layers are re-parsed and injection queries are rerun only for layers which actually changed from any set of edits.

For a more intuitive idea of how this works, imagine a large Markdown list. The Markdown Tree-sitter parser is actually split into two: one for block syntax like code fences and another for "inline" syntax like bold, italics and inline code. The Markdown parser injects the "inline Markdown" parser for situations like list items, so a very large list in Markdown means thousands of small injections of the "inline" parser for each list item.

With the switch to Tree-house, editing within one list item in a large list will only cause the re-parsing and rerun of injections queries for root layer and the edited "inline" layer - the minimum amount of work required.

#### Locals

Another useful concept from `tree-sitter-highlight` is _locals_. `locals.scm` is a query used to tag nodes which should have their highlight applied to any later _references_ within the same _scope_.

Imagine a simple Rust function:

```rust
fn add(a: usize, b: usize) -> usize {
    a + b
}
```

The `a` and `b` parameters to this function should be highlighted as parameters - possibly a different highlight depending on the theme. To track this information, the locals query captures nodes for function parameters like `a` and `b` and also the _scope_: the function body in this case. Any _references_ - also captured by the locals query - within the scope should inherit the definition's highlight.

Tree-house takes a different approach to locals which solves a longstanding bug in Helix. Since the highlighter is only run for the small range you can see on your screen, locals disappeared whenever the definition fell out of view.

{{ asciinema(id="viewport-locals-before", width="94", height="25") }}

Notice how the parameters `slice`, `char_idx` and `n` lose their parameter highlight (underlined) when the parameters go out of view.

With Tree-house, the definitions are tracked at parse-time and stored in a tree format, like injections, for fast lookup. So the current view of the code doesn't make a difference. Parameters are highlighted correctly regardless of whether the definition is within view.

{{ asciinema(id="viewport-locals-after", width="94", height="25") }}

Now `slice`, `char_idx` and `n` parameters keep their highlight no matter how far into the function you go.

#### Injections for all

One of the nicer quality-of-life improvements brought by Tree-house is that Tree-house's `Syntax` type - corresponding to a parsed document - has functions for working across injections smoothly. `Syntax` is organized as a tree of trees, so looking up injection layers is done in logarithmic time rather than a full scan of all layers.

Building on this, a `TreeCursor` type mirroring the `TreeCursor` from Tree-sitter C library moves across injection layers with an API nearly identical to Tree-sitter's `TreeCursor` API. A new `QueryIter` type provides the ability to run any query across all injection layers in the document - not just highlights.

Taking advantage of injections for all Tree-sitter based features will lead to a more consistent experience across language boundaries. Comment tokens and textobjects within an HTML `<script>` tag should follow JavaScript rules rather than HTML. Indentation within a Markdown code-fence should follow the language you're writing out, not Markdown. These features are not yet merged or released but eventually all Tree-sitter based Helix features should behave as consistently as highlighting.

## Wrapping up

These has been the highlights from the 25.07 release plus a deeper dive into our Tree-sitter integration. Check out the full [changelog] for the details.

Come chat about usage and development questions in the [Matrix space][matrix]
and follow along with Helix's development in the [GitHub repository][helix-git].

[changelog]: https://github.com/helix-editor/helix/blob/master/CHANGELOG.md#2507-2025-07-TODO
[helix-git]: https://github.com/helix-editor/helix/
[matrix]: https://matrix.to/#/#helix-community:matrix.org
