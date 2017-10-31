---
title: Dependency injection in Javascript
date: 2017-10-30 22:03
tags: javascript, dependency injection
image: /images/posts/dependency-injection-in-javascript.png
---

Dependency injection is a software engineering concept/ pattern,
which can have many different implementations,
moreso in Javascript due to the language's inherent flexibility.
This post will first demonstrate a particular implementation
of dependency injection in Javascript with code examples,
and subsequently follow on with a discussion of dependency injection itself,
using these code examples.

## A technique

This dependency injection technique consists of
a **factory function** and an **implementation function** per module.

### Module factory

A factory function creates a new instance of the module,
accepting parameters for inputs used in instantiation.
It is a software engineering pattern in its own right.
When it is used in dependency injection,
the inputs are expected to be dependencies.
An example factory function would look like this:

```javascript
function MyModuleFactory({
    InternalModule,
    ExternalModule,
}) {
    // throw error if any dependency is missing

    const state = {}; // private to MyModule instance

    function init({
        optionA,
        optionB,
    }) {
        // perform configuration tasks using options
    }

    function myMethodA () {
        // do stuff
    }

    return  {
        init,
        myMethodA,
    }; // public in MyModuleInstance
}
```

How this works:

- The factory function accepts an object of dependencies,
  where each key is the name of a dependency that it requires.
  - The use of parameter object destructuring here is intended to
    look akin to named parameters.
- It checks whether any dependencies are missing,
  nad if so, throws an error.
- It returns an object,
  where each key is the name of member functions that should be exposed.
  - An `init` function that accepts an options object is expected, at minimum.
- Any state is encapsulated within the function,
  and is not exposed via the return object, and thus remains private.

Note that the dependencies passed into the factory function can be anything,
and it is up to the implementor of the factory function to define the requisite
behaviour expected of the dependencies injected into it:

- They may be Javascript/ NodeJs globals
  (e.g. `setTimeout`), or
- required built-in or 3rd-party modules
  (e.g. `require('fs')` or `require('express')`), or
- required internal modules conforming to the same dependency injection pattern
  (e.g. `require('./my-other-module.js')`).
- They may also be either used as exported
  (e.g. `require('workshopper-exercise')`), or
- used as invoked/ instantiated
  (e.g. `require('./my-other-module.js').impl({})({})`).

### Module implementation

The factory function is all that is necessary for this
dependency injection technique.
However, this baseline is not very convenient,
as it would require much ceremony
each time one instantiates a module defined in this way.
Thus, for convenience, an **implementation function** is also provided.

An example implementation function would look like this:

```javascript
function MyModuleImplementation(dependencies) {
    return (options = {}) => {
        dependencies.ExternalModule =
            depenencies.ExternalModule || new require('external-module')();
        dependencies.InternalModule =
            dependencies.InternalModule ||
            require('./internal-module.js').implementation(dependencies)({
                // options to pass to internal module
            });
        const myModuleInstance = MyModuleFactory(dependencies);
        myModuleInstance.init(options);
        return myModuleInstance;
    }
}
```

How this works:

- The implementation function takes in a non-optional dependencies object,
  just like the factory function takes in.
- The implementation function returns a function that takes in an optional options object.
- Thus to create an instance of this module, the invocation would be:
  `const myModuleInstance = MyModuleImplementation(dependencies)(options);`
- The implementation function is a wrapper around the factory function,
  with two key differences:
  - If there are any dependencies that are not defined,
    a default implementation is added
  - The instance returned from the factory function automatically has its
    `init()` function called with the options passed in.
- The implementation function thus serves a second purpose, as a "live definition"
  of the requirements of the dependencies which should be passed in the factory function.
- The dependencies may be passed on to any other modules which may need them,
  as seen above in the default implementation for `InternalModule`.
  - They may be passed as-is when we want to share dependencies,
    which is handy when the dependency module is internal (from the same library).
  - They may be passed as a subset, or not passed at all in other cases,
    which is handy when the dependency module is external (from another library).

### Module exports

The module would need to export its factory function **and**
its implementation function.

```javascript
module.exports = MyModuleFactory;
module.exports.implementation = MyModuleImplementation;
```

Thanks to Javascript's extraordinary flexibility,
we can set the implementation function as a property of
the factory function (without altering the behaviour of either).
This makes for convenient usage:

```javascript
// When we wish to use the implementation function for convenient instantiation,
// using default implementations for all dependencies
const myModuleInstance = require('./my-module.js')({})({
    optionA: true,
    optionB: 123,
});
```

```javascript
// When we wish to use the factory function directly,
// explicitly specifying all dependencies
const myModuleInstance = require('./my-module.js')({
    InternalModule: require('some-alternative-implementation'),
    ExternalModule: require('external-module/something-else')(),
})
myModuleInstance.init({
    optionA: true,
    optionB: 123,
});
```

## Dependency injection

Dependency injection is one manner of
**inversion of control** where module lookup is contextualised -
in this case by the factory function.
It allows, therefore, decoupling through layered abstractions.
Software architecture theories aside,
this results in two interesting practical applications:
**Swappable dependencies** and **complete mocking**.

### Swappable dependencies

With dependency injection,
each module accepts dependencies that are passed into its factory function.
Whether a dependency is acceptable is determined by whether it is capable of
performing the actions required of it (instead of its actual type).
This, coupled with loose types in Javascript,
makes it possible to do duck typing
(if it walks like a duck, and quacks like a duck, it must be a duck.)

```javascript
function MyModuleFactory({
    Print
}) {
    doSomething(x) {
        // do stuff
        Print('something done', x);
    }
    /*...*/
}
```

The above factory function takes in a depdendency named `Print`,
which it calls with a variable number of arguments
that it wishes to output.

```javascript
const generalInstance = MyModuleFactory({
    Print: console.log,
});
```

The default implementation of `Print` is simply an alias to `console.log`.

```javascript
const nodejsOnlyInstance = MyModuleFactory({
    Print: function(...args) {
        process.stdout.write(`${Date.now()} ${util.format.apply(undefined, args)}\n`);
    },
});
```

However, further along, someone might want a different implementation
for every line that was printed,
and come up with this implementation.

```javascript
const logInstance = MyModuleFactory({
    Print: function(...args) {
        fs.appendFile(logFilePath, `${Date.now()} ${util.format.apply(undefined, args)}\n`);
    }
});
```

Even further along, someone might decide to repurpose
the `Print` completely for logging instead,
and use yet another alternative implementation.

All three implementations of `Print` have defined their external interfaces
indentically (a single function which accepts any number of parameters),
thus satisfying the duck typing requirement.
However, what they do internally varies significantly from one to the next.
Of note here, is that `MyModuleFactory` only needs to care about the
external parts of the dependencies,
and does not need to care about their internals at all.

### Complete mocking

Once we attain dependencies that are swappable,
it is possible to acheive a completely different,
and more **powerful** form of mocking.
This is especially useful, in the context of Javascript,
when it comes to globals.

The most "notorious" offenders amongst these are the time-related globals,
such as `setTimeout`, `setInterval`, etc.
These functions are rather difficut to mock or stub.
There are even quite a number of mocking libraries published on `npm`
specifically dedicated to solving this problem.
Each one of them has its own set of quirks due to the techniques it uses,
or its implementation details.
Additionally each one has its own API,
which the developer must learn,
and are quite often very different from the API of the global functions
that they help to mock.
The result of this are numerous solutions that almost work,
but fall short of the mark.

Using dependency injection, however,
one can completely side step the need to even use a 3rd-party library,
and simply inject an alternative,
whose API corresponds exactly with the thing being mocked,
but whose implementation is tailored to the test being run.

```javascript
it('should get result from doSomething after 500ms', () => {
    let waitTimer;
    let timer;
    let waitFn;
    function mockSetTimeout(fn, time) {
        waitFn = fn;
        waitTimer = time;
        timer = 0;
    }
    function mockTick(time) {
        timer += time;
        if (timer >= waitForTimer) {
            waitFn();
        }
    }
    const myTimeInstance = MyTimeModule.impl({
        setTimeout: mockSetTimeout,
    })({});
    const doSomethingHasCalledBack = false;
    myTimeInstance.doSomething((err, result) => {
        doSomethingHasCalledBack = true;
    });
    mockTick(200);
    // assert `doSomethingHasCalledBack` equals `false`
    mockTick(300);
    // assert `doSomethingHasCalledBack` equals `true`
});
```

The above shows a concise example of mocking `setTimeout`,
where the developer would only need to learn `setTimeout`'s API,
and would *not* need to learn any test/ mocking library's API
(because no 3rd-party library was necessary to begin with).
This form of mocking is more **complete** than
the form of mocking that be acheived via a 3rd-party library because
it would acts as a gateway and thus allows you to interact with the mock
only in the predefined ways that it anticipates and caters for;
whereas by injecting mocks directly,
you have complete control over defining the mock's behaviour.

## Credits

Thanks to [@indieisaconcept](https://github.com/indieisaconcept)
for explaining the
[`_implementation` technique](https://github.com/newscorpaus/implementation),
and his feedback on this dependency injection technique.