# ECMAScript Proposal: referential destructuring

**Stage:** 0, looking to advance to 1

**Author:** [Ben Newman](https://github.com/benjamn)

**Reviewers:** TBD

**Specification:** TBD

**AST:** TBD

**Transpiler:** TBD


## Introduction

Currently, when an object is destructured by assignment to a left-hand-side _ObjectBindingPattern_, the bound identifiers capture "snapshots" of the state of the object at the moment of destructuring:

```js
let obj = { a: 1, b: 2 };
let { a, b: c } = obj;
console.log(c); // 2
obj.b += 10; // no effect on c
console.log(c); // still 2, not 12
c += 1; // no effect on obj.b
```

This value-shapshotting behavior often matches the desires of the programmer, but not always. Sometimes, one would like for a bound identifier to remain a shorthand for the current value of a property in the destructured object, rather than a copy of some previous value.

In terms of the example above, this proposal introduces new syntax that would allow `c` to continue to refer to the `.b` property of the object that was originally destructured.

Among its other benefits, this syntax should improve the usability of [dynamic `import()`](https://github.com/tc39/proposal-dynamic-import) in the way it handles [_live bindings_](http://2ality.com/2015/07/es6-module-exports.html), making alternate proposals like my [nested `import` declarations](https://github.com/benjamn/reify/blob/master/PROPOSAL.md) proposal unnecessary.

At this stage, we are not committed to any specific syntax. For lack of a better color to paint the shed, I will adopt the ampersand (`&`) reference notation used by other languages (such as C++), because it seems not to collide with existing syntax.


## Examples

In its most basic form, when the `&` token prefixes a key in an object pattern, it indicates that the key should be bound as a reference to that property in the parent object:

```js
let obj = { a: 1, b: 2 };
let { &a, &b: c } = obj;
console.log(a); // 1, same as console.log(obj.a)
console.log(c); // 2, same as console.log(obj.b)
obj.b += 10;
console.log(c); // 12
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

where `_obj$0` is a temporary variable. As this desugaring suggests, if `_obj$0.a` or `_obj$0.b` are implemented by getter properties, the getter function will be called whenever the bound identifier (e.g. `a` or `c`) is evaluated.

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

Note that the `getKey()` expression is *not* reevaluated each time `value` is evaluated, but only once, at the time of destructuring.

The `&` syntax even works well with destructuring patterns in function parameter lists:

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

If this mutability is undesirable, simply use a `const` destructuring declaration instead:

```js
const obj = { x: 1234 };
const { &x: y } = obj;
console.log(y); // 1234
obj.x += 1111; // ok
console.log(y); // 2345
y += 2222; // throws
```

As in other languages with similar syntax, we anticipate that `const` references will come to be regarded as a best practice, just as `const` declarations are preferred wherever possible.

Note also that `y` is essentially an _immutable live binding_, much like a symbol imported by an `import` declaration (though the original value resides in an object, rather than a module environment record). This insight leads us to the most compelling application of this sytax...


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

> Note that this example pretends `await` is allowed at the top level, which is a feature that has never been formally proposed, though this proposal is fully compatible with top-level `await`.

If you are tempted to use a destructuring declaration to bind individual identifiers with the desired names (e.g., `c` instead of `moduleNs.b`), then you lose the live binding behavior:

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

However, with referential destructuring, the addition of two `&`s makes `a` and `c` behave as you would hope:

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
  // Return a closure that always has access to the latest values.
  return () => a + c;
}).then(...);
```

In principle, since every ECMAScript module has a namespace object, it should be possible to desugar any top-level `import` declaration to a combination of dynamic `import()`, top-level `await`, and referential destructuring:

```js
import def, { a, b as c } from "./module";
// ...is roughly equivalent to...
const { &default: def, &a, &b: c } = await import("./module");
```

This desugaring won't work until both this proposal and top-level `await` are implemented, but I think harmonizing language features in this way is a worthwhile long-term goal.


## Relationship to nested `import` declarations

In the July 2016 TC39 meeting, I presented a [proposal](https://github.com/benjamn/reify/blob/master/PROPOSAL.md) to allow nesting `import` declarations inside blocks and functions, with support for live bindings. This proposal technically predated the dynamic `import()` proposal, though there was already talk of a module-scoped replacement for `System.import(id, parent)` at the time of my proposal.

Two concerns were raised during that discussion that led to my withdrawing the nested `import` proposal until I could find satisfactory solutions:

1. It was unclear when (or if) module source code should be obtained, since it's too late to start asynchronous network activity at the moment when the `import` declaration is first evaluated.

2. Synchronous `import` declarations seemed at odds with a future in which module evaluation may be asynchronous, thanks to the possibility of top-level `await`.

Since I first presented that proposal, I've become convinced that the answer to the first objection is that the runtime should make no special effort to fetch the code for modules imported in nested contexts. It's simply the responsibility of the pogrammer to use a bundling tool that ensure the code is synchronously available.

I do not have a perfect solution for the second objection, other than to throw if a nested `import` declaration tries to import an asynchronous module, which should encourage the programmer to hoist the `import` declaration to the top level, or use dynamic `import()`.

While I could argue that these solutions are good enough to justify reviving the nested `import` proposal, the truth is that dynamic `import()` solves both problems already, and has much more momentum as an ECMAScript proposal.

My one remaining regret is the loss of live bindings when using dynamic `import()` and destructuring together. In addition to its other benefits, referential destructuring solves this exact problem, and **I would be happy to withdraw the nested `import` proposal permanently if referential destructuring gains traction with the committee.**


## Relationship to [Babel](http://babeljs.io/)

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
console.log(a, c);
```

and then transpile the referential destructuring syntax using subsequent compiler plugins.

Put another way, if the referential destructuring proposal makes progress, Babel should consider adding support for `&` syntax to [their destructuring transform](https://www.npmjs.com/package/babel-plugin-transform-es2015-destructuring), which would allow the CommonJS modules transform to be significantly simplified.


## Relationship to [Reify](https://github.com/benjamn/reify)

I maintain another compiler for ECMAScript module syntax, called Reify, which [Meteor](https://github.com/meteor/meteor/) uses via [this Babel plugin](https://www.npmjs.com/package/babel-plugin-transform-es2015-modules-reify). Disclosure: I'm also the lead maintainer of Meteor.

One of Reify's claimed benefits is that simulates live bindings better than the default Babel transform:

```js
import a, { b, c as d } from "./module";
```

becomes

```js
// Local symbols are declared as ordinary variables.
let a, b, d;
module.watch(require("./module"), {
  // The keys of this object literal are the names of exported symbols.
  // The values are setter functions that take new values and update the
  // local variables.
  default(value) { a = value; },
  b(value) { b = value; },
  c(value) { d = value; },
});
```

Whenever `./module` exports a new value for `default`, `b`, or `c`, the callback function associated with that export will be called to update the local variable. No references need to be rewritten, since the simulated live bindings are just local variables, and those local variables are easier to inspect in a debugger.

However, if referential destructuring was available, the generated code could be vastly simpler:

```js
const { &default: a, &b, &c: d } = module.watch(require("./module"));
```

This is beginning to look a lot like the code that Babel would generate. So much so, I'm not sure the Reify compiler would continue to exist as an alternative tool. And that's a good thing.


## Non-goals of this proposal

### No references to identifiers

In contrast to languages like C++, this proposal does not allow for references from one identifier to another:

```js
const &ref = someOtherVariable; // not allowed
```

Instead, just refer to `someOtherVariable`.


### No reference parameters

In languages like C++ and Scala, it's possible for a function to mutate the caller's copy of a variable passed as an argument to the function. This can be surprising if you aren't paying close attention to the signature and implementation of the function.

The following code is not supported by this proposal:

```js
function illegal(a, &b) {
  b = a + 1;
}

let a = 1, b = 2;
illegal(a, b);
```

The `&` token is allowed in parameter lists only inside object destructuring patterns, which means it can only be used to modify the contents of caller-provided objects that were already mutable.


### Not another `with` statement

Unlike the widely-deprecated `with` statement, this proposal requires explicitly mentioning each reference property that you want to declare, and does not insert any new records into the scope chain.

The presence of a `with` statement made it impossible to know, statically, whether a free variable inside the `with` block referred to a property of the `with` object or a variable from another enclosing scope.

That was a disaster for static analysis and engine performance, but this proposal is no more problematic than the desugared code we've seen above.


## Open questions

Although I hope the foregoing examples justify exploring this proposal further, there are several details that need to be worked out in order to make the proposal fully concrete.

### Is there a better term than "referential destructuring"?

JavaScript, like Java and many other languages, already has a concept of a "reference," meaning a variable that refers to a heap-allocated object. Heap-allocated objects are always passed by reference in JavaScript, rather than by value, and there are no "pointers" in the language that must be explicitly "dereferenced."

The syntax this proposal introduces doesn't really fit this traditional definition of a "reference," though it is inspired by similar syntax in languages (like C++) that distinguish between pointers and references.

Is there a better term for what this proposal introduces?

### Deeper nesting?


### Array patterns?


### Assignment patterns?


### Interaction with TypeScript and/or Flow syntax?
