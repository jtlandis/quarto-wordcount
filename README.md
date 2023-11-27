
<!-- README.md is generated from README.qmd. Please edit that file -->

# Quarto word count

- <a href="#why-counting-words-is-hard"
  id="toc-why-counting-words-is-hard">Why counting words is hard</a>
- <a href="#using-the-word-count-script"
  id="toc-using-the-word-count-script">Using the word count script</a>
  - <a href="#using-as-an-extension" id="toc-using-as-an-extension">Using
    as an extension</a>
  - <a href="#using-without-an-extension"
    id="toc-using-without-an-extension">Using without an extension</a>
- <a href="#how-this-all-works" id="toc-how-this-all-works">How this all
  works</a>
- <a href="#appendices" id="toc-appendices">Appendices</a>
- <a href="#example" id="toc-example">Example</a>
- <a href="#credits" id="toc-credits">Credits</a>

## Why counting words is hard

In academic writing and publishing, word counts are important, since
many journals specify word limits for submitted articles. Counting how
many words you have in a Quarto Markdown file is tricky, though, for a
bunch of reasons:

1.  **Compatibility with Word**: Academic publishing portals tend to
    care about Microsoft Word-like counts, but lots of R and Python
    functions for counting words in a document treat word boundaries
    differently.

    For instance, Word considers hyphenated words to be one word (e.g.,
    “A super-neat kick-in-the-pants example” is 4 words in Word), while
    `stringi::stri_count_words()` counts them as multiple words (e.g. “A
    super-neat kick-in-the-pants example” is 8 words with {stringi}).
    Making matters worse, {stringi} counts “/” as a word boundary, so
    URLs can severely inflate your actual word count.

2.  **Extra text elements**: Academic writing typically doesn’t count
    the title, abstract, table text, table and figure captions, or
    equations as words in the manuscript.

    In computational documents like Quarto Markdown, these often don’t
    appear until the document is rendered, so simply running a
    word-counting function on a `.qmd` file will count the code
    generating tables and figures, again inflating the word count.

3.  **Citations and bibliography**: Academic writing typically counts
    references as part of the word count (even though IT SHOULDN’T).
    However, in Quarto Markdown (and all other flavors of pandoc-based
    markdown), citations don’t get counted until the bibliography is
    generated, which only happens when the document is rendered.

    Simply running a word-counting function on a `.qmd` file (or
    something like the super neat
    [{wordcountaddin}](https://github.com/benmarwick/wordcountaddin))
    will see citekeys in the document like `@Lovelace1842`, but it will
    only count them as individual words (e.g. not “(Lovelace 1842)” in
    in-text styles or ‘Ada Augusta Lovelace, “Sketch of the Analytical
    Engine…,” *Taylor’s Scientific Memoirs* 3 (1842): 666–731.’ in
    footnote styles), and more importantly, it will not count any of the
    automatically generated references in the final bibliography list.

## Using the word count script

This extension fixes all three of these issues by relying on a [Lua
filter](_extensions/wordcount/wordcount.lua) to count the words after
the document has been rendered and before it has been converted to its
final output format. [Frederik Aust (@crsh)](https://github.com/crsh)
uses the same Lua filter for counting words in R Markdown documents with
the [{rmdfiltr}](https://github.com/crsh/rmdfiltr) package (I actually
just copied and slightly expanded [that package’s
`inst/wordcount.lua`](https://github.com/crsh/rmdfiltr/blob/master/inst/wordcount.lua)).
The filter works really well and [is generally comparable to Word’s word
count](https://cran.r-project.org/web/packages/rmdfiltr/vignettes/wordcount.html).

The word count will appear in the terminal output when rendering the
document. It shows three different values: (1) the total count, (2) the
count for the document sans references, and (3) the count for the
reference list alone.

``` text
133 total words
-----------------------------
76 words in text body
57 words in reference section
```

There are two ways to use the filter: (1) as a formal Quarto format
extension and (2) as a set of pandoc filters. You should definitely
glance through the [“How this all works” section](#how-this-all-works)
to understand… um… how it works.

### Using as an extension

Install the extension in an already-existing project by running this in
your terminal:

``` bash
quarto add andrewheiss/quarto-wordcount
```

Or create a brand new project by running this:

``` bash
quarto use template andrewheiss/quarto-wordcount
```

You can then specify one of three different output formats in your YAML
settings: `wordcount-html`, `wordcount-pdf`, and `wordcount-docx`:

``` yaml
title: Something
format:
  wordcount-html: default
```

The `wordcount-FORMAT` format type is really just a wrapper for each
base format (HTML, PDF, and Word), so all other HTML-, PDF-, and
Word-specific options work like normal:

``` yaml
title: Something
format:
  wordcount-html:
    toc: true
    fig-align: center
    cap-location: margin
```

### Using without an extension

Alternatively, if you don’t want to install the extension, download the
two Lua scripts [`wordcount.lua`](_extensions/wordcount/wordcount.lua)
and [`citeproc.lua`](_extensions/wordcount/citeproc.lua), put them
somewhere in your project, and reference them in the YAML front matter
of your document. Make sure you also disable citeproc so that it doesn’t
run twice.

``` yaml
title: Something
format:
  html:
    citeproc: false
    filters: 
      - at: pre-render
        path: "citeproc.lua"
      - at: pre-render
        path: "wordcount.lua"
```

Or as another option, install the extension like normal, but don’t use
the `wordcount-html` format and instead refer to the Lua filters that
live in the extension folder:

``` yaml
title: Something
format: 
  html:
    citeproc: false
    filters: 
      - at: pre-render
        path: "_extensions/andrewheiss/wordcount/citeproc.lua"
      - at: pre-render
        path: "_extensions/andrewheiss/wordcount/wordcount.lua"
```

## How this all works

Behind the scenes, pandoc typically converts a Markdown document to an
abstract syntax tree (AST), or an output-agnostic representation of all
the document elements. In AST form, it’s easy to use the [Lua
language](https://pandoc.org/lua-filters.html) to extract or exclude
specific elements of the document (i.e. exclude captions or only look at
the references).

Quarto was designed to be language-agnostic, so {rmdfiltr}’s approach of
using R to dynamically set the path to its Lua filters in YAML front
matter does not work with Quarto files. ([See this comment from the
Quarto team stating that you cannot use R output in the Quarto YAML
header](https://github.com/quarto-dev/quarto-cli/issues/1391#issuecomment-1185348644).)

But it’s still possible to use the fancy {rmdfiltr} Lua filter with
Quarto with a little trickery!

In order to include citations in the word count, we have to feed the
word count filter a version of the document that has been processed with
the [`--citeproc`
option](https://pandoc.org/MANUAL.html#citation-rendering) enabled.
However, in both R Markdown/knitr and in Quarto, the `--citeproc` flag
is designed to be the last possible option, resulting in pandoc commands
that look something like this:

``` sh
pandoc whatever.md --output whatever.html --lua-filter wordcount.lua --citeproc
```

The order of these arguments matter, so having
`--lua-filter wordcount.lua` come before `--citeproc` makes it so the
words will be counted before the bibliography is generated, which isn’t
great.

{rmdfiltr} gets around this ordering issue by editing the YAML front
matter to (1) disable citeproc in general and (2) specify the
`--citeproc` flag before running the filter:

``` yaml
output:
  html_document:
    citeproc: false
    pandoc_args:
      - '--citeproc'
      - '--lua-filter'
      - '/path/to/rmdfiltr/wordcount.lua'
```

That generates a pandoc command like this, with `--citeproc` first, so
the generated references get counted:

``` sh
pandoc whatever.md --output whatever.html --citeproc --lua-filter wordcount.lua
```

Quarto doesn’t have a `pandoc_args` option though. Instead, it has a
`filters` YAML key that lets you specify a list of Lua filters to apply
to the document at specific steps in the rendering process:

``` yaml
format:
  html:
    citeproc: false
    filters: 
      - at: pre-render
        path: "/path/to/wordcount.lua"
```

However, there’s no obvious way to reposition the `--citeproc` argument
and it will automatically appear at the end, making it so generated
references aren’t counted.

Fortunately, [this GitHub
comment](https://github.com/quarto-dev/quarto-cli/issues/2294#issuecomment-1238954661)
shows that it’s possible to make a Lua filter that basically behaves
like `--citeproc` by feeding the whole document to
`pandoc.utils.citeproc()`. That means we can create a little Lua script
like `citeproc.lua`:

``` lua
-- Lua filter that behaves like `--citeproc`
function Pandoc (doc)
  return pandoc.utils.citeproc(doc)
end
```

…and then include *that* as a filter:

``` yaml
format:
  html:
    citeproc: false
    filters: 
      - at: pre-render
        path: "/path/to/citeproc.lua"
      - at: pre-render
        path: "/path/to/wordcount.lua"
```

This creates a pandoc command that looks something like this, feeding
the document to the citeproc “filter” first, then feeding that to the
word count script:

``` sh
pandoc whatever.md --output whatever.html  --lua-filter citeproc.lua --lua-filter wordcount.lua
```

Eventually [the Quarto team is planning on allowing filter options to
get injected at different stages in the rendering
process](https://github.com/quarto-dev/quarto-cli/issues/4113), so
someday we can skip the citeproc wrapper filter and just do something
like this:

``` yaml
format:
  html:
    filters:
      post:
        - '/path/to/wordcount.lua'
```

But that doesn’t work yet.

## Appendices

In academic writing, it’s often helpful to have a separate word count
for content in the appendices, since things there don’t typically count
against journal word limits. [Quarto has a neat feature for
automatically creating an appendix
section](https://quarto.org/docs/authoring/appendices.html) and moving
content there automatically as needed. It does this (I think) with a
fancy Lua filter.

However, Quarto’s appendix-generating process comes *after* any custom
Lua filters, so even though the final rendered document creates a div
with the id “appendix”, that div isn’t accessible when counting words
(since it doesn’t exist yet), so there’s no easy way to extract the
appendix words from the rest of the text.

So, as a temporary workaround until I can figure out how to make this
Lua filter run after the creation of the appendix div, you can get a
separate word count for the appendix by creating your own div with the
id `appendix-count`:

``` markdown
# Introduction

Regular text goes here.

::: {#appendix-count}

# Appendix {.appendix}

More words here

:::
```

That will create this word count:

    5 in the main text + references, with 4 in the appendix
    -------------------------------------------------------
    5 words in text body
    0 words in reference section
    4 words in appendix section

## Example

You can see a minimal sample document at [`template.qmd`](template.qmd).

## Credits

The [`wordcount.lua`](_extensions/wordcount/wordcount.lua) filter comes
from [Frederik Aust’s (@crsh)](https://github.com/crsh)
[{rmdfiltr}](https://github.com/crsh/rmdfiltr) package.
