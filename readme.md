[<img alt="GitHub Workflow" src="https://img.shields.io/github/actions/workflow/status/propensive/cataclysm/main.yml?style=for-the-badge" height="24">](https://github.com/propensive/cataclysm/actions)
[<img src="https://img.shields.io/discord/633198088311537684?color=8899f7&label=DISCORD&style=for-the-badge" height="24">](https://discord.gg/7b6mpF6Qcf)
<img src="/doc/images/github.png" valign="middle">

# Cataclysm

__Cataclysm__ provides a typesafe representation of CSS, including properties, selectors, rules and
stylesheets.

## Features

- typesafe DSL for CSS
- model stylesheets in Scala
- use typed expressions for CSS attribute values
- integration with [Honeycomb](https://github.com/propensive/honeycomb/) and [Scintillate](https://github.com/propensive/scintillate/)
- write CSS selectors as Scala expressions


## Availability

Cataclysm has not yet been published as a binary, though work is ongoing to fix this.

## Getting Started

Cataclysm provides several types for modeling CSS.

### Selectors

CSS selectors are currently unparsed, and represented by the `Selector` type, which is just a
wrapper for a `String`. The easiest way to construct a new `Selector` is with the `sel`
interpolator, for example:
```scala
val selector: Selector = sel".form input[type=radio]"
```

In a later version, these will be parsed at compiletime to check they are well-formed.

CSS selectors are implemented using a DSL that allows values, representing CSS classes, identifiers
HTML elements and pseudo-classes, to be combined with a variety of combinators, much as they are
in a CSS stylesheet.

The syntax is similar to CSS, but must nevertheless be valid Scala. While this compromises
familiarity with CSS syntax, there are many advantages too: the code is parsed and typechecked,
so many errors can be detected earlier. And every value is an expression, which facilitates code
reuse. And since selectors are algebraic expressions, with parentheses available, they can express
some selectors more concisely than is possible in CSS.

The syntax mirrors CSS as much as possible, with a couple of differences. A `|` is used in place
of CSS's `,`, and the use of juxtaposition in CSS, with or without a space, is made explicit with
the operators `>>` and `&` respectively. In a general context, these operators should be intuitive.

Elements are represented by `Tag` types from [Honeycomb](https://github.com/propensive/honeycomb/),
and CSS classes may be specified using an interpolated string with the `cls` prefix, e.g.
`cls"hidden"`. Likewise DOM IDs are interpolated strings with the `id` prefix.

Pseudo-classes are available inside the `cataclysm.pseudo` package, and have the same name as their
CSS equivalents, though dashed syntax is translated to camelcase. For example, `first-child`
becomes `firstChild`.

CSS                   | Cataclysm
----------------------+---------
`.myClass`            | `cls"myClass"`
`#domId`              | `id"domId"`
`table`               | `Table` (`honeycomb.Table`)
`table tr`            | `Table >> Tr`
`tr>td`               | `Tr > Td`
`td, th`              | `Td | Th`
`td.hide`             | `Td&cls"hide"`
`td+th`               | `Td + Th`
`td~th`               | `Td ~ Th`
`tr td, tr th`        | `Tr >> (Td | Th)`
`tr td.hide`          | `Tr >> Td&cls"hide"`
`tr:first-child`      | `Tr&firstChild`
`tr.head:first-child` | `Tr&cls"head"&firstChild`

Note in particular that `Tr >> (Td | Th)` gets rewritten to the equivalent of
`Tr >> Td | Tr >> Th`, as parentheses are not available in CSS.

Of course, identifiers like `cls"hide" and `id"banner"` are just values like any other, so it
would be typical to first define, for example,
```scala
val hide = cls"hide"
val banner = id"banner"
```
and then refer to `hide` and `banner` directly in the selector. These same identifiers can be used as
values for Honeycomb element attributes, for example,
```scala
Div(id = banner)(contents*)
```
or,
```scala
Tr(hclass = hide)(Th(heading))
```

### Specifying attributes

CSS attributes may be added to a `Selector` to create a `Rule` by binding the selector to a
`Style` value (typically created with the `Css` constructor) using the `:=` operator. For example,
```scala
val rule: Rule = Tr >> Th := Css(margin = (1.px, 2.px, 1.px), cursor = Cursor.Pointer)
```

This rule would serialize to the CSS,
```css
tr th { margin: 1px 2px 1px; cursor: pointer; }
```

Each named parameter takes its name from the CSS attribute, where CSS's dashed naming style
(for example, `animation-fill-mode`) should be translated into camelcase (for example,
`animationFillMode`).

Every CSS attribute may be transformed in this way, without exception. Only valid CSS attributes
(after transformation) may be used as named parameters, and any attempt to use a nonexistent CSS
attribute will produce a compile error.

For example,
```scala
val rule: Rule = sel".form input[type=radio]"(fontWeight = 600, textAlign = TextAlign.Center)
```

### Stylesheets

A number of `Rule`s may be combined into a single `Stylesheet`. The `Stylesheet` constructor can
take repeated `Rule` arguments, for example:
```scala
val styles = Stylesheet(
  Form := Css(backgroundColor = colors.White),
  Form&mainForm >> Input := Css(display = Display.InlineBlock),
  rule3
)
```

### Other directives

In addition to `Rule`s, a `Stylesheet` may contain a number of other directives, such as imports and
media queries.

These include:
- `Keyframes`
- `Import`

Support for others, in particular media queries, will be added in due course.

### Attribute values

As much as possible, attribute values are strongly typed, which means arbitrary Scala expressions
may be used for attribute values, provided they conform to a suitable type.

The process of finding the most suitable types for each attribute is an ongoing process, and
different CSS attributes are gradually being migrated from using `String`s to more precise types.

Additionally, arithmetic expressions involving dimensional values will use CSS's `calc` function
where necessary, while any Scala expression is valid for an attribute value, as long as it returns
a value of a suitable type.

#### Dimensional values

Lengths and other values use instances of the `Length` enumeration. This provides representations of
all CSS unit types (`px`, `pt`, `in`, `pc`, `cm`, `mm`, `em`, `ex`, `ch`, `rem`, `vw`, `vh`, `vmin`
and `vmax`). These are made available as extension methods on `Double` values.

#### Colors

Colours are provided by [Iridescence](https://github.com/propensive/iridescence/) which can
represent colors in a variety of different models. Colors defined using color models which are
natively supported by CSS will be embedded using that same model, while other colors will be
converted automatically to SRGB.

#### Enumerated values

Many CSS attributes take one of a selection of enumerated possibilities. These are generally
represented by an `enum` whose name is taken from the capitalized, camelcase name of that attribute,
and whose members' names are similarly transformed.

For example, the CSS value, `color-dodge` for the attribute, `mix-blend-mode` is called,
`MixBlendMode.ColorDodge`.

#### Multi-part values

Some CSS attributes, such as `border`, can accept multiple arguments. While these would be
separated by spaces in CSS, they should be embedded in a tuple using Cataclysm. For example,
```scala
Css(margin = (1.px, 3.em))
```



## Related Projects

The following _Scala One_ libraries are dependencies of _Cataclysm_:

[![Anticipation](https://github.com/propensive/anticipation/raw/main/doc/images/128x128.png)](https://github.com/propensive/anticipation/) &nbsp; [![Gossamer](https://github.com/propensive/gossamer/raw/main/doc/images/128x128.png)](https://github.com/propensive/gossamer/) &nbsp; [![Iridescence](https://github.com/propensive/iridescence/raw/main/doc/images/128x128.png)](https://github.com/propensive/iridescence/) &nbsp;

The following _Scala One_ libraries are dependents of _Cataclysm_:

[![Tarantula](https://github.com/propensive/tarantula/raw/main/doc/images/128x128.png)](https://github.com/propensive/tarantula/) &nbsp;

## Status

Cataclysm is classified as __fledgling__. For reference, Scala One projects are
categorized into one of the following five stability levels:

- _embryonic_: for experimental or demonstrative purposes only, without any guarantees of longevity
- _fledgling_: of proven utility, seeking contributions, but liable to significant redesigns
- _maturescent_: major design decisions broady settled, seeking probatory adoption and refinement
- _dependable_: production-ready, subject to controlled ongoing maintenance and enhancement; tagged as version `1.0` or later
- _adamantine_: proven, reliable and production-ready, with no further breaking changes ever anticipated

Projects at any stability level, even _embryonic_ projects, are still ready to
be used, but caution should be taken if there is a mismatch between the
project's stability level and the importance of your own project.

Cataclysm is designed to be _small_. Its entire source code currently consists
of 665 lines of code.

## Building

Cataclysm can be built on Linux or Mac OS with [Fury](/propensive/fury), however
the approach to building is currently in a state of flux, and is likely to
change.

## Contributing

Contributors to Cataclysm are welcome and encouraged. New contributors may like to look for issues marked
<a href="https://github.com/propensive/cataclysm/labels/beginner">beginner</a>.

We suggest that all contributors read the [Contributing Guide](/contributing.md) to make the process of
contributing to Cataclysm easier.

Please __do not__ contact project maintainers privately with questions unless
there is a good reason to keep them private. While it can be tempting to
repsond to such questions, private answers cannot be shared with a wider
audience, and it can result in duplication of effort.

## Author

Cataclysm was designed and developed by Jon Pretty, and commercial support and training is available from
[Propensive O&Uuml;](https://propensive.com/).



## Name

Cataclysm takes its name from the sweeping inundation (typical of a _waterfall_, or _cascade_) since it represents _Cascading_ Style Sheets.

## License

Cataclysm is copyright &copy; 2021-23 Jon Pretty & Propensive O&Uuml;, and is made available under the
[Apache 2.0 License](/license.md).
