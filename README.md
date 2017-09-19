# ECMAScript Proposal: referential destructuring

**Stage:** 0, looking to advance to 1

**Author:** [Ben Newman](https://github.com/benjamn)

**Reviewers:** TBD

**Specification:** TBD

**AST:** TBD

**Transpiler:** TBD

## Problem statement and rationale

Currently, when an object is destructured by assignment to a left-hand-side _{Object,Array}{Binding,Assignment}Pattern_, the bound identifiers become "snapshots" of the state of the object at the moment of destructuring:

```js
let obj = { a: 1, b: 2 };
let { a, b: c } = obj;
console.log(c); // 2
obj.b += 10; // no effect on c
console.log(c); // still 2, not 12
c += 1; // no effect on obj.b
```

This behavior often matches the desires of the programmer, but not always. Sometimes, one would like for the bound identifier to remain a shorthand for the current value of the property in the destructured object, rather than a copy of some previous value.

This proposal introduces new syntax that would allow `c` to continue to refer to the `.b` property of the object that was originally destructured.

At this stage, we are not committed to any specific syntax. For lack of a better color to paint the shed, I will adopt the ampersand (`&`) reference notation used by other languages (such as C++), because it seems not to collide with existing syntax.

## Examples

In its most basic form, the `&` token may prefix keys in object patterns, indicating that the key should be bound as a reference to the given property in the parent object:

```js
let { &a, &b: c } = obj;
console.log(a); // same as console.log(obj.a)
console.log(c); // same as console.log(obj.b)
```

Although the parent object in this case happens to be called `obj`, of course the object need not have a name:

```js
let { &a, &b: c } = getObject();
console.log(a);
console.log(c);
```

Naturally, `a` and `c` should be interpreted as references to properties of the unnamed object returned by the `getObject()` call expression. In other words, a reasonable transpilation of this example might look like

```js
const _obj$0 = getObject();
console.log(_obj$0.a);
console.log(_obj$0.b);
```

The `&` syntax also works well with computed properties:

```js
let { &[getKey()]: value } = getObject();
console.log(value);
```

This example might be transpiled to

```js
const _obj$0 = getObject(), _key$0 = getKey();
console.log(_obj$0[_key$0]);
```

The `&` syntax works well with destructuring patterns in function parameter lists:

```js
function f({ &x: y, setX }) {
  setX(y + 1);
  return y;
}

const obj = {
  x: 1,
  setX(newX) {
    obj.x = newX;
  }
};

console.log(f(obj)); // 2
```

With a `let` (or `var`) destructuring declaration, the bound identifiers can be reassigned, and the original parent object will be updated:

```js
const obj = { x: 1234 };
let { &x: y } = obj;
y += 1111;
console.log(obj.x); // 2345
```

If this behavior is undesirable, simply use a `const` destructuring declaration instead:

```js
const obj = { x: 1234 };
const { &x: y } = obj;
y += 1111; // throws
```

As in other languages that support references, `const` references are likely to be regarded as a best practice, just as `const` declarations are preferred wherever possible.

Note also that `y` is essentially an _immutable live binding_, much like a symbol imported by an `import` declaration. This insight leads us to the most compelling application of this sytax...

## Cooperation with dynamic `import()`

As currently proposed and implemented, the [dynamic `import()`](https://github.com/tc39/proposal-dynamic-import) syntax has one lingering drawback compared with top-level `import` declarations: the only way to achieve live bindings is to retain a reference to the module's namespace object, so that you can access its latest properties.

In other words, the closest dynamic equivalent to this `import` declaration

```js
import { a, b as c } from "./module";

function getSum() {
  return a + c;
}
```

would be something like

```js
const moduleNs = await import("./module");

function getSum() {
  return moduleNs.a + moduleNs.b;
}
```

Note that this example pretends `await` is allowed at the top level, which is a feature that has never been formally proposed, though this proposal is entirely compatible with top-level `await`.

If you are tempted to use a destructuring declaration to bind individual identifiers with the desired names (`c` instead of `moduleNs.b`), then you lose the live binding behavior:

```js
const { a, b: c } = await import("./module");

// Every time this function is called, it returns the sum of the values
// of `a` and `b` exported by `./module` that were available at the time
// the `await import(...)` expression was evaluated, even if `./module`
// has exported new values since then.
function getSum() {
  return a + c;
}
```

If `./module` exports new values for `moduleNs.a` and `moduleNs.b`, those new values won't be visible to the `getSum` function, since it only has access to local variables that contain the original values.

However, with referential destructuring, a simple change makes `a` and `c` behave as you would hope:

```js
const { &a, &b: c } = await import("./module");

// Every time this function is called, it returns the sum of the *latest* values
// of `a` and `b` exported by `./module`.
function getSum() {
  return a + c;
}
```

Not only does `&` preserve live bindings, but the destructuring syntax is also more statically analyzable than the `moduleNs` object, since it's more obvious which properties are being used and which are not, and a reference to the namespace object can never leak into hard-to-analyze code.

This static analysis is the basis of important optimizations like "tree shaking," which (without this proposal) become significantly more difficult when dynamic `import()` is used.

The static analysis argument applies even if you choose to use the `Promise` API, thanks to function parameter destructuring:

```js
import("./module").then(({ &a, &b: c }) => {
  return () => a + c;
});
```

In fact, since every ECMAScript module has a namespace object, it should be possible to desugar any top-level `import` declaration to a combination of dynamic `import()`, top-level `await`, and referential destructuring:

```js
import def, { a, b as c } from "./module";
// ...is roughly equivalent to...
const { &default: def, &a, &b: c } = await import("./module");
```

If you enjoy explaining ECMAScript features in terms of other ECMAScript features as much as I do, then you'll understand why I find this analogy compelling.

## Relationship to nested `import` declarations

In the July 2016 TC39 meeting, I presented a [proposal](https://github.com/benjamn/reify/blob/master/PROPOSAL.md) to allow nesting `import` declarations inside blocks and functions, with support for live bindings. This proposal technically predated the dynamic `import()` proposal, though there was already talk of a module-scoped replacement for `System.import(id, parent)` at the time of my proposal.

Two concerns were raised during that discussion that led to my withdrawing the nested `import` proposal until I could find satisfactory solutions:

1. It was unclear when (or if) module source code should be obtained, since it's too late to start asynchronous network activity at the moment when the `import` declaration is first evaluated.

2. Synchronous `import` declarations seemed at odds with a future in which module evaluation may be asynchronous, thanks to the possibility of top-level `await`.

Since I first presented that proposal, I've become convinced that the answer to the first objection is that the runtime should make no special effort to fetch the code for modules imported in nested contexts. It's simply the responsibility of the pogrammer to use a bundling tool that ensure the code is synchronously available.

I do not have a perfect solution for the second objection, other than to throw if a nested `import` declaration tries to import an asynchronous module, which should encourage the programmer to hoist the `import` declaration to the top level, or use dynamic `import()`.

While I could argue that these solutions are good enough to justify reviving the nested `import` proposal, the truth is that dynamic `import()` solves both problems already, and has much more momentum as an ECMAScript proposal.

My one remaining regret is the loss of live bindings when using dynamic `import()` and destructuring together. In addition to its other benefits, referential destructuring solves this exact problem, and **I would be happy to withdraw the nested `import` proposal permanently if referential destructuring seems promising to the committee.**

## Transpiler appeal

If referential destructuring was already part of ECMAScript when [Babel](http://babeljs.io/) implemented [their ES2016-to-CommonJS modules transform](http://npmjs.org/package/babel-plugin-transform-es2015-modules-commonjs), then the `&` syntax would have made a very convenient compilation target.

Instead of transpiling

```js
import { a, b as c } from "./module"
console.log(a, c);
```

to

```js
var _module = require("./module");
console.log(_module.a, _module.b);
```

Babel could simply generate

```js
const { &a, &b: c } = require("./module");
```

and then transpile the referential destructuring syntax using subsequent compiler plugins.

Put another way, if the referential destructuring proposal makes progress, Babel should consider adding support for `&` syntax to [their destructuring transform](https://www.npmjs.com/package/babel-plugin-transform-es2015-destructuring), which would allow the CommonJS modules transform to be significantly simplified.

## Open questions

Although I hope the foregoing examples justify exploring this proposal further, there are several details that need to be worked out in order to make the proposal fully concrete.

### Deeper nesting?


### Array patterns?


### Assignment patterns?


### Interaction with TypeScript and/or Flow syntax?
