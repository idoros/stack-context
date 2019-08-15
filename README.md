**THIS IS NOT PRODUCTION CODE** :)

We wanted to go over the interesting ideas in the post [Algebraic effects for the rest of us](https://overreacted.io/algebraic-effects-for-the-rest-of-us/) by Dan Abramov, where he explains what are algebraic effects and how they might be implemented in a futuristic version of ES.

In this document we hope to go over use cases that can benefit from such a solution and see how far we can go in implementing a mechanism that would work today (**not for production obviously**).

---

# The use case

// ToDo: add context use case

---

# Solution...

## The basic

In the post, the feature is using `try/perform/handle` extending on the principles of `try/throw/catch`, but differ slightly - while the stack between the `throw` and `catch` is stopped and discarded, the stack between the `perform` and `handle` can be rolled back and resolved using a `resume with` keyword.

Essentially, allowing the executed code to request "stuff" from a higher call in the execution stack, without having any special handling in the levels between them.

Sounds like how React context passes from provider to consumer through the rendering tree, but instead of "components" listening to the context, we have "call executions" that can access some "context" API.

> **extra thought:** it might seems like React components are somehow more stateful or exist longer in time, but a call execution has arguments like props, variables that can change like state and can stay alive and updating for as long as they are referenced.

So we need to be able to add and read information to a "context" that will extend any previous "context" that was provided along the execution stack.

## Can we create a polyfill?

While we can't build on top of the `try/catch/throw` concept with JavaScript _(no way back from throw)_, we can absolutely run code in a closure up the stack today, and we do it all the time through callbacks.

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
// consumer
const getName = context((stackContext, user) => {
    let name = user.name
    if (name === null) {
        name = stackContext.askName()
    }
    return name
})
// provider
const app = context(stackContext => {
    stackContext.askName = () => {
        return 'Arya Stark'
    }
})
```

> Notice that we have a slightly different API then the original post, we are no longer calling a generic `perform` that would propagate through the `try/handle` chain. Instead we have a context data that we access.
>
> Theoretically anything can be passed, however to avoid namespace issues and mutations we use symbol keys to function values.

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
        /*contextual name*/
    }
})
```

> **extra thought:** borrowing from Typescript [(`this: T`) => {}](https://www.typescriptlang.org/docs/handbook/functions.html#this-parameters) parameter syntax, it might be possible to add the polyfill at transpile.
>
> **also**: knowing that a function will need a specific context type might... maybe... allow to statically suggest missing or mismatched context values... at marked top level execution points...

If such an ability would be implemented into the language (in 2025), it would be nice to have the `stackContext` available without adding it to the argument list. However types would still benefit from having the type.

## Async?

The big issue with this type of injection is asynchronous code, since our context is constructed out of the running call stack, any function that is delayed, is essentially disconnected from our `stackContext` value.

To be honest, I'm a little bit of unsure what the behavior of the async example is - when the `perform` is delayed, is everything suspended until the `resume with` is called?

Anyway, we can do an ugly manual patch the context value back into the execution once it resumes:

```js
readContext(async (stackContext, user) => {
    const contextStack = stackContext
    await goDoSomethingForAWhile()
    // special method to patch back the context
    stackContext[continueWithAsync](contextStack)
    // ...continue running code from the same context
})
```

> This is far from perfect, and it requires unrelated async methods to "play along" if they are in the path of our context : (

On the other hand, rolling back the context down the stack can probably be handled automatically (also error handling).

## Top level context?

In the post examples, `try/handle` is used directly on the module top level. We avoided touching a `stackContext` on our top level because our polyfill is implemented by wrapping functions and providing them with a context API.

This is harder to do with the solution we have, since we have no simple way to wrap the module execution function. However, in a hypothetical world a module defining a top level `stackContext` could be used to provide a default value context for the functions that are defined within it:

```js
export const generateUniqueId = Symbol('gen-id')
// default generator
stackContext[generateUniqueId] = new UniqueIdFactory()
// use the context
export const doSomething = readContext((stackContext) => {
    // get from call stack or fallback to module default
    const uniqueId = stackContext[generateUniqueId]()
}
```
