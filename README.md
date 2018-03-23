# proposal-private-static
A TC39 follow-on proposal to extend Private syntax to static fields/methods in classes

## Motivation and History

The main motivation to allowing static private fields/methods inside classes is to allow common refactorings. This means both switching from public to private and extracting common code into static helper methods should be supported with minimal fuss. For a full write up on the use-case, please see [Example of the utility of static private methods](https://github.com/tc39/proposal-static-class-features/issues/4).

But, static private cannot be implemented prefectly, as demonstrated by the [subclass footgun](https://github.com/tc39/proposal-class-fields/issues/43#issuecomment-328681487). Unlike static public, static privates are not inherited through the prototype chain, and resolving the property through object-resolved bindings (eg, `this.#x`) breaks the brand checking. Further, there is no clear way to inherit static privates without [considerable](https://github.com/tc39/proposal-static-class-features/pull/7) [hacks](https://github.com/tc39/proposal-static-class-features/issues/24).

One [suggested path forward](https://github.com/tc39/proposal-class-fields/issues/43#issuecomment-335935224) is to rewrite static private in terms of lexically scoped variables and methods, to which they're analagous. This lead to several proposals, including [`local` lexical bindings](https://github.com/tc39/proposal-static-class-features/issues/4#issuecomment-356744507), [Classes 1.1](https://github.com/zenparsing/js-classes-1.1), and [static initializer blocks](https://github.com/tc39/proposal-static-class-features/issues/23).

However, we never considered just making the static private lexical, and keeping everything else the same. Until now.

## Proposal

Static private methods are now lexically-resolved variables and functions, very similar to the [`local` lexical bindings](https://github.com/tc39/proposal-static-class-features/issues/4#issuecomment-356744507) proposal. But, instead of defining a new syntax, we continue to use the `static` keyword and the `#` private syntax. Further, static private are forbidden (via early syntax errors) from being used in object-resolved bindings, and instance privates are forbidden from using lexical-resolved bindings.

```js
class Example {
  static #staticField = 1;

  static #staticMethod() {}

  #instanceField = 1;

  #instanceMethod() {}

  constructor() {
    // Static fields are lexically-resolved
    #staticField;
    #staticMethod();

    // Instance fields are object-resolved
    this.#instanceField;
    this.#instanceMethod();


    /**
    // Static fields may not be object-resolved
    this.#staticField // Syntax Error!
    this.#staticMethod() // Syntax Error!

    // Instance fields may not be lexically-resolved
    #instanceField // Syntax Error!
    #instanceMethod() // Syntax Error!
    **/
  }
}
```

### Interaction with `this` and `super` in static private methods

Since static private methods are lexically-resolved and invoked, it doesn't make sense to provide a `this` context. You may, however, use `Function.p.call` to provide a `this`.

Additionally, `super` is defined just like any other static method, in terms of the constructor's `__proto__`.

```js
class Base {
  static x = 1;
}

class Example extends Base {
  static #staticMethod() {
    return [this, super.x];
  }

  constructor() {
    #staticMethod(); // => [undefined, 1]
    #staticMethod.call({}); // => [{}, 1]
  }
}
```
