# Socket Servers w/ Svelte (realtime chat)

## What are Web Sockets?
	- TCP and require a handshake to establish connection
	- realtime communication tech (chat, multiplayer games etc.)
	- remember to implement heartbeats to maintain connections
	- socket.io is a great library that abstracts sockets across programming languages, but they're easy to use in JS without additional libraries.

## A quick server for both http and WebSockets (ws)

[ws](https://www.npmjs.com/package/ws) - A node package for creating servers that understand the WebSocket protocol

```js
/* 
1. Open up a socket server
2. Maintain a list of clients connected to the socket server
3. When a client sends a message to the socket server, forward it to all
connected clients
*/

const express = require('express'),
      app     = express(),
      ws      = require('ws'),
      http    = require('http')

app.use( express.static('public') )

const server = http.createServer( app ),
      socketServer = new ws.Server({ server }),
      clients = []

socketServer.on( 'connection', client => {
    
  // when the server receives a message from this client...
  client.on( 'message', msg => {
	  // send msg to every client EXCEPT the one who originally sent it
    clients.forEach( c => if( c !== client ) c.send( msg ) )
  })

  // add client to client list
  clients.push( client )
})

server.listen( 3000 )
```

## Svelte Client
- Make template project using [svelte-template](https://github.com/sveltejs/template):

```
npx degit sveltejs/template svelte-app
cd svelte-app
npm i
```

Then edit your App.js component:

```js
// App.js
<script>
  let ws, msgs = []
  window.onload = function() {
    ws = new WebSocket( 'ws://127.0.0.1:3000' )
    // when connection is established...
    ws.onopen = () => {
		ws.send( 'a new client has connected.' )
      ws.onmessage = msg => {
	      // add message to end of msgs array,
	      // re-assign to trigger UI update
        msgs = msgs.concat([ 'them:' + msg.data ])
      }
    }
  }

  const send = function() {
    const txt = document.querySelector('input').value
    ws.send( txt )
    msgs = msgs.concat([ 'me:' + txt ])
  }
</script>

<input type='text' on:change={send} />

{#each msgs as msg }
  <h3>{msg}</h3>
{/each}
```

We can use the same server to make a simple “drawing” application. We’ll create a `<canvas>` object that fills the entire window, and then send the current mouse position whenever the window is clicked. We’ll then add code to take any message received and draw a box at the provided location.

```html
<html lang="en">
  <head>
    <style> 
		body { 
			margin:0; 
			background:black 
		} 
	  </style>
    <script>
      let ws, msgs = [], ctx = null
      
      window.onload = function() {
        ws = new WebSocket( 'ws://127.0.0.1:3000' )

        ws.onopen = () => {
          ws.onmessage = msg => {
            const [x,y] = msg.data
              .split(':')
              .map( v => parseInt(v) )

            ctx.fillStyle = 'red'
            ctx.fillRect( x,y,50,50 )
          }
        }

        const canvas = document.querySelector('canvas')
        canvas.width = window.innerWidth
        canvas.height = window.innerHeight
        ctx = canvas.getContext( '2d' )

        window.onclick = e => {
          ws.send( `${e.pageX}:${e.pageY}` )
          ctx.fillStyle = 'yellow'
          ctx.fillRect( e.pageX,e.pageY,50,50 )
        }
      }
    </script>
  </head>
  <body>
    <canvas></canvas>
  </body>
</html>
```
