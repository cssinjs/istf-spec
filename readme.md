## Interoperable Style Transfer Format [DRAFT].

The biggest issue at scale we currently have with all CSSinJS implementations is that they are not sharable without the runtime.

Using regular CSS doesn't support all features we need. CSS notation is designed for humans and is too slow for parsing.

This format is [blazingly fast](https://esbench.com/bench/592d599e99634800a03483d8) to parse and will support all needs of the current CSSinJS implementations.

Package maintainers will be able to publish JavaScript files to NPM in this format instead of plain CSS or any variation of objects notation.

Do not confuse this format with an AST. It is designed purely for interoperability and publishing. Using this format you can build an AST or regular CSS or any other structure.

### Glossary

The following are some terms that are used throughout the spec, or are relevant when discussing the spec.

- **Markers:** Like nodes in an AST format, these tuples indicate significant literals and values in CSSinJS (See [Markers](#markers))
- **References:** References to primitives in the host language, typically known as "interpolations" in tagged template literals in JavaScript
- **ISTF array:** An array of markers representing CSSinJS code
- **Parsing:** The process of converting CSSinJS to an ISTF array
- **Transformation:** The act of changing an ISTF array
- **User environment / Runtime:** The environment wherein ISTF is turned back into CSS and displayed
- **Preprocessing:** Transforming ISTF arrays to make it distributable and ready for usage in a user environment
- **Postprocessing:** Last transforming steps of ISTF in a user environment, like autoprefixing
- **Evaluation:** Evaluating references to be able to put them into the CSS string during stringification
- **Stringification:** Turning ISTF back into regular CSS, which might involve evaluating references
- **Injection:** — Injecting CSS into the stylesheet

### Markers

Examples are using constant names instead of values for readability within the spec. The resulting format will use their values.

```js
const RULE_START = 0
const RULE_END = 1
const RULE_NAME = 2
const SELECTOR = 3
const PARENT_SELECTOR = 4
const COMPOUND_SELECTOR_START = 5
const COMPOUND_SELECTOR_END = 6
const SPACE_COMBINATOR = 7
const DOUBLED_CHILD_COMBINATOR = 8
const CHILD_COMBINATOR = 9
const NEXT_SIBLING_COMBINATOR = 10
const SUBSEQUENT_SIBLING_COMBINATOR = 11
const PROPERTY = 12
const VALUE = 13
const COMPOUND_VALUE_START = 14
const COMPOUND_VALUE_END = 15
const CONDITION = 16
const FUNCTION_START = 17
const FUNCTION_END = 18
const ANIMATION_NAME = 19
const SELECTOR_REF = 20
const PROPERTY_REF = 21
const VALUE_REF = 22
const PARTIAL_REF = 23
```

#### Rule start

`[RULE_START, <type>]`

Marker `RULE_START` specifies the beginning of a rule, has a rule type.

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

`[RULE_END]`

Marker `RULE_END` specifies the end of a rule.

#### Named rule.

`[RULE_NAME, <name>]`

Marker `RULE_NAME` specifies a rule name.

Instead of using a selector, a rule can be identified by a name. This is a preferred way instead of using hardcoded selectors. Consumer library will generate a selector for the rule and user can access it by name.
We rely on the JavaScript module scope. Name should be unique within that module scope.

#### Selector

`[SELECTOR, <selector>]`

Marker `SELECTOR` specifies a selector.

Selector e.g. `.foo` => `[SELECTOR, '.foo']`

Selector list e.g. `.foo, .bar` => `[SELECTOR, '.foo'], [SELECTOR, '.bar']`

#### Compound selector

`[COMPOUND_SELECTOR_START], [SELECTOR, <selector>]+, [*COMBINATOR]?, [COMPOUND_SELECTOR_END]`

[Compound](https://drafts.csswg.org/selectors/#compound) selector consists of simple selectors which might be separated by [combinators](https://drafts.csswg.org/selectors/#combinators).

Selector compound without combinators e.g. `.foo.bar` =>

```js
[SELECTOR_COMPOUND_START],
  [SELECTOR, '.foo'],
  [SELECTOR, '.bar'],
[SELECTOR_COMPOUND_END]
```

Selector compound with space combinator e.g. `.foo .bar` =>

```js
[SELECTOR_COMPOUND_START],
  [SELECTOR, '.foo'],
  [SPACE_COMBINATOR],
  [SELECTOR, '.bar'],
[SELECTOR_COMPOUND_END]
```

Selector compound with next sibling combinator e.g. `.foo + .bar` =>

```js
[SELECTOR_COMPOUND_START],
  [SELECTOR, '.foo'],
  [NEXT_SIBLING_COMBINATOR],
  [SELECTOR, '.bar'],
[SELECTOR_COMPOUND_END]
```

#### Parent selector

`[PARENT_SELECTOR]`

Marker `PARENT_SELECTOR` specifies a selector, which is a reference to the parent selector.
Useful for nesting.

E.g.: `&:hover` =>

```js
[COMPOUND_SELECTOR_START]
  [PARENT_SELECTOR],
  [SELECTOR, ':hover'],
[COMPOUND_SELECTOR_END]
```

#### Property name

`[PROPERTY, <property>]`

Marker `PROPERTY` specifies a property name e.g.: `color` => `[PROPERTY, 'color']`.

#### Simple value

`[VALUE, <value>]`

Marker `VALUE` specifies a property value e.g.: `red` => `[VALUE, 'red']`.

#### Values list

`[VALUE, 'red']+`

A comma separated list of simple values e.g.: `red, green` => `[VALUE, 'red'], [VALUE, 'green']`.


#### Compound value

`[COMPOUND_VALUE_START], [VALUE, <value>]+, [COMPOUND_VALUE_END]`

A value that consists of multiple space separated simple values: e.g.: `10px 20px` =>

```js
[COMPOUND_VALUE_START],
  [VALUE, '10px'],
  [VALUE, '20px'],
[COMPOUND_VALUE_END]
```
#### Function

`[FUNCTION_START, <name>], [SELECTOR|VALUE, <argument>], [FUNCTION_END]`

Function specifies [functional pseudo class](https://drafts.csswg.org/selectors/#functional-pseudo-class) as well as [functional notation](https://drafts.csswg.org/css-values-4/#functional-notation).

Functional pseudo class e.g. `.foo:matches(:hover, :focus)` =>

```js
[SELECTOR_COMPOUND_START],
  [SELECTOR, '.foo'],
  [FUNCTION_START, ':matches'],
    [SELECTOR, ':hover'],
    [SELECTOR, ':focus'],
  [FUNCTION_END],
[SELECTOR_COMPOUND_END]
```

Functional value notation e.g.: `color: rgb(100, 200, 50)` =>

```js
[PROPERTY, 'color'],
[FUNCTION_START, 'rgb'],
  [VALUE, 100],
  [VALUE, 200],
  [VALUE, 50],
[FUNCTION_END]
```

### Condition

`[CONDITION, <condition>]`

Marker `CONDITION` specifies a condition for conditional rules.

E.g. `@media all` => `[RULE_START, 4], [CONDITION, 'all']`

#### Value reference

`[VALUE_REF, ref]`

Variable `ref` can be ISTF, a string or a function returning any of those. Using ISTF value gives more power to post-processors. Using string value results in better performance.

`border: red, green` =>

```js
// Using ISTF value:
[PROPERTY, 'border'],
[VALUE_REF, () => [
  [VALUE, 'red'],
  [VALUE, 'green'],
]]

// Using a string value:
[VALUE_REF, () => 'red, green']
```

#### Property reference

`[PROPERTY_REF, ref]`

Variable `ref` is a string or a function which returns a string.

`border: red` =>

```js
[PROPERTY_REF, () => 'border'],
[VALUE, 'red']
```

#### Selector reference

`[SELECTOR_REF, ref]`

Variable `ref` is a string or a function which returns a string.

Simple selector: `.foo` => `[SELECTOR_REF, () => '.foo']`
Compound selector: `.foo.bar` =>

```js
[COMPOUND_SELECTOR_START]
  [SELECTOR, '.foo'],
  [SELECTOR_REF, () => '.bar'],
[COMPOUND_SELECTOR_END]
```

#### Partial reference

`[PARTIAL_REF, ref]`

Variable `ref` is an ISTF array or a function which returns an ISTF array.

```css
.foo {
  color: red;
}
.partial {
  color: green;
}
```
```js
[
  [RULE_START, 1],
    [SELECTOR, '.foo'],
    [PROPERTY, 'color'],
    [VALUE, 'red'],
  [RULE_END],
  [PARTIAL_REF, () => [
    [RULE_START, 1],
      [SELECTOR, '.partial'],
      [PROPERTY, 'color'],
      [VALUE, 'green'],
    [RULE_END],    
  ]]
]
```

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
  color: palevioletred;
}
```

```js
[
  [RULE_START, 1],
    [SELECTOR, '.foo'],
    [PROPERTY, 'color'],
    [VALUE, 'red'],
    [PROPERTY, 'color'],
    [VALUE, 'palevioletred'],
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
    [RULE_START, 1],
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
      [COMPOUND_SELECTOR_START],
        [PARENT_SELECTOR],
        [SELECTOR, ':hover'],
      [COMPOUND_SELECTOR_END],
      [PROPERTY, 'color'],
      [VALUE, 'green'],
    [RULE_END],
  [RULE_END]
]
```

### Nesting with a compound selector

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
      [COMPOUND_SELECTOR_START],
        [PARENT_SELECTOR],
        [SELECTOR, '.bar'],
        [SELECTOR, '.baz'],
        [SPACE_COMBINATOR],
        [SELECTOR, '.bla'],
      [COMPOUND_SELECTOR_END],
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
    [COMPOUND_SELECTOR_START],
      [SELECTOR, '.foo'],
      [FUNCTION_START, ':matches'],
        [SELECTOR, ':hover'],
        [SELECTOR, ':focus'],
      [FUNCTION_END],
    [COMPOUND_SELECTOR_END],
    [PROPERTY, 'color'],
    [VALUE, 'red']
  [RULE_END]  
]
```

### Functional value notation

```css
.foo {
  background: url(http://www.example.org/image);
  color: rgb(100, 200, 50);
  content: counter(list-item) ". ";
  width: calc(50% - 2em);
}
```

```js
[
  [RULE_START, 1],
    [SELECTOR, '.foo'],
    [PROPERTY, 'background'],
    [FUNCTION_START, 'url'],
      [VALUE, 'http://www.example.org/image']
    [FUNCTION_END],
    [PROPERTY, 'color'],
    [FUNCTION_START, 'rgb'],
      [VALUE, 100],
      [VALUE, 200],
      [VALUE, 50]
    [FUNCTION_END],    
    [PROPERTY, 'content'],
    [COMPOUND_VALUE_START],
      [FUNCTION_START, 'counter'],
        [VALUE, 'list-item']
      [FUNCTION_END],     
      [VALUE, '". "']
    [COMPOUND_VALUE_END],
    [PROPERTY, 'width'],
    [FUNCTION_START, 'calc'],
      [VALUE, '50% - 2em'],
    [FUNCTION_END],
  [RULE_END]  
]
```

#### Value, Selector and Partial using a function

```css
.foo.bar {
  color: red;
  margin: 10px 20px;
  border: green, red;
}
.partial {
  color: green;
}
```

```js
[
  [RULE_START, 1],
    [COMPOUND_SELECTOR_START],
      [SELECTOR, '.foo'],
      [SELECTOR_REF, () => '.bar'],
    [COMPOUND_SELECTOR_END],
    [PROPERTY, 'color'],
    [VALUE_REF, () => [
      [VALUE, 'red']
    ]],
    [PROPERTY, 'margin'],
    [VALUE_REF, () => [
      [COMPOUND_VALUE_START],
        [VALUE, '10px'],
        [VALUE, '20px'],
      [COMPOUND_VALUE_END]
    ]],
    [PROPERTY, 'border'],
    [VALUE_REF, () => [
      [VALUE, 'green'],
      [VALUE, 'red']
    ]],    
  [RULE_END],
  [PARTIAL_REF, () => [
    [RULE_START, 1],
      [SELECTOR, '.partial']
      [PROPERTY, 'color'],
      [VALUE, 'green'],
    [RULE_END],    
  ]]
]
```

#### Keyframes

```css
@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
}
```

```js
[
  [RULE_START, 7],
    [ANIMATION_NAME, 'fadeIn'],
    [RULE_START, 8],
      [RULE_NAME, 'from'],
      [PROPERTY, 'opacity'],
      [VALUE, 0],
    [RULE_END],
    [RULE_START, 8],
      [RULE_NAME, 'to'],
      [PROPERTY, 'opacity'],
      [VALUE, 1],
    [RULE_END],
  [RULE_END]
]
```

#### Media query

```css
@media all {
  .foo {
    color: red;
  }
}
```

```js
[
  [RULE_START, 4],
    [CONDITION, 'all'],
    [RULE_START, 1],
      [SELECTOR, '.foo'],
      [PROPERTY, 'color'],
      [VALUE, 'red'],
    [RULE_END],
  [RULE_END]
]
```
