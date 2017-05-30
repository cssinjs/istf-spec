## Interoperable CSSinJS format.

The biggest issue at scale we currently have with all CSSinJS implementations is that they are not sharable without the runtime.

We can define a format that is easy to parse and supports all the needs of the current CSSinJS implementations on top of CSS.

Package maintainers will be able to publish JavaScript files to NPM with this format inside instead of plain CSS.

## First draft

Format is optimized for parsing in JavaScript, not for human readability.

### Constants

#### Markers

Examples are using constant names instead of values for readability. The end format will use the values.

```js
const RULE_START = 0
const RULE_TYPE = 1
const RULE_END = 2
const SELECTOR = 3
const SCOPED = 4
const PROPERTY = 5
const VALUE = 6
const REF = 7
const CONDITION = 8
```

#### Rule types

Copied from https://wiki.csswg.org/spec/cssom-constants

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

### Global tag selector

```css
body {
  color: red
}
```

```js
[
  [RULE_START],
    [RULE_TYPE, 1],
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
  [RULE_START],
    [RULE_TYPE, 1],
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
  border: 1px solid red, 1px solid green
}
```

```js
[
  [RULE_START],
    [RULE_TYPE, 1],
    [SELECTOR, '.foo'],
    [PROPERTY, 'border'],
    [VALUE, '1px solid red'],
    [VALUE, '1px solid green'],
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
  [RULE_START],
    [RULE_TYPE, 1],
    [SELECTOR, '.foo'],
    [PROPERTY, 'color'],
    [VALUE, 'red'],
    [PROPERTY, 'color'],
    [VALUE, 'linear-gradient(to right, red 0%, green 100%)'],
  [RULE_END]
]
```

### Media rules

```css
@media all {
  .foo {
    color: red;
  }
}
```

```js
[
  [RULE_START],
  [RULE_TYPE, 4],
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
  [RULE_START],
    [RULE_TYPE, 1],
    [SELECTOR, '.foo'],
    [PROPERTY, 'color'],
    [VALUE, 'red'],
    [RULE_START],
      [SELECTOR, REF, ':hover'],
      [PROPERTY, 'color'],
      [VALUE, 'red'],
    [RULE_END],
  [RULE_END]
]
```

### Scoped class name

```css
.foo-123456 {
  color: red
}
```

```js
[
  [RULE_START],
    [RULE_TYPE, 1],
    [SELECTOR, SCOPED, '.foo'],
    [PROPERTY, 'color'],
    [VALUE, 'red']
  [RULE_END]
]
```
