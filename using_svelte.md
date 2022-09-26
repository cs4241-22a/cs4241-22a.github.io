# Server
The demo server for this example code has three routes to read, add, and update todos that are stored in memory. 

```js
const express  = require( 'express' ),
      app      = express()

const todos = [
  { name:'buy groceries', completed:false }
]

app.use( express.json() )
app.use( express.static( 'public' ) )

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

# Svelte
http://svelte.dev
- Uses code generation to create a small, optimized application
- Not made by Facebook(!)

### init a new svelte app
https://github.com/sveltejs/template

```
npx degit sveltejs/template svelte-app
cd svelte-app
```

## edit our App.svelte file
A file to load/display data from our server:

```html
<script>
  const getTodos = function() {
    const p = fetch( '/read', {
      method:'GET' 
    })
    .then( response => response.json() )
    .then( json => {
      console.log(json)
      return json 
    })
 
    return p
  }
  
  let promise = getTodos()
</script>
  
{#await promise then todos}
  <ul>

  {#each todos as todo}
    <li>{todo.name} : <input type='checkbox' todo={todo.name} checked={todo.completed}></li>
  {/each}

  </ul>
{/await}  
```

The part inside the `<script>` tag should look fairly normal. Outside of it is pretty weird looking though. The first line `{#await promise then todos}` basically is saying. Additionally, it's also saying "Everytime the promise resolves (re)create this list." Then, we start an unordered list `<ul>`. Next we're saying "for each todo in our todos variable, create a list item." We can see that we can insert JavaScript expressions inside of the `{}` characters, similar to how in ES6 we can put them expressions between `${ }` inside of template strings.
  
If you look at the `package.json` file included in the template, you'll see there's a `build` script. Use `npm run build` to compile the Svelte application (make sure you run `npm i` before compiling for the first time). You'll also need to install the `express` and `body-parser` packages, and then start the server using `node server.js`.
  
## adding new todos (reactive programming)
Here's where the magic happens:

```html
<script>
  const getTodos = function() {
    const p = fetch( '/read', {
      method:'GET' 
    })
    .then( response => response.json() )
    .then( json => {
      console.log(json)
      return json 
    })
 
    return p
  }

  const addTodo = function( e ) {
    const todo = document.querySelector('input').value
    promise = fetch( '/add', {
      method:'POST',
      body: JSON.stringify({ name:todo, completed:false }),
      headers: { 'Content-Type': 'application/json' }
    })
    .then( response => response.json() )
  }
  
  let promise = getTodos()
  </script>


<input type='text' />
<button on:click={addTodo}>add todo</button>

{#await promise then todos}
  <ul>

  {#each todos as todo}
    <li>{todo.name} : <input type='checkbox' todo={todo.name} checked={todo.completed} on:click={toggle}></li>
  {/each}

  </ul>
{/await}
```

In the above code we're adding the `addTodo` function and a button that triggers it. The magic is that, simply by redefining our `promise`. our list is recreated everytime we add a new todo. The UI is *reactive* to changes in the underlying data. OK, let's finish by adding a checkbox to toggle whether each todo item has been completed.


```html
<script>
  const getTodos = function() {
    const p = fetch( '/read', {
      method:'GET' 
    })
    .then( response => response.json() )
    .then( json => {
      console.log(json)
      return json 
    })
 
    return p
  }

  const addTodo = function( e ) {
    const todo = document.querySelector('input').value
    promise = fetch( '/add', {
      method:'POST',
      body: JSON.stringify({ name:todo, completed:false }),
      headers: { 'Content-Type': 'application/json' }
    })
    .then( response => response.json() )
  }

  const toggle = function( e ) {
    fetch( '/change', {
      method:'POST',
      body: JSON.stringify({ name:e.target.getAttribute('todo'), completed:e.target.checked }),
      headers: { 'Content-Type': 'application/json' }
    })
  }

  let promise = getTodos()
</script>

<input type='text' />
<button on:click={addTodo}>add todo</button>

{#await promise then todos}
  <ul>

  {#each todos as todo}
    <li>{todo.name} : <input type='checkbox' todo={todo.name} checked={todo.completed} on:click={toggle}></li>
  {/each}

  </ul>
{/await}
```

Here we're simply submitting a POST to request whenever each checkbox is checked/unchecked to update the data on the server. The UI is automatically changed via normal HTML behavior, however, the UI will also reflect changes to the data when the page is refreshed.
