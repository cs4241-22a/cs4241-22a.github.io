# Modules and bundling like a pro: Vite

There are a variety of systems for modularizing JS code, and a huge number of libraries have been created over the users for this. The simplest method is to check for the existence of a global object and place our code into this object. If it doesn’t exist, create it first. For example:

```js
if( typeof window.app === 'undefined' ) window.app = {}

window.app.mymodule = {
  foo:1,
  bar() { console.log( this.foo ) }
}
```

Using this strategy, we can safely import any number of files via script tags and not have to worry about the order that they’re loaded in, assuming only one of them makes use of `window.onload` to start our app running.

However, this can lead to some spaghetti code. What if one particular module needs to know about the existence of another module so that it can call functions from it? We need some sort of dependency system for this. [Vite](https://vitejs.dev/) is one such tool for the client that takes advantage of ES modules in the browser.

## Before we get to Vite, what about importing / exporting in node.js?
Node.js conveniently includes a `require` function to include files and a `module.exports` object for exporting objects / functions / values. This method of exporting / requiring modules is known as `CommonJS`. We've used require a lot in our servers, but you might not have used `module.exports` yet:

```js
/************ FILE 1: module.js, a module to export **********/

module.exports = function() {
  console.log( 'a function' )
}

/************ FILE 2: importing our module ************/
const demomodule = require( './module.js' )

demomodule() // logs 'a function'
```

Any module can `require` any other module, and node.js will resolve circular dependencies whenever possible.

## ES6 import / export
ES6 added `import` and `export` keywords, however, it's only been in the last few years that browsers have added support (it's also only available in the most recent versions of Node). This module syntax is shortened to the name ESM for convenience, to distinguish it from CommonJS.

#### The basics
1. Let's make a basic module that exports a single number and name it `module.js`:
```js
export default 42
```

The `default` keyword tells us that `42` is the standard export for this file. You can only have one `default` export per module, however, we'll see soon that named exports enable us to export multiple values.

2. Let's create an `app.js` file that imports this value:
```js
import anumber from `module.js`

console.log( 'anumber is:', anumber )
```
3. OK, so we can include `app.js` in a web page with two extra steps:
   - We need to run a server with CORS enabled (e.g. http-server . -p 30000), can't load via `file://`
   - We need to add the `type="module"` attribute to our `<script>` tag.

```html
<!doctype html>
<html lang='en'>
<head>
  <script src='app.js' type='module'></script>
</head>

<body></body>

</html>
```

4. If you open the page and developer's console, you should see the number `42` printed.

#### Slightly more complex
In this next example, we'll export a file named `utilities.js`, that will provide us with a couple of different functions to use. We'll export these using `named` exports, instead of `default` exports like we did in our previous example.

```js
// utilities.js
const log = ( obj, prop ) => console.log( `${prop}: ${obj[ prop ]}`)

const square = num => num * num

export { log, square }
```

Now we can import and use those functions in our `app.js` file:

```js
// app.js
import { log, square } from './utilities.js' 

const app = { myvalue: square(42) }

log( app, 'myvalue' )
```

We can import `app.js` using the same HTML / server we used in the previous example. In the next example we'll look at how to make sure this script can run in all browsers, using `browserify` and `babel`, a transpiler for JavaScript.

## Demo w/ Three.js and ES Modules
Let's walk through using various ways to import modules. We'll use [three.js](http://threejs.org/), the most popular library for 3D in the browser, as an example module to play with.

1. Open some type of bash shell. Create a new directory `mkdir mydir` and cd into it.
3. Install three.js via npm. `npm install three --save`
4. OK, let’s make our main html file. We’ll include single JS file (named `main.js`). We *have to include the 'module' type attribute in order to be able to import other libraries inside of this file*. Don't forget this!

```html
<!doctype html>
<html lang='en'>
  <head>
    <style> body { margin:0 } </style>
    <script src='./main.js' type="module"></script>
  </head>
  <body></body>
</html>
```

6. Now we need to create our main JavaScript file. Before we learn how import three from the module we just installed, let's look at using it from a CDN:

```js
// import our three.js reference
import * as THREE from 'https://unpkg.com/three/build/three.module.js'

const app = {
  init() {
    this.scene = new THREE.Scene()

    this.camera = new THREE.PerspectiveCamera()
    this.camera.position.z = 50 

    this.renderer = new THREE.WebGLRenderer()
    this.renderer.setSize( window.innerWidth, window.innerHeight )

    document.body.appendChild( this.renderer.domElement )
    
    this.createLights()
    this.knot = this.createKnot()

    // ...the rare and elusive hard binding appears! but why?
    this.render = this.render.bind( this )
    this.render()
  },

  createLights() {
    const pointLight = new THREE.PointLight( 0xffffff )
    pointLight.position.z = 100
    this.scene.add( pointLight )
  },

  createKnot() {
    const knotgeo = new THREE.TorusKnotGeometry( 10, .1, 128, 16, 5, 21 )
    const mat     = new THREE.MeshPhongMaterial({ color:0xff0000, shininess:2000 }) 
    const knot    = new THREE.Mesh( knotgeo, mat )

    this.scene.add( knot )
    return knot
  },

  render() {
    this.knot.rotation.x += .025
    this.renderer.render( this.scene, this.camera )
    window.requestAnimationFrame( this.render )
  }
}

window.onload = ()=> app.init()
```

## Using Vite
Vite let's us use our ES Module syntax but figures out exactly what JS (and CSS too!) we need to put on the server and wraps it all up nicely for us. 
Otherwise you can quickly get into a situation where you're having to upload your entire `node_modules` folder to your server... take a look 
inside of that folder, it's usually pretty gnarly. Vite will figure out exactly what JS is needed for a given project and make it 
easily available for you. Vite has a [great guide for getting started](https://vitejs.dev/guide/) with it.

1. Run `npm create vite@latest` from a terminal. Choose "vanilla" and "javascript" for the template, but note there are also templates for building projects 
using React and Svelte, two frameworks we'll be discussing in class on Monday.

2. `cd` into the generated directory and install three `npm install three`

2. Instead of loading three.js from our CDN, use the following line of code in `main.js` (comment out or replace the previous module loader):

```js
import * as THREE from 'three'
```

3. Launch the Vite dev server. In a terminal, make sure the working directory is the top-level of your project and run: `npm run dev`. Go ahead and check out the link the server gives you, which will automatically update as you change files.

4. Take a look at all the files that have been loaded in the Sources tab of your browser's developer tools to get a sense of the magic that is happening.

5. OK, pretty cool, but chances are you'll want to run your own server to serve up your site, and you definitely don't want to be running the Vite dev server in production. We can tell Vite to wrap everything up nice and neat for us using `npm run build`. Vite then creates a `dist` folder; if you launch a standard server you can view all your files. You could also upload your dist folder to Glitch / Heroku etc. 
