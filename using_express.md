# Webware - Express

- What is Express?
  - Provides convenient "middleware" services to easily handle dataflow and server logic.
  - Extremely popular, with lots of plugins that make common web tasks easy.
  
- What is middleware?
  - A simple function with the signature `function( request, response, next ){}`, where
    `next` is a function that is called when the middleware function has finished processing
    data.
  - We call our middleware function with the `use` function of our express app e.g. `app.use( someMiddleWare )`

Below is a simple server-side logger of requested urls (adapted from express documentation). We can start by remixing a [express template project on glitch](https://webware-2022-express.glitch.me), and ignoring any HTML for now.

```js
const express = require( 'express' ),
      app = express()

const logger = (req,res,next) => {
  console.log( 'url:', req.url )
  next()
}

app.use( logger )

app.get( '/', ( req, res ) => res.send( 'Hello World!' ) )

app.listen( process.env.PORT || 3000 )
```

## Delvering static files
"Static" files on servers are files with no dynamic components; the server doesn't need to do any additional
processing on them before delivering them to clients. Express comes with the `express.static` middleware
function for delivering these files. In this project, you'll note there's two directories (`views,public`) containing files for the client. To create a simple server delivering these files we could use:

```js
const express = require( 'express' ),
      app = express()

app.use( express.static( 'public' ) )
app.use( express.static( 'views'  ) ) 

app.listen( process.env.PORT || 3000 )
```

... and done! We get a bunch of really nice features from this:

- all files delivered
- MIME types are correctly added for us
- automatically converts `/` urls to `./index.html`
- ... probably lots of other features I'm not thinking of at the moment

## Writing middleware to handle POST requests
Now that we've seen a bit of middleware in action, how would we write middleware to handle JSON information sent via POST request? We'll build off the GUI from an old Glitch example that enables users to post their dreams... however, we'll start by avoiding the GUI and just making our fetch request to the server from directly within the browser's development console.

```js
// server
const express = require('express'),
      app = express(),
      dreams = []

const middleware_post = ( req, res, next ) => {
  let dataString = ''

  req.on( 'data', function( data ) {
    dataString += data 
  })

  req.on( 'end', function() {
    const json = JSON.parse( dataString )
    dreams.push( json )

    // add a 'json' field to our request object
    // this field will be available in any additional
    // routes or middleware.
    req.json = JSON.stringify( dreams )

    // advance to next middleware or route
    next()
  })
}

app.use( middleware_post )

app.post( '/submit', ( req, res ) => {
  // our request object now has a 'json' field in it from our previous middleware
  res.writeHead( 200, { 'Content-Type': 'application/json'})
  res.end( req.json )
})

const listener = app.listen( process.env.PORT || 3000 )
```

Once you run this server in Glitch you'll see an error; we're not delivering any HTML files right now. 
Let's just test this file from the developer's console to make sure it works before we incoporate a GUI.

```js
// run in developer's console
fetch( '/submit', {
  method:  'POST',
  headers: { 'Content-Type': 'application/json' },
  body:    JSON.stringify(['test'])
})
.then( response => response.json() )
.then( console.log ) 
```

## But there's a middleware for grabbing  JSON, right?
Yes indeedy. `express.json()` will handle this nicely for us. There's a [list of other express middleware](https://expressjs.com/en/resources/middleware.html) that's worht checking out. *IMPORTANT:* For JSON data, the body-parser middleware will only take action if the data sent to the server is passed with a `Content-Type` header of `application/json`.

For example:
```js
fetch( '/submit', {
  method:  'POST',
  headers: { 'Content-Type': 'application/json' },
  body:    JSON.stringify( data )
})
```

Below is an entire example server, in a dozen lines of code not counting comments.

```js
const express    = require('express'),
      app        = express(),
      dreams     = []

app.use( express.static( 'public' ) )
app.use( express.static( 'views'  ) )
app.use( express.json() )

app.post( '/submit', (req, res) => {
  dreams.push( req.body.newdream )
  res.writeHead( 200, { 'Content-Type': 'application/json' })
  res.end( JSON.stringify( dreams ) )
})

app.listen( process.env.PORT )
```

We can also tell express to only use the `json()` middleware for a particular route,
which is usually a more efficient way to use it. To do this we simply
pass the middleware as the second argument to our route. You can do this
with any route/middleware combination... the `post`, `get`, `put`, `delete`
methods of the `app` object are all overloaded.
  
```js
app.post( '/submit', express.json(), ( req, res ) {
  dreams.push( req.body.newdream )
  res.writeHead( 200, { 'Content-Type': 'application/json'})
  res.end( JSON.stringify( dreams ) )
})
```
