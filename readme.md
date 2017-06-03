## Interoperable CSSinJS format.

The biggest issue at scale we currently have with all CSSinJS implementations is that they are not sharable without the runtime.

We can define a format that is easy to parse and supports all the needs of the current CSSinJS implementations on top of CSS.

Package maintainers will be able to publish JavaScript files to NPM with this format inside instead of plain CSS.

## First draft

Format is optimized for parsing in JavaScript, not for human readability.

### Markers

Examples are using constant names instead of values for readability within the spec. The resulting format will use their values.

```js
const RULE_START = 0
const RULE_END = 1
const RULE_NAME = 2
const SELECTOR = 3
const PARENT_SELECTOR = 4
const SPACE_COMBINATOR = 5
const DOUBLED_CHILD_COMBINATOR = 6
const CHILD_COMBINATOR = 7
const NEXT_SIBLING_COMBINATOR = 8
const SUBSEQUENT_SIBLING_COMBINATOR = 9
const PROPERTY = 10
const VALUE = 11
const CONDITION = 12
const FUNCTION = 13
```

#### Rule start

Marker `RULE_START` specifies the beginning of a rule, has a rule type.

`[RULE_START, RULE_TYPE]`

All rule types from https://wiki.csswg.org/spec/cssom-constants

```
1 STYLE_RULE CSSOM
2 CHARSET_RULE CSSOM
3 IMPORT_RULE CSSOM
4 MEDIA_RULE CSSOM
5 FONT_FACE_RULE CSSOM
6 PAGE_RULE CSSOM
7 KEYFRAMES_RULE css3-animations
8 KEYFRAME_RULE css3-animations
9 MARGIN_RULE CSSOM
10 NAMESPACE_RULE CSSOM
11 COUNTER_STYLE_RULE css3-lists
12 SUPPORTS_RULE css3-conditional
13 DOCUMENT_RULE css3-conditional
14 FONT_FEATURE_VALUES_RULE css3-fonts
15 VIEWPORT_RULE css-device-adapt
16 REGION_STYLE_RULE proposed for css3-regions
17 CUSTOM_MEDIA_RULE mediaqueries
```

#### Rule end

Marker `RULE_END` specifies the end of a rule.

#### Named rule.

Marker `RULE_NAME` specifies a rule name.

Instead of using a selector, a rule can be identified by a name. This is a preferred way instead of using hardcoded selectors. Consumer library will generate a selector for the rule and user can access it by name.
We rely on the JavaScript module scope. Name should be unique within that module scope.

E.g. `[RULE_NAME, 'bar']`

#### Selector

Marker `SELECTOR` specifies a selector or selector compound.

Selector e.g. `.foo` => `[SELECTOR, '.foo']`

Selector compound e.g. `.foo.bar` => `[SELECTOR, '.foo', '.bar']`

Selector list e.g. `.foo, .bar` => `[SELECTOR, '.foo'], [SELECTOR, '.bar']`

#### Parent selector

Marker `PARENT_SELECTOR` specifies a selector, which is a reference to the parent selector.
Useful for nesting.

E.g.: `&:hover` => `[PARENT_SELECTOR, ':hover']`

#### Combinators

All combinator constants denote any of 5 [CSS combinators](https://drafts.csswg.org/selectors/#combinators).

E.g.: `.foo .bar` => `[SELECTOR, '.foo', SPACE_COMBINATOR, '.bar']`
or `.foo + .bar` => `[SELECTOR, '.foo', NEXT_SIBLING_COMBINATOR, '.bar']`

#### Property name

Marker `PROPERTY` specifies a property name e.g.: `[PROPERTY, 'color']`.

#### Property value

Marker `VALUE` specifies a property value e.g.: `[VALUE, 'red']`.

Multiple comma separated values e.g.: `red, green` => `[VALUE, 'red'], [VALUE, 'green']`.

#### Condition

Marker `CONDITION` specifies a condition for conditional rules.

E.g. `@media all` => `[RULE_START, 4], [CONDITION, 'all']`

#### Function

Marker `FUNCTION` specifies [functional pseudo class](https://drafts.csswg.org/selectors/#functional-pseudo-class) as well as [functional notation](https://drafts.csswg.org/css-values-4/#functional-notation).

After the FUNCTION marker follows the function name and each argument as a separate entry: `[FUNCTION, {NAME}, {ARG0}, {ARG1}, ...]`

Functional pseudo class e.g. `.foo:matches(:hover, :focus)` => `[SELECTOR, '.foo', [FUNCTION, ':matches', ':hover', ':focus']]`

Functional value notation e.g.: `color: rgb(100, 200, 50)` => `[PROPERTY, 'color'], [VALUE, [FUNCTION, 'rgb', 100, 200, 50]]`

## Examples

### Global tag selector

```css
body {
  color: red
}
```

```js
[
  [RULE_START, 1],
    [SELECTOR, 'body'],
    [PROPERTY, 'color'],
    [VALUE, 'red']
  [RULE_END]
]
```

### Selectors list

```css
body, .foo {
  color: red
}
```

```js
[
  [RULE_START, 1],
    [SELECTOR, 'body'],
    [SELECTOR, '.foo'],
    [PROPERTY, 'color'],
    [VALUE, 'red']
  [RULE_END]
]
```

### Multiple values

```css
.foo {
  border-color: red, green
}
```

```js
[
  [RULE_START, 1],
    [SELECTOR, '.foo'],
    [PROPERTY, 'border'],
    [VALUE, 'red'],
    [VALUE, 'green'],
  [RULE_END]
]
```

### Fallbacks

```css
.foo {
  color: red;
  color: linear-gradient(to right, red 0%, green 100%);
}
```

```js
[
  [RULE_START, 1],
    [SELECTOR, '.foo'],
    [PROPERTY, 'color'],
    [VALUE, 'red'],
    [PROPERTY, 'color'],
    [VALUE, 'linear-gradient(to right, red 0%, green 100%)'],
  [RULE_END]
]
```

### Conditionals

```css
@media all {
  .foo {
    color: red;
  }
}
```

```js
[
  [RULE_START, 1],
  [CONDITION, 'all'],
    [RULE_START],
      [RULE_TYPE, 1],
      [SELECTOR, '.foo'],
      [PROPERTY, 'color'],
      [VALUE, 'red'],
    [RULE_END],
  [RULE_END]
]
```

### Nesting

```css
.foo {
  color: red;
  &:hover {
    color: green;
  }
}
```

```js
[
  [RULE_START, 1],
    [SELECTOR, '.foo'],
    [PROPERTY, 'color'],
    [VALUE, 'red'],
    [RULE_START, 1],
      [PARENT_SELECTOR, ':hover'],
      [PROPERTY, 'color'],
      [VALUE, 'green'],
    [RULE_END],
  [RULE_END]
]
```

### Nesting with a compound selector and regular selector

```css
.foo {
  color: red;
  &.bar.baz .bla {
    color: green;
  }
}
```

```js
[
  [RULE_START, 1],
    [SELECTOR, '.foo'],
    [PROPERTY, 'color'],
    [VALUE, 'red'],
    [RULE_START, 1],
      [PARENT_SELECTOR, '.bar', '.baz', SPACE_COMBINATOR, '.bla'],
      [PROPERTY, 'color'],
      [VALUE, 'green'],
    [RULE_END],
  [RULE_END]
]
```

### Scoping or named rules

```css
.foo-123456 {
  color: red
}
```

```js
[
  [RULE_START, 1],
    [RULE_NAME, 'foo'],
    [PROPERTY, 'color'],
    [VALUE, 'red']
  [RULE_END]
]
```

### Functional pseudo class

```css
.foo:matches(:hover, :focus) {
  color: red
}
```

```js
[
  [RULE_START, 1],
    [SELECTOR, '.foo', [
      FUNCTION, ':matches', ':hover', ':focus'
    ]],
    [PROPERTY, 'color'],
    [VALUE, 'red']
  [RULE_END]  
]
```

### Functional value notation

 `color: rgb(100, 200, 50)` => `[PROPERTY, 'color'], [FUNCTION, 'rgb', 100, 200, 50]`
```css
.foo {
  background: url(http://www.example.org/image);
  color: rgb(100, 200, 50 );
  content: counter(list-item) ". ";
  width: calc(50% - 2em);
}
```

```js
[
  [RULE_START, 1],
    [SELECTOR, '.foo'],
    [PROPERTY, 'background'],
    [VALUE, [FUNCTION, 'url', 'http://www.example.org/image']],
    [PROPERTY, 'color'],
    [VALUE, [FUNCTION, 'rgb', 100, 200, 50]],
    [PROPERTY, 'content'],
    [VALUE, [FUNCTION, 'counter', 'list-item'], '". "'],
    [PROPERTY, 'width'],
    [VALUE, [FUNCTION, 'calc', '50% - 2em']]
  [RULE_END]  
]
```
