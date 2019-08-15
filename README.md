**THIS IS NOT PRODUCTION CODE**

We wanted to go over the interesting ideas in the post [Algebraic effects for the rest of us](https://overreacted.io/algebraic-effects-for-the-rest-of-us/) by Dan Abramov, where he explains what are algebraic effects and how they might be implemented in a futuristic version of ES.

In this document we hope to go over use cases that can benefit from such a solution and see how far we can go in implementing a mechanism that would work today (**not for production obviously**).

-----

# The use case
// ToDo: add context use case

-----

# Solution...

## The basic

In the post, the feature is using `try/perform/handle` extending on the principles of `try/throw/catch`, but differ slightly - while the stack between the `throw` and `catch` is stopped and discarded, the stack between the `perform` and `handle` can be rolled back and resolved using a `resume` keyword.

Essentially, allowing the executed code to request "stuff" from a higher call in the execution stack, without having any special handling in the levels between them.

Sounds like how React context passes from provider to consumer through the rendering tree, but instead of "components" listening to the context, we have "call executions" that can access some "context" API.

> extra thought: it might seems like React components are somehow more stateful or exist longer in time, but a call execution has arguments like props, variables that can change like state and can stay alive and updating for as long as they are referenced.

So we need to be able to add and read information to a "context" that will extend any previous "context" that was provided along the execution stack.

## Can we create a polyfill?

While we can't build on top of the `try/catch/throw` concept with JavaScript *(no way back from throw)*, we can absolutely run code in a closure up the stack today, and we do it all the time through callbacks.

Let's imagine the runtime providing us with a magical `stackContext` keyword that allows us to request values from whoever executed our code:

```js
// consume
function getName(user) {
  let name = user.name;
  if (name === null) {
  	name = stackContext.askName();
  }
  return name;
}
// provide
function app() {
    stackContext.askName = () => {
        return 'Arya Stark'
    }
}
```

Now we need to figure out where the `stackContext` reference comes from. We can wrap our consumer/provider functions with something that will provide it as an argument and make sure to pass it along:

```js
// consume
const getName = readContext((stackContext, user) => {
  let name = user.name;
  if (name === null) {
  	name = stackContext.askName();
  }
  return name;
});
// provide
const app = writeContext((stackContext) => {
    stackContext.askName = () => {
        return 'Arya Stark'
    }
})
```

> Notice that unlike Dan's syntax, we are no longer calling a generic `perform`, although we could similarly decide to place a perform function, call it, and have the intercepting provider decide if it wants to do something with our data or pass it along.

### What should be passed?

Theoretically anything can be passed, however to avoid namespace issues and mutations we should probably stick to key `Symbols` and function values.

## Types?

Basic types in our closure are easy since we actually have the parameter to type:

```ts
const getName = Symbol('get-name')
interface NamesApi {
    [getName](): string
}
readContext((stackContext: NamesApi, user) => {
    stackContext[getName] // $ExpectType () => string
})
writeContext((stackContext: NamesApi) => {
    stackContext[getName] // $ExpectType () => string
})
```

The types that we are missing is the contextual types that are required by the code that we execute. That means that when we run a function, we cannot know what `stackContext` will be required down the line.

## Async?

The big issue with this type of injection is asynchronous code, since our context is constructed out of the running call stack, any function that is delayed, is essentially disconnected from our `stackContext` value.

What we can do is patch the context value back into the execution once it resumes:

```js
readContext(async (stackContext, user) => {
    const contextStack = stackContext
    await goDoSomethingForAWhile()
    // special method to patch back the context
    stackContext.asyncContinueWith(contextStack)
    // ...continue running code from the same context
})
```

This is far from perfect, and it requires unrelated async methods to "play along" if they are in the path of our context. However, To be honest, Dan wrote about this, but I cannot grasp what would actually happen in his example, where he delayed the response also (is everything suspended until the the `resume` is called?).

## Talking hypothetical

If such a feature would be implemented into the language, it would be nice to have the `stackContext` available without adding it to the argument list. And borrowing from Typescript [this parameter](https://www.typescriptlang.org/docs/handbook/functions.html#this-parameters) syntax, it could be optionally added for type definition and removed in compilation.

Looking at Dan's example there is one thing that is not possible with this version. Dan starts the execution directly in the root of the module and provides the requested values through a `catch` like mechanism.

This is not possible with our solution since we have no top level function. However, in a hypothetical world a module defining a top level `stackContext` could be considered to provide a default value context for the functions that are defined within it:

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