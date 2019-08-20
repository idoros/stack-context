**THIS IS NOT PRODUCTION CODE** :)

We wanted to go over the interesting ideas in the original post [Algebraic effects for the rest of us](https://overreacted.io/algebraic-effects-for-the-rest-of-us/) by Dan Abramov, where he explains what algebraic effects are and how they might be implemented in a futuristic version of ES.

In this document we hope to go over use cases that can benefit from such a solution and see how far we can go in implementing a mechanism that can work today in Javascript.

---

# The use case

// ToDo: add context use case

---

# Solution...

## The basic

In the original post, the feature is using `try/perform/handle` extending on the principles of `try/throw/catch`, but with a small change - while the stack between the `throw` and `catch` is stopped and discarded, the stack between the `perform` and `handle` can be rolled back and resolved using a `resume with` keyword.

Essentially, allowing the executed code to request "values" from a call higher in the execution stack, without requiring any special handling across the levels between them.

This is reminiscent of how React context passes from provider to consumer through the rendering tree, but instead of "components" listening to the context, we have "call executions" that can access some "context API".

> **extra thought:** it might seem like React components are somehow more stateful or persistent, however a call execution has input arguments like props, closure variables that can act like state and keep-alive/updating for as long as they are referenced.

To do this we need to be able to read "values" from a "context" that was passed on the call stack and also write to a new extended "context" that will propagate to any execution calls down the line.

## Can we create a polyfill?

While we can't build on top of the `try/catch/throw` concept with JavaScript (no way to simply resume from a throw), we can absolutely run code in a closure up the stack today, and we do it all the time through callbacks.

Let's imagine the runtime providing us with a magical `stackContext` keyword that allows us to request values from whoever executed our code:

```js
// consume
function getName(user) {
    let name = user.name
    if (name === null) {
        name = stackContext.askName()
    }
    return name
}
// provide
function app() {
    stackContext.askName = () => {
        return 'Arya Stark'
    }
}
```

Now we need to figure out where the `stackContext` reference comes from. We can wrap our context functions with something that provides the context as an argument, accepts modifications to create an extended context and pass it down the stack:

```js
// consume
const getName = context((stackContext, user) => {
    let name = user.name
    if (name === null) {
        name = stackContext.askName()
    }
    return name
})
// provide
const app = context(stackContext => {
    stackContext.askName = () => {
        return 'Arya Stark'
    }
})
```

Notice that we have reached a slightly different API compared to the original post, we are no longer calling a generic `perform` statement that propagates through the `try/handle` chain. Instead we have a context reference that we control.

## Types?

Since our polyfill already has a first argument for the context data, then we might as well slap a type to it:

```ts
const getName = Symbol('get-name')
interface NamesApi {
    [getName](): string
}
context((stackContext: NamesApi) => {
    // read
    stackContext[getName] // $ExpectType () => string
    stackContext[getName]() // $ExpectType string
    // extend for nested stack calls
    stackContext[getName] = () => {
        return 'overridden name'
    }
})
```

Borrowing from Typescript [(`this: T`) => {}](https://www.typescriptlang.org/docs/handbook/functions.html#this-parameters) parameter syntax, it might be possible to add the polyfill during transpilation.

If such an ability would be implemented into the language (in 2025), it would be nice to have the `stackContext` available without adding it to the argument list. However a benefit can still be gained by explicitly defining the `stackContext` type somehow.

> **Note**: knowing that a function will need a specific context type might allow us to statically suggest missing or mismatched context values (given marked top level execution points).

## Async?

The big issue with this type of injection is asynchronous code. Since our context is constructed out of the running call stack, any function that is delayed, is essentially disconnected from our `stackContext` value.

To be honest, I'm a little bit unsure what's the behavior of the async example in the original post - when the `perform` is delayed, is everything suspended until the `resume with` is called?

In our case we require no new ECMAScript features, so accessing the context acts like a normal execution. We can then provide a way to do a manual patch of the context back into the execution once it resumes:

```js
context(async stackContext => {
    await goDoSomethingForAWhile()

    // special method to patch back the context
    stackContext[continueAsync]()
    // ...continue running code from the same context
})
```

> Unfortunately this requires unrelated async methods to "play along" if they are in the path of our context :(

On the other hand, rolling back the context down the stack can probably be handled automatically (also error handling).

## Top level context?

In the original post examples, `try/handle` is used directly on the module top level. We avoided touching a `stackContext` on our top level because our polyfill is implemented by wrapping functions and providing them with a context API.

This is harder to do with the solution we have, since we have no simple way to wrap the module execution function. However, in a hypothetical world a module defining a top level `stackContext` could be used to provide a default value context for the functions that are defined within it:

```js
export const generateUniqueId = Symbol('gen-id')

// default generator
stackContext[generateUniqueId] = new UniqueIdFactory()

// use the context
export const doSomething = context((stackContext) => {
    // get from call stack or fallback to module default
    const uniqueId = stackContext[generateUniqueId]()
}
```
