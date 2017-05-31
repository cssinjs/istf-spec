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
const SELECTOR = 2
const NAME = 3
const PROPERTY = 4
const VALUE = 5
const PARENT_SELECTOR = 6
const SPACE = 7
const CONDITION = 8
```

#### RULE_START

Denotes the beginning of a rule, has a rule type.

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

#### RULE_END

End of a rule.

#### SELECTOR

Denotes a selector or selector compound.

Selector compound e.g. `.foo.bar` => `[SELECTOR, '.foo', '.bar']`

Multiple classes e.g. `.foo .bar` => `[SELECTOR, '.foo'], [SELECTOR, '.bar']`

#### NAME

Instead of using a selector, a rule can be identified by a name. This is a preferred way instead of using hardcoded global class names. Consumer library will generate a selector for the rule and user can access it by name.
We rely on the JavaScript module scope. Name should be unique within that module scope.

E.g. `[NAME, 'bar']`

#### PROPERTY

Denotes a property name e.g.: `[PROPERTY, 'color']`.

#### VALUE

Denotes a property value e.g.: `[VALUE, 'red']`.

Multiple comma separated values e.g.: `red, green` => `[VALUE, 'red'], [VALUE, 'green']`.

#### PARENT_SELECTOR

Denotes a reference to the selector of the parent rule. Only useful for nesting.

E.g.: `&:hover` => `[SELECTOR, PARENT_SELECTOR, ':hover']`

#### SPACE

Denotes a space.

E.g.: `& .foo` => `[SELECTOR, PARENT_SELECTOR, SPACE, '.foo']`

#### CONDITION

Denotes conditions for conditional rules.

E.g. `@media all` => `[RULE_START, 4], [CONDITION, 'all']`


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

### Multiple classes in one selector

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
      [SELECTOR, PARENT_SELECTOR, ':hover'],
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
      [SELECTOR, PARENT_SELECTOR, '.bar', '.baz', SPACE, '.bla'],
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
    [NAME, 'foo'],
    [PROPERTY, 'color'],
    [VALUE, 'red']
  [RULE_END]
]
```
