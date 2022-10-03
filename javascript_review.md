# JS review

## Variable declaration
Three ways to declare variables: `let`, `const`, and `var`

`let`: a block-scoped value/reference that is going to change over time
`const`: a block-scoped a value/reference that is not going to change over time
`var`: a function-scoped variable that will change over time

`var` is strange! avoid `var`

```js
test = function() {
  let a = 0; // available in entire function block
  
  if( a === 0 ) {
    let b = 1; // only available in this block
    var c = 1; // available outside of block... what?
    console.log( 'inside block:', a,b,c )
  }
  console.log( 'outside block c:', c )
  console.log( 'outside block b:', b )
}
```

when run yields:

> inside block: 1 1
> outside block c: 1
> VM569:9 Uncaught ReferenceError: b is not defined
>    at test (<anonymous>:9:36)
>    at <anonymous>:1:1
  
Our reading argued that you should use `var` to indicate variables that are deliberately scoped to entire functions, even though a top-level `let` variable has the same scope. I think this advice is overthinking it.

## Two ways to define functions

```js
const test = function(a,b,c) {
  return a + b + c
}
// vs.
const arrow = (a,b,c) => a + b + c
```

Arrow functions are really expressive for chained data processing:

```js
// fetch an array of data as json, double all values, 
// remove all values over 10, and log
fetch('/readArray')
  .then( response => response.json() )
  .then( jsonArray => 
    jsonArray
      .map( v => v*2 )
      .filter( v => v < 10 ) 
  )
  .then( console.log )
```

much better (?) than:

```js
fetch('/readArray')
  .then( function( response ) { return response.json() } )
  .then( function( jsonArray ) { 
    return jsonArray
      .map( function(v) { return v*2 } )
      .filter( function(v) { return v < 10 } ) 
  })
  .then( console.log )
```

## Scope / Closures
As mentioned earlier, variables in JavaScript are scoped to functions / blocks. We can use this for some interesting effects, like creating "private" variables that can only be retrieved using appropriate getter methods.

```js
function Car() {
  const data = 42
  this.getData = ()=> data
}
const test = new Car()
console.log( test.data, data ) // undefined, undefined
console.log( test.getData() )  // 42

// functions in functions in functions oh my
// inner functions always have access to the data in surrounding functions.
// these variables are called 'upvalues', and the combination of a function 
// and the upavalues in its scope is called a 'closure'

function topfnc() {
  const a = 0
  function midfnc() {
    const b = 1
    console.log( 'mid:', a,b )
    return function innerfnc() {
      const c = 2
      console.log( 'inner:', a,b,c )
    }
  }
  console.log( 'top:', a )
  return mid
}
mid = topfnc()    // 0, 
inner = mid()     // 0, 1
inner()           // 0, 1, 2
  
```

## Prototypes vs Classes

```js
// prototype + mixin
const grandparent = { 
  test() { console.log( 'proto:', this.value ) } 
}
const parent = Object.assign(
  Object.create( A ),
  { value: 42 } // mixin
)
const child = Object.create( parent )
child.test() // proto: 42

// classes
class Grandparent {
  test() { console.log( 'class:', this.value ) }
}

class Parent extends Grandparent {
  constructor() { 
    super()
    this.value = 42 
  }
}

const _child = new Grandparent()
_child.test() // class: 42
```

## What the heck is this?

`this` is known as the execution context in JavaScript. It changes depending on how functions it is used inside of are created and/or called.

```js
// *** Default binding: ***
console.log( this ) // -> In browser, window, unless in strict mode

// *** Implicit binding ***
// assume the object calling the function should be the excution context
const obj = {
  value:42,
  test() { console.log( this.value ) }
}
obj.test()

// *** Explicit binding ***
// tell the function what the execution context should be
obj.test.call({ value:Infinity })

// *** Lexical binding ***
// uses the same execution context as when the function was authored
// only works for arrow functions
const obj2 = {
  test: ()=> console.log( this ) 
}
obj2.test() // whoa, window!

// lexical binding is great for adding event handlers to DOM objects
const buttonMaker = {
  value:42,
  create() {
    const btn = document.createElement('button')
    btn.innerText = 'click me'
    onclick = ()=> console.log( this.value )
    document.body.appendChild( btn )
  }
}
buttonMaker.create() // 42

// *** Hard Binding ***
// create a copy of a function with a permanently bound execution context
const myfunc = function() {
  console.log( this.value )
}
const boundFunc = myFunc.bind({ value:42 })
boundFunc() // 42
```

Hard binding was used in the [three.js example from class](https://github.com/cs4241-21a/cs4241-21a.github.io/blob/main/webaudio_canvas_three.md#threejs)

## How do we structure our client-side applications?

We can use module syntax for this. You need to specify that your file is a ES Module to use this syntax e.g. `<script src='app.js' type='module'>`.
  
```js
import Login from 'login.js'
import View  from 'view.js'

const App = {
  init() {
    this.login = Login()
    this.view  = View()
    
    return this  
  }
}

window.addEventListener( 'load', App.init.bind( App ) )
```

In the above example, `login.js` would look something like:

```js
const Login = function() {
  const login = {
    submit() {
      fetch( '/login' )
    }
  }
}

export default Login
```

You can then use systems like Snowpack/Webpack/Vite to link them all together. See the [class notes on modules]([https://github.com/cs4241-22a/cs4241-22a.github.io/blob/main/prototypes_and_modules.md#modules-and-bundling-like-a-pro-snowpack--webpack](https://github.com/cs4241-22a/cs4241-22a.github.io/blob/main/using_modules.md)) for more info.
