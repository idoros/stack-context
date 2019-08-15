**THIS IS NOT PRODUCTION CODE**

An exercise on the mechanism from the interesting post [Algebraic effects for the rest of us](https://overreacted.io/algebraic-effects-for-the-rest-of-us/) by Dan Abramov.

## What does it do?

Putting aside the specific `try/catch` like syntax that Dan choose, what we have is some code at the top of the stack providing "stuff" for the executed code in a lower part of the stack, with no direct intervention of the steps between them.

There are other similar mechanisms that we use that do the same thing. Generally this sounds like dependency Injection that passes "stuff" down to parts of the application.

**Passing down through what?**

While dependency injection is implemented in many ways, Dan specifically talks about passing down the call stack, unlike React Context that passes down the rendering tree, our mechanism provides information through the execution stack.

## Can we create a polyfill?

While Dan's hypothetical syntax relies on traveling up the stack through a mechanism like `try/catch/throw` cannot be implemented with JavaScript *(there is no way back once we throw)*, we can absolutely run code in a closure up the stack today, and we do it all the time through callbacks.

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