[![Build Status](https://travis-ci.org/kaelzhang/node-comment-json.svg?branch=master)](https://travis-ci.org/kaelzhang/node-comment-json)
[![Coverage](https://codecov.io/gh/kaelzhang/node-comment-json/branch/master/graph/badge.svg)](https://codecov.io/gh/kaelzhang/node-comment-json)
<!-- optional appveyor tst
[![Windows Build Status](https://ci.appveyor.com/api/projects/status/github/kaelzhang/node-comment-json?branch=master&svg=true)](https://ci.appveyor.com/project/kaelzhang/node-comment-json)
-->
<!-- optional npm version
[![NPM version](https://badge.fury.io/js/comment-json.svg)](http://badge.fury.io/js/comment-json)
-->
<!-- optional npm downloads
[![npm module downloads per month](http://img.shields.io/npm/dm/comment-json.svg)](https://www.npmjs.org/package/comment-json)
-->
<!-- optional dependency status
[![Dependency Status](https://david-dm.org/kaelzhang/node-comment-json.svg)](https://david-dm.org/kaelzhang/node-comment-json)
-->

# comment-json

- [Parse] JSON strings with comments into JavaScript objects and MAINTAIN comments
  - supports comments everywhere, yes, **EVERYWHERE** in a JSON file, eventually 😆
  - fixes the known issue about comments inside arrays.
- [Stringify] the objects into JSON strings with comments if there are

The usage of `comment-json` is exactly the same as the vanilla [`JSON`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON) object.

## Install

```sh
$ npm i comment-json
```

## Usage

package.json:

```js
{
  // package name
  "name": "comment-json"
}
```

```js
const {
  parse,
  stringify
} = require('comment-json')
const fs = require('fs')

const obj = parse(fs.readFileSync('package.json').toString())

console.log(obj.name) // comment-json

stringify(obj, null, 2)
// Will be the same as package.json,
// which will be very useful if we use a json file to store configurations.
```

## parse()

```ts
parse(text, reviver? = null, remove_comments? = false)
  : object | string | number | boolean | null
```

- **text** `string` The string to parse as JSON. See the [JSON](http://json.org/) object for a description of JSON syntax.
- **reviver?** `Function() | null` Default to `null`. It acts the same as the second parameter of [`JSON.parse`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse). If a function, prescribes how the value originally produced by parsing is transformed, before being returned.
- **remove_comments?** `boolean = false` If true, the comments won't be maintained, which is often used when we want to get a clean object.

Returns `object | string | number | boolean | null` corresponding to the given JSON text.

If the `content` is:

```js
/**
 before-all
 */
// before-all
{ // before:foo
  // before:foo
  /* before:foo */
  "foo" /* after-prop:foo */: // after-comma:foo
  1 // after-value:foo
  // after-value:foo
  , // before:bar
  // before:bar
  "bar": [ // before:0
    // before:0
    "baz" // after-value:0
    // after-value:0
    , // before:1
    "quux"
    // after-value:1
  ] // after-value:bar
  // after-value:bar
}
// after-all
```

```js
const parsed = parse(content)
console.log(parsed)
```

And the result will be:

```js
{
  // Comments before the JSON object
  [Symbol.for('before-all')]: [{
    type: 'BlockComment',
    value: '\n before-all\n ',
    inline: false
  }, {
    type: 'LineComment',
    value: ' before-all',
    inline: false
  }],
  ...

  [Symbol.for('after-prop:foo')]: [{
    type: 'BlockComment',
    value: ' after-prop:foo ',
    inline: true
  }],

  // The real value
  foo: 1,
  bar: [
    "baz",
    "quux,

    // The property of the array
    [Symbol.for('after-value:0')]: [{
      type: 'LineComment',
      value: ' after-value:0',
      inline: true
    }, ...],
    ...
  ]
}
```

There are **EIGHT** kinds of symbol properties:

```js
Symbol.for('before-all')

// If all things inside an object or an array are comments
Symbol.for('before')

// comment tokens before
// - a property of an object
// - an item of an array
// and before the previous comma(`,`) or the opening bracket(`{` or `[`)
Symbol.for(`before:${prop}`)

// comment tokens after property key `prop` and before colon(`:`)
Symbol.for(`after-prop:${prop}`)

// comment tokens after the colon(`:`) of property `prop` and before property value
Symbol.for(`after-colon:${prop}`)

// comment tokens after
// - the value of property `prop` inside an object
// - the item of index `prop` inside an array
// and before the next key-value/item delimiter(`,`)
// or the closing bracket(`}` or `]`)
Symbol.for(`after-value:${prop}`)

// if comments after
// - the last key-value:pair of an object
// - the last item of an array
Symbol.for('after')

Symbol.for('after-all')
```

And the value of each symbol property is an **array** of `CommentToken`

```ts
interface CommentToken {
  type: 'BlockComment' | 'LineComment'
  // The content of the comment, including whitespaces and line breaks
  value: string
  // If the start location is the same line as the previous token,
  // then `inline` is `true`
  inline: boolean
}
```

### Parse into an object without comments

```js
console.log(parse(content, null, true))
```

And the result will be:

```js
{
  foo: 1,
  bar: [
    "baz",
    "quux"
  ]
}
```

### Special cases

```js
const parsed = parse(`
// comment
1
`)

console.log(parsed === 1)
// false
```

If we parse a JSON of primative type with `remove_comments:false`, then the return value of `parse()` will be of object type.

The value of `parsed` is equivalent to:

```js
const parsed = new Number(1)

parsed[Symbol.for('before-all')] = [{
  type: 'LineComment',
  value: ' comment',
  inline: false
}]
```

Which is similar for:

- `Boolean` type
- `String` type

For example

```js
const parsed = parse(`
"foo" /* comment */
`)
```

Which is equivalent to

```js
const parsed = new String('foo')

parsed[Symbol.for('after-all')] = [{
  type: 'BlockComment',
  value: ' comment ',
  inline: true
}]
```

But there is one exception:

```js
const parsed = parse(`
// comment
null
`)

console.log(parsed === null) // true
```

## stringify(object, replacer?, space?): string

The arguments are the same as the vanilla [`JSON.stringify`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify).

And it does the similar thing as the vanilla one, but also deal with extra properties and convert them into comments.

```js
console.log(stringify(parsed, null, 2))
// Exactly the same as `content`
```

#### space

If space is not specified, or the space is an empty string, the result of `stringify()` will have no comments.

For the case above:

```js
console.log(stringify(result)) // {"a":1}
console.log(stringify(result, null, 2)) // is the same as `code`
```

## License

[MIT](LICENSE)
