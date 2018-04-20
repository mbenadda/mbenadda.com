---
author: mehdi
title: "Typescript's \"strict\" compiler option"
cover: "/images/posts/2018-04-19/cover.jpg"
date: "2018-04-19"
category: default
tags:
    - javascript
    - typescript
    - build
    - "static typing"
---
In TypeScript 2.3, a new compiler option, "strict", was introduced. As its name implies, it provides more stringent compiler checks than the default. It's actually a shortcut for a collection of other options.

*This post is based on a talk [Felix Billon](https://twitter.com/felix_billon) did at the Paris TypeScript Meetup #13. I could not find a blog post equivalent so I made one. Thanks for sharing the knowledge @Felix!*

## How does TypeScript "strict" work?

"strict" is `false` by default, but `true` when you create a new `tsconfig.json` via `tsc --init`. It enables the following compiler options (we will come back to each of these):

- `noImplicitAny`
- `noImplicitThis`
- `alwaysStrict`
- `strictNullChecks`
- `strictFunctionTypes`
- `strictPropertyInitialization`

Individual flags for each of these options are prioritized over "strict". This means you can enable "strict" and individually disable any of its 6 parts to adjust the compiler to your needs.

It is particularly helpful to add TypeScript to an existing project without being overcome by compiler errors. Or to gradually introduce your teammates to the it.

Later on, it will also enable any new *strictness* compiler options that are added to the compiler. Which means you're assured to get them when you upgrade TypeScript.

## What do the "strict" checks entail?

### `noImplicitAny`

`noImplicitAny` raises compiler errors in situations when a value's type has not been declared and cannot be inferred.

This is particularly useful as an undeclared `any` is easy to overlook. It can cause errors downstream in situations when you think the compiler has inferred the types, but it's actually just leaking `any`.

```typescript
// "noImplicityAny": "false"
function getModelName(someUri) {
  return someUri.split('/');
}
getModelName({ model: 'user' }); // Runtime error...
```

Think the compiler guessed `someUri` should be a string? Think again! Now with `noImplicitAny` enabled:

```typescript
// "noImplicitAny": "true"
// [ts] Parameter 'someUri' implicitly has an 'any' type.
function getModelName(someUri) {
  return someUri.split('/');
}
getModelName({ model: 'user' });

```

As it causes all type checks to pass, it is basically like disabling the compiler and reverting to plain Javascript. Not something you want to do unintentionally.

### `strictNullChecks`

`strictNullChecks` raises errors when you attempt to use `null` or `undefined` when a value of another type is expected.

This differs from the default as `string` in the default mode is effectively equivalent to `string | null | undefined` under `strictNullChecks`.

Although more obvious than implicit `any` situations, overlooked null checks are in our experience the most common source of actual bugs when the option is not enabled.

Common cases may include:

```typescript
// Function calls that may or may not return an actual value
// -> [ts] Object is possibly 'null'.
document.querySelector('some-class').innerHTML

// Optional properties of objects taken for granted
type Post = { title?: string };
// ...
const post: Post = {};
// ...
// -> [ts] Object is possibly 'undefined'.
let header = post.title.toUpperCase();

// Set a variable to null when it expects an actual value
let name = 'foo';
// ...
// [ts] Type 'null' is not assignable to type 'string'.
name = null;
```

`strictNullChecks` will result in more ceremonial around accessing values and properties in general. But it will also force you to write more robust code. Missing null checks are effectively unspecified behavior: hence, bugs.

Either you will add checks and specify behavior for null values, or you will architect data flows that branch early and don't propagate unknown quantities.

### `noImplicitThis`

`noImplicitThis` raises errors when `this` is implicitly typed as `any`: when its type has not been declared and cannot be inferred.

One way this can happen is when you use patterns such as mixins and define functions that use `this` in a separate module or a separate scope.

```typescript
function func() {
  // The expectation for func's `this.prop` existence and shape is implicit
  this.prop.doSomething();
}
// ...
const obj = { id: 42, method: func };
// ...
obj.method(); // Runtime error...
```

You may or may not encounter this situation depending on your favored programming paradigm. In any case, in our opinion it's always better to
explicitly define the assumptions we make. You can do is like `this`:

```typescript
function func(this: { prop: { doSomething: () => void } }) {}
```

This option is pretty beginner-friendly as less experienced developers can often struggle or make wrong assumptions about the value of `this`. Here, either the compiler can tell you itself or it will demand you tell it.

### `strictPropertyInitialization`

`strictPropertyInitialization` is only useful if you work with classes. It raises errors when a non-optional property is neither set via a default nor set in the `constructor` method.

This can happen if you expect to set the property in another method that you manually call at some point. It creates uncertainty: it is basically a race condition between your manual call and any access to that property anywhere in the code.

```typescript
class SomeClass {
  prop: string;
  constructor() {}
  onInit() { this.prop = 'some string'; }
}
// ...
const cls = new SomeClass();
// ...
cls.prop.split(' '); // Runtime error...
// ...
// This could easily happen if cls is passed around a lot
cls.onInit();
```

With `strictPropertyInitialization` the compiler will raise `[ts] Property 'prop' has no initializer and is not definitely assigned in the constructor.` at the point where you define the mandatory-but-not-defined property.

```typescript
// You could specify the uncertainty in your definition...
class SomeClass {
  prop?: string;
  constructor() {}
  onInit() { this.prop = 'some string'; }
}

// Set a sensible default value...
class SomeClass {
  prop: string = 'sensible default';
  constructor() {}
  onInit() { this.prop = 'some string'; }
}

// Manage to do the setting in the constructor...
class SomeClass {
  prop: string;
  constructor() { this.prop = 'some string'; }
}

// Or use the definite assertion operator and manage the race condition yourself
class SomeClass {
  prop!: string;
  constructor() {}
  onInit() { this.prop = 'some string'; }
}
```

Again, this option requires you to specify behavior and handle it everywhere, or explicitly use the escape hatch (`!`) if you need to. Either way, you make your intentions clear for the compiler and other developers.

### `strictFunctionTypes`

Like `strictPropertyInitialization`, `strictFunctionTypes` is intended to make working with classes safer. It makes function argument checking covariant instead of bivariant.

For a detailed explanation on covariance & contravariance, take a look at [this wikipedia page](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)). As for us, we'll just illustrate this with an example:

[Example taken (and slightly adapted) from @andy-ms on GitHub.](https://github.com/Microsoft/TypeScript/issues/19661#issuecomment-341213979)

```typescript
class Vehicle {
  turn: (t: this) => void;
}
class Motorcycle extends Vehicle { handlebars = function() {} }
class Car extends Vehicle { wheel = function() {} }

let motorcycle: Motorcycle = new Motorcycle();
let car: Car = new Car();

let vehicle: Vehicle = motorcycle;
motorcycle.turn = (motorcycle_: Motorcycle) => {
  motorcycle_.handlebars();
};
let turn: (vehicle: Vehicle) => void = vehicle.turn;
vehicle = car;
vehicle.turn = turn;

car.turn(car); // Runtime error: car does not have handlebars
```

Here, the compiler with `strictFunctionTypes` would raise an error on `let vehicle: Vehicle = motorcycle;` because `Property 'handlebars' is missing in type 'Vehicle'.`, which shows us pretty clearly where things went wrong.

As you can see, you need to be doing convoluted stuff with classes, which we **do not** recommend as there's pretty much always a better, simpler way in Javascript. Still, it's especially nice to have a little help from the compiler when code gets a little messy.

If you're looking for a deeper explanation for `strictFunctionTypes`, [this PR by @ahejlsberg on GitHub](https://github.com/Microsoft/TypeScript/pull/18654) might be useful to you.

### `alwaysStrict`

`alwaysStrict` simply enables ES5's strict mode, which has been the preferred mode to write Javascript in for a long time. It will also emit `"use strict"` if you don't compile to ES6 with modules (those are always in strict mode).

Check out [the entry on the Mozilla Developer Network](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode) if you want to learn more about Javascript strict mode.

## TL:DR;

Always set `"strict": true` whenever you start a new TypeScript project.

Set `"strict": true` with some flags disabled on an existing project and gradually re-enable everything until you're up-to-date.

The compiler will be much more helpful in avoiding runtime errors and catching some real issues before they get to production.
