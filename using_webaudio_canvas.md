# Web Audio + Canvas

Let's make an audio visualizer. This will touch on using Canvas, the WebAudio API, Cross-Origin Resource Sharing (CORS), Typed Arrays in JavaScript, and more.

In A4, you'll have the opportunity to use these APIs in a creative sketch of your choosing (there will also be a technical option for those who prefer it).

# Web Audio / FFT visualization

The browser includes an audio API that is very capable. You can either assemble graphs of C++ DSP nodes or write your own DSP routines using JavaScript. Reference: https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API

You can run the example below directly in the console. However, when actually creating web pages, you need to create your `AudioContext` in response to user interaction. This is to avoid, for example, unscrupulous developers playing loud advertisements as soon as a site is loaded without any user permission. Putting everything below inside a `window.onclick` event handler is a simple solution for this.

```js
audioCtx = new AudioContext()
osc = audioCtx.createOscillator()
osc.connect( audioCtx.destination )

// now tell our osc oscillator to start running
// the 0 argument means start now, but we could also
// delay until some point in the future.
osc.start( 0 )

// we can change the frequency....
osc.frequency.value = 220

// we can also tell the oscillator to gradually ramp
// to a new frequency. Time is measured in seconds oscce
// the audio context was first created, to get a relative time value
// we can use the ctx.currentTime property

osc.frequency.linearRampToValueAtTime( 1760, audioCtx.currentTime + 30 )

// to stop...
// osc.stop()
```

We can change the `type` property of our oscillator to waveforms with more frequency content. Possible values include: `sine sawtooth triangle square`.

## Using gain nodes

Let's add another node to our graph to control the loudness of our oscillator. We can do this using a `GainNode`.

```js
audioCtx = new AudioContext()
osc  = audioCtx.createOscillator()
gainNode = audioCtx.createGain()

osc.connect( gainNode )
gainNode.connect( audioCtx.destination )

osc.start( 0 )

// try different values
gainNode.gain.value = .1

// osc.stop()
```

## Adding a filter
One more example, let's add a lowpass filter. This will attenuate the frequencies above a defined cutoff point in the signal.

```js
audioCtx = new AudioContext()
osc  = audioCtx.createOscillator()

// filters don't work well with sine oscillators...
osc.type = 'sawtooth'

gainNode = audioCtx.createGain()
gainNode.gain.value = .1

biquad   = audioCtx.createBiquadFilter()

osc.connect( biquad )
biquad.connect( gainNode )
gainNode.connect( audioCtx.destination )

osc.start( 0 )

// try different values... 110, 550, 1100, 4400 etc.
biquad.frequency.value = 110
```

## Filter sweep
Classic lowpass filter sweep, using the same `linearRampToValueAtTime` method we used to change our oscillator frequency earlier:

```js
audioCtx = new AudioContext()
osc  = audioCtx.createOscillator()

// filters don't work well with sine oscillators...
osc.type = 'sawtooth'

gainNode = audioCtx.createGain()
gainNode.gain.value = .1

biquad   = audioCtx.createBiquadFilter()

osc.connect( biquad )
biquad.connect( gainNode )
gainNode.connect( audioCtx.destination )

osc.start( 0 )
biquad.frequency.value = 110
biquad.frequency.linearRampToValueAtTime( 1760, audioCtx.currentTime + 2 )
biquad.frequency.linearRampToValueAtTime( 110,  audioCtx.currentTime + 4 )
// change the 'quality' of the filter, which emphasizes frequencies
// near the cutoff frequency... CAREFUL. THIS CAN BLOW UP YOUR EARZ.
biquad.Q.value = 20
```

# Audio + Visuals via FFT

## Basic FFT understanding
- The sine oscillator we heard earlier is the fundamental unit of almost all audio synthesis
- Joseph Fourier proved that any wave (not just sound) can be represented by sums of sine waves with different frequencies and amplitudes.
- So how can we determine which frequencies are present in a wave?
  - Turns out taking the dot product of the two signals (multiplying each point in time and adding the results)
    is a good indicator, see https://jackschaedler.github.io/circles-sines-signals/dotproduct3.html
- That means we just take the dot product of every frequency we're interested in.
- The WebAudio API does this by evenly dividing the available frequencies into *bins*
  - Sampling rate = 44100 samples per second... Nyquist frequency = 22050 Hz.
  - The *fftSize* determines how many samples are used to generate each FFT result. 
  - The *frequencyBinCount* is a read-only property that is always equal to fftSize / 2.
    This gives us the nunber of different values we can use for our frequency visualization.
    - the default fftSize is 1024, for 512 bins
  - 44100 / 512 = 86 Hz per bin
- The big tradeoff here is that in order to get better frequency resolution you need to increase the number of samples per window. This basically means as the frequency resolution increases, the temporal resolution (responsiveness) *decreases*. There's no real way to avoid this tradeoff.

## Getting an FFT with the Web Audio API

```js
// go ahead and reuse our previous code
// except lets make a much longer, wider frequency sweep
audioCtx = new AudioContext()
osc  = audioCtx.createOscillator()
osc.frequency.value = 80
osc.frequency.linearRampToValueAtTime( 880 * 4, audioCtx.currentTime + 30 )
osc.connect( audioCtx.destination )

const analyser = audioCtx.createAnalyser() // british spelling!

// set FFT size to make it easy to log, 
// note that 32 isn't large enough to do useful work
analyser.fftSize = 32
console.log( analyser.frequencyBinCount ) // > 16

// connect our sin oscillator to our analyser node
osc.connect( analyser )

// create a typed JS array to hold analysis results
const results = new Uint8Array( analyser.frequencyBinCount )

// every second, get our results and print them
const loop = setInterval( function() {
  // put analysis into our typed array
  analyser.getByteFrequencyData( results )
  console.log( results.toString() )
}, 1000 )
```

- Lovers of static types, rejoice... we have typed arrays for numbers, such as `Uint8Array`, `Float32Array` etc.
- In addition to be typed, these arrays are stored in contiguous memory, making successive lookup operations more efficient.
- Very useful for DSP work, computer vision, physics etc.

## Combine with Canvas

The [canvas element](https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API) lets us draw both 2D and 3D scenes in the browser. It's used for everything from games to displaying maps; here will use it to visualize our FFT analysis.

```js
document.body.innerHTML = ''
canvas = document.createElement( 'canvas' )
document.body.appendChild( canvas )

canvas.width = canvas.height = 512

// get our 2D drawing context, pass 'webgl' for most 3D drawing
ctx = canvas.getContext( '2d' )
audioCtx = new AudioContext()
osc  = audioCtx.createOscillator()
osc.type = 'square'
osc.frequency.value = 80
osc.frequency.linearRampToValueAtTime( 880 * 4, audioCtx.currentTime + 30 )
osc.start()
osc.connect( audioCtx.destination )

var analyser = audioCtx.createAnalyser()

analyser.fftSize = 1024 // 512 bins

// connect our sin oscillator to our analyser node
osc.connect( analyser )

// create a typed JS array to hold analysis results
var results = new Uint8Array( analyser.frequencyBinCount )

draw = function() {
  // temporal recursion, call tthe function in the future
  window.requestAnimationFrame( draw )
  
  ctx.fillStyle = 'black' 
  ctx.fillRect( 0,0,canvas.width,canvas.height )
  ctx.fillStyle = 'white' 
  
  analyser.getByteFrequencyData( results )
  
  for( let i = 0; i < analyser.frequencyBinCount; i++ ) {
    ctx.fillRect( i, 0, 1, results[i] ) // upside down
  }
}

// must call draw to start the loop
draw()
```

## Loading / visualizing audiofiles
We can create an audiofile visualizer the exact same way as above,
except instead of using an oscillator we're going to load an audiofile
using the `<audio>` element and the `MediaElementSource` node of the
Web Audio API. You can then connect that `MediaElementSource` node to
the analyser node like we did with the oscillator in our previous examples.

```js
ctx = new AudioContext()
audioElement = document.createElement( 'audio' )
document.body.appendChild( audioElement )
player = ctx.createMediaElementSource( audioElement )
player.connect( ctx.destination )
```
     
* To load a file, we set the `src` property on our audio element and
  then play the file using the play() method. However, there's a problem... CORS.
  
* ### CORS policy:
  * Can't load files from different URLs
  * Servers can selectively enable CORS.
  * You can turn it off in Chrome for particular sessions, for testing purposes
    http://www.thegeekstuff.com/2016/09/disable-same-origin-policy/
  * You can add a --cors flag with the http-server module for testing, or to use
    express middleware: https://medium.com/@alexishevia/using-cors-in-express-cac7e29b005b
    
Here's a super simmple server with CORS enabled:
```js
const express = require('express'),
      app     = express(),
      cors    = require('cors')

app.use( cors() )
app.use( express.static('./') )
app.listen( 3000 )
```
    
* Once we've set up CORS, now we can load files:
  
```js
audioElement.src = 'media/somefile.mp3'
audioElement.play()
```

One last gotcha to  note... if you're using Glitch to host your project, you need to change the [crossOrigin property](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/crossorigin) to load audiofiles from Glitch's asset server:

`audioElement.crossOrigin = 'anonymous'`

### All together now...

```html
<!doctype html>
<html lang='en'>
  <head></head>
  <body><button>click me</button></body>
  <script>
  const start = function() {
    document.body.innerHTML = ''
    const canvas = document.createElement( 'canvas' )
    document.body.appendChild( canvas )
    canvas.width = canvas.height = 512
    const ctx = canvas.getContext( '2d' )

    // audio init
    const audioCtx = new AudioContext()
    const audioElement = document.createElement( 'audio' )
    document.body.appendChild( audioElement )

    // audio graph setup
    const analyser = audioCtx.createAnalyser()
    analyser.fftSize = 1024 // 512 bins
    const player = audioCtx.createMediaElementSource( audioElement )
    player.connect( audioCtx.destination )
    player.connect( analyser )

    // make sure, for this example, that your audiofle is accesssible
    // from your server's root directory... here we assume the file is
    // in the ssame location as our index.html file
    audioElement.src = './trumpet.wav'
    audioElement.play()

    const results = new Uint8Array( analyser.frequencyBinCount )

    draw = function() {
      // temporal recursion, call tthe function in the future
      window.requestAnimationFrame( draw )
      
      // fill our canvas with a black box
      // by doing this every frame we 'clear' the canvas
      ctx.fillStyle = 'black' 
      ctx.fillRect( 0,0,canvas.width,canvas.height )
      
      // set the color to white for drawing our visuaization
      ctx.fillStyle = 'white' 
      
      analyser.getByteFrequencyData( results )
      
      for( let i = 0; i < analyser.frequencyBinCount; i++ ) {
        ctx.fillRect( i, 0, 1, results[i] ) // upside down!
      }
    }
    draw()
  }

  window.onload = ()=> document.querySelector('button').onclick = start
  </script>
</html>
```
