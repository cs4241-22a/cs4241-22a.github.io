# Server
The demo server for this example code has three routes to read, add, and update todos that are stored in memory. 
Make sure this file is placecd and run in the top-level of your React project.

```js
const express  = require( 'express' ),
      app      = express()

const todos = [
  { name:'buy groceries', completed:false }
]

app.use( express.json() )

// this will most likely be 'build' or 'public'
app.use( express.static( 'build' ) )

app.get( '/read', ( req, res ) => res.json( todos ) )

app.post( '/add', ( req,res ) => {
  todos.push( req.body )
  res.json( todos )
})

app.post( '/change', function( req,res ) {
  const idx = todos.findIndex( v => v.name === req.body.name )
  todos[ idx ].completed = req.body.completed
  
  res.sendStatus( 200 )
})

app.listen( 8080 )
```

If you watched the Svelte tutorial, note that we serve our files from the `build` directory, which is where `create-react-app` places all files for deployment.

# React

## Setup
There are [lots of ways to start a React project](https://reactjs.org/docs/add-react-to-a-website.html). 
Alternatively, you can create a basic React project (using Babel to compile and Snowpack to bundle) by 
following [these instructions at createapp.dev](https://createapp.dev/snowpack). You can also use createapp.dev to generate Svelte apps, and add in additional 
features and libraries like Bootstrap, Typescript etc. We'll assume you've created a React app bundled with Snowpack for purposes of this tutorial. Also, make sure you run `npm install` in the project directory 
in order to get all needed libraries... you'll also need to install express separately. 

## Quick test
First, we require a basic file that loads our `App` component into a particular DOM element. Below is the `index.jsx` file given by createapp.dev; 
you shouldn't need to modify this file. Note that `.jsx` is a hybrid of JS / HTML and templating markup that won't run in the browser unless we compile
it first. Snowpack will handle that for us using the [Babel compiler](https://babeljs.io).

```js
import React from "react";
import ReactDOM from "react-dom";
import App from "./App";

var mountNode = document.getElementById("app");
ReactDOM.render( <App name="Jane" />, mountNode );
```

In the example above we've imported the `App` *component*, which we can then instantiate using HTML syntax. A component is an entity that (typically) has data,
behavior, and styling associated with it. In React the data and behavior for components go in a single file while CSS can be imported for styling. You'll note that
createapp.dev has already given you a `App.jsx` file; we'll edit this in a bit but for now let's compile our React app by running:

`npm run build`

This script (found in the `package.json` file) calls `snowpack build`, which will build your files and place them in the `build` directory. If you start 
the express server at the top of this handout and visit [http://127.0.0.1:8080](http://127.0.0.1:8080), then you should see our React project running. Great!

If you look at the `App.jsx` file you might be able to get a sense of what is happening. The App component is reading in any attributes we pass when 
we instantiate it in `index.jsx` and then assigning the values to `this.props`. Then we can use the React templating notation to easily interpolate those values
as needed. Try adding another attribute to the App element in `index.jsx`, and then see if you can get it displayed by changing `App.jsx`. Make sure you remember
to re-run `npm run build` after making changes.

## Making a TODO list application

Next let's edit our `App.jsx` file to work with our todo list express server. Again, this file is converted into valid JS by Snowpack/Babel, but React enables us to author
our components by freely mixing HTML and JS. 

```js
import React from 'react';

// we could place this Todo component in a separate file, but it's
// small enough to alternatively just include it in our App.js file.

class Todo extends React.Component {
  // our .render() method creates a block of HTML using the .jsx format
  render() {
    return <li>{this.props.name} : 
      <input type="checkbox" defaultChecked={this.props.completed} onChange={ e => this.change(e) }/>
    </li>
  }
  // call this method when the checkbox for this component is clicked
  change(e) {
    this.props.onclick( this.props.name, e.target.checked )
  }
}

// main component
class App extends React.Component {
  constructor( props ) {
    super( props )
    // initialize our state
    this.state = { todos:[] }
    this.load()
  }

  // load in our data from the server
  load() {
    fetch( '/read', { method:'get', 'no-cors':true })
      .then( response => response.json() )
      .then( json => {
         this.setState({ todos:json }) 
      })
  }

  // render component HTML using JSX 
  render() {
    return (
      <div className="App">
      <input type='text' /><button onClick={ e => this.add( e )}>add</button>
        <ul>
          { this.state.todos.map( (todo,i) => <Todo key={i} name={todo.name} completed={todo.completed} onclick={ this.toggle } /> ) }
       </ul> 
      </div>
    )
  }
}

export default App;
```

There's a lot of boilerplate to get started, but here we're creating two different components. 
The first is an item to represent a single Todo, while the second is our higher-level application. 
Note that we pass data from our app component to each todo via the `props` object, which lets us pass data through HTML attributes. So, in our example above, each `Todo` is created with a `name` attribute, and that attribute is subsequently available in the `props` object of the associated `Todo`. You can see we also assign/access the `.completed` and `.onclick` property of each `Todo` in the same way.

We also have introduced the `.state` property. In React, this is immutable, meaning you can't change values found inside of `.state`, you can only reset the state in its entirety. We to this using the `.setState` method of the `React.Component` superclass, and these changes in state are what effectivey trigger reactive changes in our UI.

OK, last but not least we add functions or updating our exising todos and adding new todos.

```js
  // when an Todo is toggled, send data to server
  toggle( name, completed ) {
    fetch( '/change', {
      method:'POST',
      body: JSON.stringify({ name, completed }),
      headers: { 'Content-Type': 'application/json' }
    })
  }
 
  // add a new todo list item
  add( evt ) {
    const value = document.querySelector('input').value

    fetch( '/add', { 
      method:'POST',
      body: JSON.stringify({ name:value, completed:false }),
      headers: { 'Content-Type': 'application/json' }
    })
    .then( response => response.json() )
    .then( json => {
       // changing state triggers reactive behaviors
       this.setState({ todos:json }) 
    })
  }
```
