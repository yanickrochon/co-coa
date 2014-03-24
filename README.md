# co-efficient

[![NPM](https://nodei.co/npm/co-efficient.png?compact=true)](https://nodei.co/npm/co-efficient/)

An efficient asynchronous template engine for `co`.


## Preamble

In my search for a suitable templating engine, I came across many great projects.
Some promising, some inspiring, and some featuring very good ideas. However, none
of them had all that I was looking for, or some even rejected some of the requirements
that I had. So, instead of taking an existing project and transforming it into an
aberration, I started this one, at first, as a personal exercise. I believe that
`co-efficient` is a potential templating engine, and tests are pretty concluant
about it. The project was born on March 17, 2014, but it is already working quite well!

### TODO

* **Implement modifiers** : Block segments can make use of modifier flags when rendering.
These modifier flags are parsed, but not compiled. This feature should be extendable like
helpers. They should receive a string and should output a string.
* **Data Formatters** : At the moment, there are no custom data formatter support. This is
planned for later versions! And it is a *must have*. Data formatters should be registered
as the key being the data `typeof` value, and it's value should be an async function receiving
a single argument; the data, and return a string. Data formatters should always return a string!
* **More testing!** : There is a 96% branch coverage, however it is not as solid as I'd like
it to be. For example, there *may* be use cases that are not convered in the tests that
could make the parser, or the compiler fail, or make the renderer engine behave abnormally.
* **Optimizations** : The compiler is trying to optimize the template as best as it can,
and the result is quite good so far. But it *can* be improved! Also, rendering the template
rely a lot on data contexts and some shortcuts can be made to improve performance there also.
* **Features** : Even though this project is meant to be lightweight and extendable, some
features may still be missing. Since this project belongs is open source, new features will
come as needed from the user base. For example, a rendering/streaming timeout could be useful.
Or perhaps built-in support for rendering any given templates as string instead of files.


## Features

* Compiled templates into callable JavaScript functions
* COmpiled templates are cached, much like `require`'s behaviour
* 100% Asynchronous using `co` and `--harmony-generators`
* Templates are streamable using standard `stream.Writable` interfaces
* Extendable through [`helpers`](#helpers), and [custom blocks](#custom-blocks).
* Template includes through [partials](#partials)
* Most blocks support context switching for increased template reusability
* Shared reusable blocks declarations across
* Intuitive template language (see [Syntax](#syntax).)
* Well separated clean code


## Installation

`npm install co-efficient --save`


## Configuration

* **paths**:*{String|Array}* - a list of paths where sources may be found. Each
source template will be fetched from first to last, until a file exists. If a string
is specified, eath path must be separated with a `path.delimiter`. *(default: `['.']`)*
* **ext**:*{String|Array}* - a list of source extensions to be appended at the end
a the template name. Note that the engine will also try to look for an extensionless
file as well, as a last resort, if nothing else is found first. Each extension should
start with the dot. If a string is specified, it must be comme delimited.
*(default `'.coa, .coa.html'`)*


## Public API

### Engine

The engine is the core of this module. It is the module that will actually render
the template. If the tempalte is not compiled, it will automatically invoke the
`Parser` and `Compiler` to perform the task.

A typical use of the template engine is

```javascript
Engine.config.paths = [ 'path/to/view/tempaltes' ];

// render 'path/to/view/tempaltes/foo/bar.coeft.html'
var html = yield Engine.render('foo/bar', { foo: 'bar' });
```


#### Engine API

* **cache**:*{Object}* - the actual template cache. To force template re-loading,
simply remove it from this object.
* **config**:*{Object}* - the configuration object. See [Configuration](#configuration).
* **helpers**:*{Object}* - an object of view helpers. Each value should be a
`GeneratorFunction` accepting a `Context` object, a `body` string and a `params`
object, and should return the string to render. See [Helpers](#helpers) for more
information.
* **resolve** *(name:String)*:*{GeneratorFunction}* - locate a template from the
specified configuration. The function is called asynchronously as it performs disk NIO
while searching for the file. The function accepts a single argument, the template
`name`, and returns the template info or `undefined` if nothing found.
* **render** *(name:String, data:Object)*:*{GeneratorFunction}* - render the given template
name using `data` as context root. The function returns the rendered template as a `String`.
* **stream** *(stream:stream.Writable, name:String, data:Object, autoClose:boolean)*:
*{GeneratorFunction}* :
stream the given template name using the specified stream writer, and `data` as context
root. If `autoClose` is set to true, the stream will be closed automatically once the
template is done processing. Otherwise, it is left opened, and it is the caller's
responsibility to close it. The function does not return anything.

**NOTE**: the public API is frozen and you cannot assigned new objects to the `Engine`
object. (i.e. `Engine.config = { foo: true };` will not do anything.)


### Context

A context is the equivalent of the `this` keyword in many programming languages; it
indicates the current "active" data within the data object passed to the template renderer.
However, the `Context` API is very friendly and allows also the template to walk up and
down the context path.

For example, given the following data structure :

```
{
  "company": {
    "departments": [
      "IT", "Management", "Sales", "Marketing", "Support"
    ],
    "employees": {
      "smithj": { "firstName": "John", "lastName": "Smith", "email": "john.smith@email.com" },
      ...
      }
    }
  }
}
```

and given that we current have a `ctx` instance to the with this structure. The following
are all true assertions.

```javascript
ctx.getContext('.').data;
// -> (the data tree same as reported above)

ctx.getContext('company.departments').data;
// -> [ "IT", "Management", "Sales", "Marketing", "Support" ]

ctx.getContext('company.employees').getContext('..');
// -> pointing at the 'company' node. Same as : ctx.getContext('company');

ctx.push({ foo: 'bar' });
// -> { foo: 'bar' } . Also, doing ctx.push({ foo: 'bar' }).getContext('..');
//    brings back to the previous data, completely discarding the data we just pushed.

ctx.push('foo!!').getContext('.company.employees.smithj.firstName').data;
// -> John
```

**NOTE**: the context `ctx.getContext('..........')` (or however many dots there is) will
*always* point to the root of the data tree. That is, assume that the context is already
pointing at the root, it is not possible to point any lower and the context will simply
return itself.

**NOTE**: the assertion `ctx.getContext('.') === ctx` for all contexts. However, doing
`ctx.getContext('.foo')` will get the context `foo` *from* the parent (sibling to the
current context).

**NOTE**: to get the parent context, simply use `ctx.getContext('..')`.

#### Context API

* **parent**:*{Context}* - the parent context, or null if context is root.
* **data**:*{mixed}* - just about any type of data. If `undefined`, will be set to `null`.
* **templateName**:*{String}* - for debugging purpose only, indicate the template name
whose context is associated with. Note that the value follows the template being rendered,
and rendering a partial will change the context's `templateName` value.
* **hasData**:*{boolean}* - returns true if an only if `data` is not empty.
* **push** *(mixed)*:*{Function}* - push some data into a new context whose parent is the
current context and return it.
* **pop** *()*:*{Function}* - return the parent context. If the context has no parent, return
itself.
* **getContext** *(path)*:*{Function}* - return a context relative to the current context.


### Parser

Compiling a template requires two steps. The parser is the first step. The parser will
simply tokenize the template into segments. These segments will allow the compiler to link
and organise these segment in a more optimal fashion. In the process, the parser will
also validate that each segment is well formatted and contains the necessary information
to be transformed into an executable template.

When providing a file or some text, the parser will tokenize it and return a hierarchical
object containing the template's segment tree structure. Nothing much can be done with
a parsed template, but to analyze and validate it's structure.


#### Parser API

* **parseFile** *(file:String)*:*{GeneratorFunction}* - takes a file name and return it's
tokenized segment structure as an hierarchical object.
* **parseString** *(str:String)*:*{GeneratorFunction}* - takes a string and return it's
tokenized segment structure as an hierarchical object.
* **exceptions.ParseException** *{constructor}* - the exception being thrown when the parser
encounters an error. The instances of this object inherits from `Error`.


### Compiler

Compiling a template requires two steps. The compiler is the second step. It takes a hierarchical
segment tree object and transforms it into an executable JavaScript function. The compiler's
output is the string value of the generated code and needs to be compiled (again) by Node to
be fully callable. See [Engine's source code](#lib/engine.js] on how it is typically done.

The reason the compiler does not return a real function is to allow implementation to save the
compiled template to a file. Calling a template function's `.toString()` method does not return
exactly the same string as the compiler's generated string. Besides, the `COmpiler`'s purpose
is to generate JavaScript, not a callable function. The [Engine](#engine) has this contract.


#### Compiler API

* **compile** *(rootSegment:Object)*:*{GeneratorFunction}* - compiles the hierarchical segment
tree into an executable JavaScript function and return the generated string. The `rootSegment`
should be an object compatible with the returned value of `Parser.parseString` or `Parser.parseFile`.


## Syntax

The syntax is very simplistic and minimalistic. All addons should be made through
helpers. All template control flow sections follow this pattern :

```
{type{name|literal[:context] [args][/]}[modifiers]}[content][{type{/}}]
```

A template is composed of segments: block segments and body segments.

* **Block Segments** are blocks declared using the pattern above. They are parsed and compiled
as template instructions.
* **Body Segments** are string contents and are written as is in the compiled templates.


### Context Output

The most basic thing a template needs is token replacements. These are not actual
block segments, but will simly output any context given, formatted as a string. Using the
`Engine`'s data formatter system.

#### Example

```
{{.}}      -> output the current context
{{foo}}    -> output 'foo' from the current context
{{..}}     -> output the parent context
{{.foo}}   -> output the context 'foo' from the parent context
```

Context output may be used anywhere in body segments.

### Helpers

Helpers are async functions callable from the template. They are only declared from
JavaScript and cannot be declared inline, use [inline blocks](#inline-blocks) instead.
They are composed of an identifier, and may optionally include an optional context,
parameters, and one or more content bodies.

The helper function's third argument, the chunk renderer, is the helper's bodies passed
as a renderer. The content has been parsed, but not rendered, yet, and
it's `render` function should be called to retrieve the content. A helper may also
choose to ignore segment content, too. Also, helpers support many segment content bodies,
which may be rendered however many times and whenever within the helper function. For
example :

```javascript
// render first content body (default if no argument is specified)
yield chunk.render(0);

// render second content body...
yield chunk.render(1);
```

**NOTE**: if a chunk body does not exist, an empty string is rendered.

In templates, helpers are rendered with the `{&{helper/}}` instruction. For example :
`{&{helperName:context.path arg=value/}}`. Where `context.path` and `arg=value` are optional.

**NOTE**: helper functions **must** be `GeneratorFunction` or return a `thunk`!


#### Example

```javascript
{
  "hello": function * (stream, ctx, chunk, params) {
    stream.write('<span');
    for (var attr in params) {
      stream.write(' ' + attr + '="' + params[attr] + '"');
    }
    stream.write('>');
    stream.write('Hello, ');
    yield chunk.render();   // same as : yield chunk.render(0);
    stream.write('!</span>');
  }
}
```

called from the view script

```
<div>{&{hello:user id="user" class="bold"}}{{name}}{&{/}}</div>
```

Would output, for example : `<span id="user" class="bold">Hello, John!</span>`.

**NOTE**: data can be injected directly into the context (`ctx`) before rendering chunks
by modifying it's `data` attribute. For example :
`ctx.data = 'replaced data with a single string!';` or `ctx.data['key'] = 'Some Value';`
(assuming `ctx.data` is an object).

Content bodies are unspecified and may or may not be defined in the template for a given
helper call. To define one or more content bodies within a helper, the instruction must be
closed with the `{&{/}}` instruction, and more bodies be defined using the `{&{~}}`
instruction. The special `~` is like the special `/` for closing the block segment, but it
specifies that the block has more content bodies to be associated with. For example :

```
{&{helper}}Content 1{&{~}}Content 2{&{~}}Content 3{&{/}}
```

```javascript
{
  "hello": function * (stream, ctx, chunk, params) {
    yield chunk.render(0);  // renders : "Content 1" (same as chunk.render())
    yield chunk.render(1);  // renders : "Content 2"
    yield chunk.render(2);  // renders : "Content 3"
    yield chunk.render(3);  // renders : "" (no content body)
  }
}
```

#### Helper Function Arguments

* **stream** *{stream.Writable}* : the template's writable stream. This stream
render the template in the same order data are written to it. Therefore, all
non-generator async functions (node style callbacks) must be converted into a
thunk or a generator function, and `yield` their result before returning from
the helper function. Any data written to the stream after returning from the
helper function may cause an exception or unpredictable rendered template.
* **ctx** *{Context}* : a context is equivalent to the `this` in many programming
languages. It indicate the data that can be accessed within the helper. See [Context](#context).
* **chunk** *{Renderer}* : an object with only two properties; `length`, the number of
available content bodies and `render`, a `GeneratorFunction` receiving an optinal `index`
argument, specifying which content body to render. (Ex: `yield chunk.render(0);`)
* **params** *{Object|null}* : an optional argument passing any helper's argument into the
helper function as an object. See [Block Parameters](#block-parameters).


### Inline Blocks

Inline blocks are reusable chunks of segments. They are part of the template, but are never
rendered until they are used. Inline blocks are declared with the `{#{block/}}` instruction.
It is composed of an identifier, and may also optionally include a context and a content
body. Unlike helpers, inline blocks may not have multiple content bodies, but only one.

Inline blocks are declared globally, therefore they are accessible within all views
and partials. Also, they may be overridden.

An inline block may be rendered using the `{+{block/}}` instruction. Undeclared
blocks will be replaced with a warning message in the template. If an inline block was
declared with as self closing (i.e. `{#{block/}}`, an empty block), it will be rendered as
an empty string.

By default, the current context is transferred to the inline block when rendering.
However, a block may define it's own context using `{#{block:path.to.context/}}`.
Also, when rendering a block, the specified block context may be adjusted, too,
via `{+block:relative.context/}`. **Note**: the relative context is relative to the current
context (at rendering time) and not relative to the context of the defined block! However,
the context inside of the block will have a parent equals to the defined context. For
example:

```
{#{withoutContext}}{{.}}{#{/}}
{#{withContext:foo.bar}}{{.}}{#{/}}

{&{withoutContext/}}
{&{withoutContext:context/}}
{&{withContext/}}
{&{withContext:context/}}
```


#### Example

```
{#{header}}
  <tr><th>Col 1</th><th>Col 2</th><th>Col 3</th></tr>
{#{/}}

<table>
  <thead>{+{header/}}</thead>
  <tbody>
    <tr><td>Val 1</td><td>Val 2</td><td>Val 3</td></tr>
  </tbody>
  <tfoot>{+{header/}}</thead>
</table>
```

### Partials

`{>{"path/to/partial"/}}`, and `{>{"path/to/partial":path.to.context/}}`


### Array iterations

`{@{context.array}}...{@{/}}` with `{{.id}}`, `{{.key}}`, and `{{.value}}`.



### Conditionals

`{?{context.condition}}...{?{~}}...{?{/}}`

**NOTE**: conditional blocks are the only ones that do not change context. This
is so to avoid confusion within the "else" (`{?{~}}`) block, and because they
do not semantically encapsulate a different context and because they don't
requiert modifying the context to push additional information as the iteration
block does.

### Block Parameters

*TODO*


### Block Modifier Flags

*Not implemented.*


### Custom Blocks

**DISCLAIMER** : This section is for advanced tempalte usage!

When [Helpers](#helpers) are just not enough, it is possible to extend the template
compiler to include custom block compilation. Unlike helpers, however, these
extensions are global as they are defined directly with the parser and compiler.

Custom blocks may not override built-in blocks, and must be defined using a single
valid block identifier. Block identifiers are case-sensitive within a template! This
means that one could register `a` and `A` and be two completely different blocks.

#### Custom Blocks, Step 1 : Parser Rules

Compiling a template requires two steps. The parser is the first step. To register
a new block, it must be registered with the parser or the template will generate
an error at parse-time. Parser rules are simple, but must be followed strictly,
unless you know what you're doing! A typical rule looks like this :

```
{
  openingContent: 'inBlock',
  validContent: { 'name': true, 'context': true, 'params': true },
  maxSiblings: Number.MAX_VALUE,
  selfClosing: true,
  closeBlock: true
}
```

* **openingContent** *{String}* : tells in what state the parser should be when
parsing the block's segment identifier. The possible values are `inName`, `inContext`
or `inParam`. Since all blocks follow the same syntax (see [Syntax](#syntax)), once
a block state is `inContext`, it means that the block has no `inName` state.
* **validContent** *{Object}* : enables content states for the given block. There
should be at least one content enabled.
* **maxSiblings** *{Numeric}* : how many content bodies, max, the block can have?
To disable this feature and prevent a template from declaring a content body for
the block, set this value to a `false` (or any falsy value).
* **selfClosing** *{boolean}* : whether or not the block can be self closing (i.e.
`{#{block/}}`) or not. If `closeBlock` is `false`, this value **must** be `true`.
* **closeBlock** *{boolean}* : whether or not the block can have an external closing
block (i.e. `{#{block}}{#{/}}`) or not. If `selfClosing` is `false, this value **must**
be `true`.

```javascript
Parser.registerBlockRule(id, options);
Parser.unregisterBlockRule(id);
```

Where `id` is the block identifier and `options` an object as described above.


#### Custom Blocks, Step 2 : Compiler Renderers

To actually compile the template into an executable one, all blocks are rendered
by a block segment renderer. Each renderer should match a parser rule, or an
`CompilerException` will be thrown at compile-time.

```javascript
Compiler.registerBlockRenderer(id, renderer);
Compiler.unregisterBlockRenderer(id);
```

Where `id` is the block identifier and `renderer` a `thunk` or `GeneratorFunction`.

The renderer function's signateur should be `(compiledData, segValue, segKey, segments, engine)`,
and each argument is defined as :

* **compiledData** *{Object}* : an object of compiled data so far. The object contains different
sections that will be concatenated by the compiler at finishing time.


## Contribution

All contributions welcome! Every PR **must** be accompanied by their associated
unit tests!


## License

The MIT License (MIT)

Copyright (c) 2014 Mind2Soft <yanick.rochon@mind2soft.com>

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so,
subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
