# SVG

[Reference](https://developer.mozilla.org/en-US/docs/Web/SVG)

SVG is a simple alternative to `<canvas>` for creating (primarily) 2D image content in the browser. Unlike `<canvas>` the `<svg>` element creates *vector* graphics, which means images generated are crisp and sharp regardless of how they are scaled. This makes them ideally suited for print; create an SVG once and you can use it for your website, use it on a business card or use it to fill a poster.
  
The syntax for SVG is XML, which has a few subtle differences from HTML 5 but overall should look very familiar to everyone.

```html
<svg>
  <rect fill='red' x='0' y='0' width='50' height='50' />
</svg>
```

Note that in the above example, which draws a rectangle, the `<rect>` element is *self-terminating* (there's a forward slash at the end of it). In XML, you can't have tags without closing tags, while HTML is more forgiving. The self-terminating forward slash is an alternative to `<rect></rect>`. 

Note that we can create and append SVGs to our page *almost* the same way we would with any other HTML element. The only difference is we need to use a different *namespace* for SVG, so that our browser's rendering engine knows we're not creating HTML elements but instead following a different specification. We pass this namespace as an arugment to the `document.createElementNS()` function (instead of `document.createElement()`).

```js
const ns  = 'http://www.w3.org/2000/svg'
const svg = document.createElementNS( ns, 'svg' )
const rct = document.createElementNS( ns, 'rect' )
rct.setAttribute( 'fill', 'red' )
rct.setAttribute( 'x', 0 )
rct.setAttribute( 'y', 0 )
rct.setAttribute( 'width', 50 )
rct.setAttribute( 'height', 50 )

svg.appendChild( rct )
document.body.innerHTML = ''
document.body.appendChild( svg )
```

Note that the above code is quite a bit more text than our initial XML declaration! Soon we'll see that d3 has some useful shortcuts for handling SVG attributes. One other great thing about SVG is that, like HTML elements, you can control them using CSS.

```css
<style>
  rect{ width:50; height:50; fill:'red'; x:0; y:0; }
</style>
```

Now any rectangle we create in SVG will automatically have those default characteristics. We can also use `id` and `class` attributes to provide greater specificity. This also means that all *SVG attribtues are effectively CSS properties*, so you can pass values like `3em` for width and height, or `rgb(255,127,0)` for color.

There's a lot more you can do with SVG, please check out the [reference](https://developer.mozilla.org/en-US/docs/Web/SVG) for different shapes and attributes you can use.

# D3.js

[Climate visualization from The Economist](https://www.economist.com/leaders/2019/09/19/the-climate-issue?fsrc=scn/tw/te/bl/ed/theclimateissueawarmingworld)

[d3](http://d3js.org/) is a framework for "data driven documents". These often take the form of data visualizations, but d3 is actually more broadly applicable than this and is usable for general data processing. You use it to create and modify HTML or SVG nodes declaratively. That is, you give d3 a description of what you want to occur with your data, and it figures out how to do it. This can lead to some unexpected behaviors, but once you have the quirks down it's a very powerful plugin.

We'll use the following HTML for our examples, which loads the d3 library, a JS file where we'll place our code, and defines some basic CSS formatting.

```html
<!doctype html>
<html lang='en'>
  <head>
    <script src="https://d3js.org/d3.v5.min.js"></script>
    <script src='./main.js'></script>
    
    <style>
      body { 
        margin: 0; 
        background-color: black;
      }
    </style>
  </head>
  <body></body>
</html>
```

In our first example, let's display an array of four numbers.

```js
window.onload = function() {
 const data = [ 42,43,44,45 ]
 
 d3.select( 'body' )
   .data( data )
   .join( 'div' )
     .text( datapoint => 'num: ' + datapoint )
     .style( 'color', 'white' )
}
```

OK, so what's happening here? First, we use `.select` to get our `<body>` tag, in a way that should be familiar from using `document.querySelector()`. We then want to define how our document will look, so in this case we're saying something similar to the following:
  
  1. Using our `<body>` tag
  2. ...populate them with our data ( `.data()` )
  3. ...create an div per data point and "join" them to the data( `.join('div')` ) and 
  4. ...and format each as follows (.text(), .style())

It's a bit confusing why we need the `.selectAll()` call, you can [learn more about data joins](https://observablehq.com/@d3/selection-join) with some extremely cool yet terse examples.

## Load and display data
There's [lots of great JSON formatted data to play with on the web](https://github.com/jdorfman/Awesome-JSON-Datasets). Let's start with a really simple example that compares the current exchange rate of US dollars to other currencies. For each currency, we'll append a `<div>` displaying the country abbreviation and the exchange rate. We'll also change the background color to reflect the amount of change; this is actually somewhat meaningless in terms of the data but is meant to be used as an example.
  
```js
window.onload = function() {
  fetch( "https://api.exchangerate-api.com/v4/latest/USD" )
    .then( data => data.json() )
    .then( jsonData => {
      d3.select( 'body' )
        .data( d3.entries( jsonData.rates ) )
        .join( 'div' )
          .text( d => d.key + ' : ' + d.value )
          .style( 'background', d => `rgb( ${ Math.round( d.value )}, 0, 0 )` )
          .style( 'color', 'white' )
          .style( 'border-bottom', '1px gray solid' )
    })
}
```

Hopefully the `fetch()`/ JSON parsing looks familiar by now. Once we have our data, the main element we're interested in looking at is the `rates` object that it contains (you can view the returned JSON data in the Network tab of your Developer tools to see what the actual data looks like, or just visit the URL). The `d3.entries` function is going to take a JSON object and create an array with one object per key/value pair; each array object will feature a `.key` and a `.value` property. In the case of this particular data the key represents each country's abbreviation, while the value is the exchange rate relative to the USD. Next we use a function to define the text we want placed in each of our div, and then we apply styling. Note that it would probably make more sense to define a class in CSS and then apply the class using the [d3 `.classed` function](https://jaketrent.com/post/d3-class-operations/) as opposed to redefining it for every single `<div>` (look at the resulting HTML).

OK, let's use that same data and make something with an `<svg>` element.
  
## Circles and circles

In this visualization, overlapping circles will display the the currency comparison via color. Similar our last visualization, this use of color is only meant to be an example and is relatively meaningless.

To start, we create an `<svg>` element using the SVG namespace, append it to `document.body`, and use d3 to select the resulting element. Then we grab our data with fetch and teh same basic JSON parsing setup we used in our last example. This time, instead of creating `<divs>` using our join, we'll use `<circle>` elements. We then set the values of various attributes for each circle. Note that for the `cx` property, which defines the x position of the circle's center, we pass a function that accepts both a data point and the `index` of each element that is being processed; the index is what we use to position the circle sequentially from left to right.
  
```js
window.addEventListener( 'load', () => {
  const radius = 40,
        y = 50
  
  const svg = document.createElementNS( 'http://www.w3.org/2000/svg', 'svg' )
  svg.setAttribute( 'width', 1000 )

  document.body.appendChild( svg )

  fetch( "https://api.exchangerate-api.com/v4/latest/USD" )
    .then( data => data.json() )
    .then( jsonData => {
      d3.select( 'svg' ).selectAll('circle')
        .data( d3.entries( jsonData.rates ) )
        .join( 'circle' )
        .attr( 'fill', d => `rgba( ${ Math.floor(d.value) }, 100, 100, .5 )` )
        .attr( 'cx', (d,index) => index * radius )
        .attr( 'cy', y )
        .attr( 'r', radius )
    })
})
```

## Circles and circles and text

Last but not least, we want to add text as captions for each circle. There's no support in SVG to add text to elements such as `rect` and `circle`, instead we must place both our shape and the captioning text inside a group, tersely identified with `<g>`. There's no way to chain all of this together in one long expression, so we'll need to break things up a bit here. The circle part will be identical to our prior example, except that each circle will be appended to a group.

```js
window.addEventListener( 'load', () => {
  const radius = 40,
        y = 50

  const svg = document.createElementNS( 'http://www.w3.org/2000/svg', 'svg' )
  svg.setAttribute( 'width', 1000 )

  document.body.appendChild( svg )

  fetch( "https://api.exchangerate-api.com/v4/latest/USD" )
    .then( data => data.json() )
    .then( jsonData => {
      //console.log( d3.select('body').selectAll('div') )
      const group = d3.select( 'svg' ).selectAll( 'circle' )
        .data( d3.entries( jsonData.rates ) )
        .join( 'g' )

      group.append( 'circle')
        .attr( 'fill', d => `rgba( ${ Math.floor(d.value) }, 100, 100, .5 )` )
        .attr( 'cx', (d,i) => i * radius )
        .attr( 'cy', y )
        .attr( 'r', radius )

      group.append( 'text' )
        .text( d => d.key )
        .attr( 'fill', 'white' )
        .attr( 'x', (d,i) => i * radius - radius / 2 )
        .attr( 'y', y + radius + 25 )

    })
})
```

Really, there's not much difference here outside of the grouping. 
The process of creating the text is almost the same as before, except we're using the `.text()` method to provide 
the `.innerText` of each `<text>` element, and giving it a x and y offset relative to the associated circle.
