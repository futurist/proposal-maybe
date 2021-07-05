# ECMA Proposal: Maybe

## Reason

Refer to the[ first version of JS engine](https://2ality.com/2013/10/typeof-null.html.), we can review the below:

1.  The idea of null value (JSVAL_NULL) is for machine code NULL pointer (this idea itself is a [“billion-dollar mistake.”](https://en.wikipedia.org/wiki/Tony_Hoare#Apologies_and_retractions)), and it's value returned by `typeof` (`object`) is **a famous bug** and mistake in early age of programming language design, instead, all these things should be result in errors, it's a tricky way to avoid handling errors.
2.  In Javascript/ECMAScript, The idea of undefined value (JSVAL_VOID) is for a number outside the integer range.

Above two concepts are both heavily abused in modern programming and far away from their original ideas, but **used mostly as an special case/unheathy/uncertain(optional) state in nowadays.** But some times, people don't pay attention to these states, and result in errors caused by `null/undefined`.

Modern programming gradually matched with the real world, from certainty to uncertainty, but javascript lacks one important state: uncertainty, or Maybe state, which has been introduced into modern languages like TypeScript, Rust, Swift etc, and can be a good substitution of null.

[TypeScript team totally avoid using ](https://basarat.gitbook.io/typescript/recap/null-undefined#final-thoughts)[null](https://basarat.gitbook.io/typescript/recap/null-undefined#final-thoughts) without any problems, and there is a [strictNullChecks](https://www.typescriptlang.org/tsconfig#strictNullChecks) option in TypeScript.

Existing languages like Haskell has [Maybe](https://hackage.haskell.org/package/base-4.9.1.0/docs/Prelude.html#t:Maybe), Rust has [Option](https://doc.rust-lang.org/std/option/enum.Option.html), and Java has [Optional](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/Optional.html), Swift has [Optional](https://developer.apple.com/documentation/swift/optional), Kotlin has [Null Safety](https://kotlinlang.org/docs/null-safety.html), and many tries of the concept in JS community like below:

-   https://github.com/chrissrogers/maybe

-   https://crocks.dev/docs/crocks/Maybe.html

Wait, we already have [Optional Chaining](https://github.com/tc39/proposal-optional-chaining), [Nullish coalescing Operator](https://github.com/tc39/proposal-nullish-coalescing), are those handled the case? The problem here is they are just simple operators, converting from one null/undefined into another undefined, or may produce more undefined, thinking the nature of undefined / null, they[ are evil](https://blog.ndepend.com/null-evil/) too.

This proposal try to rethinking the way of Nullish/Optional/Maybe from modern language perspect, and try to get out of the old buggy null way, and try to eliminate the null, and try to introduce a new way to programming without them in language level.

Another point is to handle error state, to propose a new elegant way to avoid writing try...catch statement everywhere.

In this proposal, the undefined, null or Error state is called unhealthy DownState, and the opposite is healthy UpState, the up and down term is easy to understand as healthy/unhealthy, and you should avoid continuing operate on an unheathy state!


## High-level API

-   ## States of Maybe:

The new global constructor Maybe can have two states:

1.  UpState has **UpValue** **,** a **healthy** state, the value can be consumed.
2.  DownState has **DownValue** (**Empty** or **Error**) **,** an **unhealthy** state, the value cannot be consumed, the program should abort and throw a **MaybeError**.

-   ## The Maybe value

-   Maybe.up(UpValue), setup UpValue, return a maybe instance in UpState. The argument is **mandatory**, and will throw TypeError when it is undefined/null.

-   Maybe.down(DownValue), setup DownValue, return a maybe instance in DownState, DownValue is **optional**, and will be set to Maybe.none when it's undefined/null.

-   Maybe.set(value), is helper methods to invoke Maybe.up when value is not undefined/null/Maybe.none, or else invoke Maybe.down.

Example:

```js
var a = Maybe.down(); // DownValue will be set to: Maybe.none
a.up(1);  // a from down state to up state, with value 1

var b = Maybe.up({x: 1});

b.down();  // b is to empty down state
b.down("Oops..."); // b is to Error down state, with DownValue "Oops..."

// set can auto switch up/down based on value

b.set(1); // UpState: 1
b.set(null);  // DownState: Maybe.none
```

-   ## The Maybe constructor

The Maybe(value) constructor is a shortcut for Maybe.set(value)

```js
var a = Maybe(1);  // same as Maybe.up(1)
var a = Maybe({x:1});  // same as Maybe.up({x:1})

// or nested
var b = Maybe({x: Maybe(1)}); // same as Maybe.up({x: Maybe.up(1)})

var c = Maybe(); // same as Maybe.down()
var c = Maybe.down('Error');
```

It's important to note that passing undefined/ null will be ignored and result in a down state.

```js
// below 3 lines all result in down states

var a = Maybe(); // same as Maybe.down()
var a = Maybe(null); // same as Maybe.down()
var a = Maybe(undefined); // same as Maybe.down()

a.ok === false;
```

Make empty as down help eliminate the `undefined / null`.

-   ## Operations

### OK for the healthy UpState

Use .ok to indicate the heathy state of maybe:

```js
a.ok // true if up, false if down
if(a.ok){...} // good to use with if statement
```

### Use the .unwrap to Get the UpValue

Maybe type instance can get it's UpValue via unwrap():

1.  When the maybe is in UpState, **return** the UpValue.
2.  When the maybe is in DownState, **throw** MaybeError(DownValue)

unwrap can have a function as argument, like below:

unwrap(value->anotherValue)

This allows transform value functional when in UpState.

### Use ! as syntax sugar for unwrap()

The new proposed maybe **!** operator is a syntax sugar for maybe.unwrap()(without argument).

Use `!` Instead of unwrap() is proposed since this can lead to an unheathy state and abort the execution, the programmer should be careful for every `!` showing-up! The `!` syntax is good for this purpose, or maybe `!!` like [in kotlin](https://kotlinlang.org/docs/null-safety.html#the-operator) as an alternative (more strength in emotion).


```js
// below is same:

var a = Maybe({x: 1});
var a = Maybe.up({x: 1});

/** Up State **/

// same as: a.unwrap().x === 1
a!.x === 1 // only `a` is in up state, `x` can be accessed!

/** Down State **/

var a = Maybe();
// same as: a.unwrap().x
a!.x // throw MaybeError

var b = Maybe.down("Oops");

// same as: b.unwrap().x
b!.x // throw MaybeError: "Oops"
```

Maybe also want to let JS code more friendly to `try...catch` by avoid using it, and led to [rust way of error handling](https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html) using `?` (here we use `!`), by nested Maybe, that is, if one thing is maybe, all parents affected will become Maybe, that way, will led to have more elegant way to handle errors than many `try...catch`.

But using `!` to handle the error is different and better than rust's `?`, is that `!` is lazy (later binding when used), and the error produced by Maybe will not immediately halt the function, so have a chance to let user handle it in the same function scope.

For example, the below `try..catch` code:
```js
function getUserName(url) : String | NetworkError {
  let user
  try{
    user = getUser(url);
    return  user.name
  }catch(e){
    return e;
  }
}
```

Compared, using below `Maybe` and `!` way:

```js
function getUserName(url) : Maybe<String, NetworkError> {
  const user = maybeGetUser(url);
  return user!.name
}
```

Since the `!` is syntax sugar of `unwrap`, if cannot unwrap, error type should be `NetworkError`(DownState), else you get `String` type (UpState).


-   ### MaybeError for the unhealthy DownState

When unwrap a DownState value, the program should not continue since it's in unheathy state, here we could use throw to abort the execution, but in fact it's not a true error, it's just indicate an abort from an unheathy state, so here we use a new MaybeError error type to indicate the state, it's more like a message passing:

1.  The error.message will be set to a string containing DownValue or "" when empty
2.  The error.value will be set to DownValue, absent/undefined/null will fall back to Maybe.none.

When unwrap after the maybe.ok check, it's always safe and never throw MaybeError.

```js
Maybe.down().unwrap();  // throw MaybeError
Maybe.down('Bad!').unwrap(); // throw MaybeError: Bad!

var a = Maybe({x: 1});
a.ok && a!.x;  // never throw here
```

-   ## Helper methods

### .else(fallbackValue) -> Value

1.  Call and return .unwrap when in UpState
2.  Or return a fallbackValue value when in DownState, fallbackValue can be a function

```js
// set to defaultValue when down state
Maybe().else('fallbackValue').unwrap() === 'fallbackValue'

// set to it's value when in up state
Maybe(1).else('fallbackValue').unwrap() === 1

// .else can use a function
Maybe().else(()=>'fallbackValue').unwrap() === 'fallbackValue'
```

### .map(upFn, downFn) -> Maybe(Value)

1.  When in UpState, get Value from calling upFn, and return Maybe(Value)
2.  When in DownState, get Value from calling downFn, and return Maybe(Value)


```js
// .map from a maybe to another maybe
Maybe.up(1).map(v=>v*2) // created: Maybe(2)
Maybe.down('Oops').map(v=>v*2, v=>v+'!') // created: Maybe("Oops!")

// chain them
Maybe().map(v=>'Hello').map(v=>v+'World').else('Sorry').unwrap()  // 'Sorry'
Maybe(1).map(v=>'Hello').map(v=>v+'World').else('Sorry').unwrap()  // 'HelloWorld'
```

Think of the `.map` and `.else` chain have same behaviors as Promise's `.then` and `.catch` chain, for example, **`.else`** will be called only on DownState happened in the chain.

The .else(fn) is a short form of .map(v=>v, fn), since the form used a lot for quickly getting out of an unhealthy DownState.


-   ## Convert to Promise

```js
var a = Maybe()
var promise = a.toPromise().then(v=>console.log("up state:" + v))
a.up(1); // console.log: up state: 1

var promise = a.toPromise().catch(v=>console.log("down state:" + v))

a.down('Oops...'); // console.log: down state: Oops...
```

**Q: It's possible to make** **.up** **and** **.down** **to return a new Promise to listen to the state change?**

-   ## Working with [Optional Chaining](https://github.com/tc39/proposal-optional-chaining) and [Nullish coalescing Operator](https://github.com/tc39/proposal-nullish-coalescing)

```js
var a = Maybe({x: Maybe({y: 1})});
a?.x?.y === 1 ; // ---> Should discuss it's behavior here

var b = Maybe();
(b ?? "ok") === "ok"; // This should be same as b.ok ? b! : "ok";

var c = Maybe(3);
(c ?? "ok") === 3;    // This should be same as c.ok ? c! : "ok";
```

**Q: These cases should be discussed more.**

# Examples

-   ## Substitute undefined/null usages example

```js
// Before using maybe:
function trueOrNull(i){
    return Math.random()>0.5 ? {x:i} : null
}
[1,2,3].forEach((i)=>{
    const val = trueOrNull(i)
    // null is evil!
    if(val !== null) {
        console.log("value:", val.x)
    } else {
        console.log("Unhealthy state!")
    }
})
```

**We can use** **Maybe** **to avoid using** **null** **:**

```js
// Helper function: convert to Maybe
function maybeTrueOrEmpty(i){
    return Maybe(trueOrNull(i)) // null -> down state
}

// use Maybe
[1,2,3].forEach((i)=>{
    const maybe = maybeTrueOrEmpty(i)
    if(maybe.ok){
        console.log("value:", val!.x)
    } else {
        console.log("Unhealthy state!")
    }
})
```

**Or use** **.map**

```js
[1,2,3].forEach((i)=>{
    const maybe = maybeTrueOrEmpty(i)
    // .map just like `.then` in Promise
    maybe.map(
        v=>console.log("value:", v.x),  // up state
        ()=>console.log("Unhealthy state!")  // down state
    )
})
```

-   ## Use `!` to make try...catch more elegant example

```js
// Helper function: convert to Maybe
function getMaybe(fn){
    return (...args){
        try{
            return Maybe(fn(...args))
        } catch(e) {
            Maybe.down(e)
        }
    }
}

var defaultUser = {name: "anonymous"};

async function getUserName(url){
    // fetch may be throw!
    const response = await getMaybe(fetch)(url);  // response is Maybe
    // Using Maybe, above `await` will never need to try...catch
    if(response.ok && response!.ok) {
        const user = await getMaybe(response!.json)() // user is Maybe
        return user.else(defaultUser).unwrap().name
    }
}
```

**Above code :**

1.  More elegant compared to try cach every possible error.
2.  Work perfectly with await since Maybe handled the down state, no need to wrap try catch around await again!
3.  No undefined or null needed!

-   ## Never rejected Promise example

When combined with Maybe and await, a Promise can always be resolved, since the Maybe DownState can be used as a synonym of rejection or undefined/null value, like below:

```js
function maybePromise(){
    return new Promise(resolve=>{
        if(Math.random()>0.5) {
            // when resolved
            // resolve to `Maybe UpState`
            resolve(Maybe({x: 1}))
        } else {
            // when rejected
            // resolve to `Maybe DownState`
            resolve(Maybe())
        }
    })
}

// below never throw
// so no need to try...cache

const maybe = await maybePromise()

if(maybe.ok){
    // indicate resolved
    ...
} else {
    // indicate rejected
}
```

But there's a caveat for this usage: resolve a Maybe(undefined/null) value can indicate rejected, this case is reasonable when the value is treated as a data.

You should not use Maybe if you still use undefined/null as a state (the traditional, old way).

The above code maybe not a good practice in the real world, but as a possible usage of combine `Maybe` and `Promise`.


## References

https://2ality.com/2021/01/undefined-null-revisited.html

