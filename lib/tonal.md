# Tonal

Tonal is the result of my journey of learning how to implement a music theory library in javascript in a functional way. I've learn a lot:

- I've learn the beauty of represent pitches using fifths and octave
- Stick with an idea can have troubles... Strings as notes. but what the hell! I still like it.
- It's easy to lost in the details
- It's easy to mess a source code by early optimizations

As an homenaje to Haskell School of Music.

## Music Theory for Javascript Programmers

Tonal music. Notes, pitch

### Prelude

Some utility functions:

```js
var isArr = Array.isArray
function isStr (s) { return typeof s === 'string' }
```

## 1. Pitches

Pitches are encoding using fifths and octaves. I've learn about it in the excellent javascript theory library [teoria] which takes the idea from

The pitch class will be encoded as an array with one integer: the number of fifths from `C` to the given pitch class:

```js
// the number of fifths to 'C D E F G A B'
var FIFTHS = [0, 2, 4, -1, 1, 3, 5]
```

If we add 7 fifths to a pitch class we add a sharp. If we subtract 7 fifths we add a flat:

```js
/**
 * Create a pitch class from its letter number
 * and it's alteration number
 *
 * @param {Integer} lnum - the letter num (0 is C, 1 is D...)
 * @param {Integer} alt - the alteration number
 * @return {Array} the pitch class in array notation
 * @example
 * pitchClass('Cb') // => [-7]
 */
export function pitchClass (lnum, alt) {
  return [FIFTHS[lnum || 0] + 7 * (alt || 0)]
}
```

A _pitch_ is a pitch class with an octave, so we are going to encode with a __two element array__ in the form `[fifths, octaves]`.

Since the fifths themselves may span one or more octaves we need to take account of it when building pitches:

```js
// given a number of fifths, return the octaves it spans
function fifthsOcts (f) { return Math.floor(f * 7 / 12) }
```

For example, although pitch class `D` is `[2]`, pitch `D0` is __not__ `[2, 0]` but `[2, -1]` since the 2 fifths spans 1 octave and we need to subtract it to stay in octave 0.

```js
/**
 * Build a pitch from letter number, alteration and octave. If
 * octave is not present, it builds a pitch class.
 *
 * @param {Integer} lnum - the letter number (0-based index)
 * @param {Integer} alt - the pitch accidentals integer
 * @param {Integer} oct - the pitch octave
 * @return {Array} the pitch in coord notation
 */
export function pitch (lnum, alt, oct) {
  var pc = pitchClass(lnum, alt)
  return !oct && oct !== 0
    ? pc : [pc[0], oct - fifthsOcts(pc[0])]
}
```

Our definition of pitch:

```js
/**
 * Test if a given object is a pitch
 *
 * @param {Object} p - the object to test
 * @return {Boolean} true if is a pitch array
 * @example
 * isPitch([3]) // => true
 */
export function isPitch (p) {
  return isArr(p) && (p.length === 2 || p.length === 1)
}
```

#### Convert strings to pitches

First we need to convert accidentals string to alteration number:

```js
/**
 * Convert accidental string to alteration number
 * @function
 * @param {String} acc - the accidental string
 * @return {Integer} the alteration number
 * @example
 * accToAlt('#') // => 1
 * accToAlt('bbb') // => -2
 * accToAlt('') // => 0
 * accToAlt('x') // => 2
 */
export function accToAlt (acc) {
  var alt = acc.replace(/x/g, '##').length
  return acc[0] === 'b' ? -alt : alt
}
```

And the opposite:

```js
/**
 * Convert alteration number to accidentals
 *
 * @function
 * @param {Integer} alt - the alteration number
 * @return {String} the accidentals string
 * @example
 * altToAcc(2) // => '##'
 * altToAcc(-2) // => 'bb'
 */
export const altToAcc = (alt) => Array(Math.abs(alt) + 1).join(alt < 0 ? 'b' : '#')
```

We need to convert from letter to letter index:

```js
export function letterIndex (letter) {
  var cc = letter.charCodeAt(0)
  return (cc > 96 ? cc - 92 : cc - 60) % 7
}
```

And with the power of regex:

```js
var PITCH_REGEX = /^([a-gA-G])(#{1,}|b{1,}|x{1,}|)(-?\d{0,1})$/
/**
 * Get the a regex to parse pitch in scientific notation
 *
 * After exec against a valid string we get:
 * - 0: the complete string
 * - 1: the letter (in upper or lower case)
 * - 2: the alterations (a list of #, b or x)
 * - 3: an optional octave number
 */
export function pitchRegex () { return PITCH_REGEX }
```

We get the complete picture:

```js
/**
 * Given a pitch string in scientific notation, get the pitch in array notation
 * @param {String} str - the string to parse
 * @return {Array} the pitch in array notation or null if not valid string
 * @example
 * pitchParse('C2') // => [2, 1]
 * pitchParse('bla') // => null
 */
export function pitchParse (str) {
  var m = PITCH_REGEX.exec(str)
  if (!m) return null
  var li = letterIndex(m[1])
  var alt = accToAlt(m[2])
  var oct = m[3] ? +m[3] : null
  return pitch(li, alt, oct)
}
```

### Pitch properties

Now we are going to write a collection of functions to get properties from pitches. Since `tonal` embraces the idea of string representing pitches, it's normal that those functions accepts strings as pitches. But also, accepting arrays as sources we open the possibility to use a different pitch notation (helmozt, for example) with the same function.

For this purpose we write the `tryPitch` function that parses notes in scientific notation, or returns the unaltered object if parse fails.

```js
function tryParser (parser) {
  return (obj) => isStr(obj) ? parser(obj) || obj : obj
}

/**
 * Given an object, try to parse as if it were a pitch in scientific notation. If success, return the parsed pitch, otherwise return the unmodified object.
 * @param {Object} obj - the object to parse
 * @return {Array|Object} the parsed pitch or the object if not valid pitch string
 * @example
 * tryPitch('G3') // => [1, 3]
 * tryPitch([1, 3]) // => [1, 3]
 * tryPitch(3) // => 2
 */
export const tryPitch = tryParser(pitchParse)
```

We write a function decorator that tries to parse first argument with `pitchParse` and return the parsed pitch or the unaltered argument. Functions decorated with `prop` accepts pitch is scientific notation or in array notation:

```js
/**
 * Decorate a function with one parameter to accepts
 * pitch in scientific notation
 * @param {Function} fn - the function to decorate
 * @return {Function} a function with one parameter that can be a pitch in scientific notation or anything else.
 */
export function prop (fn) {
  return (obj) => fn(tryPitch(obj))
}
```

With this decorator, we write our first property function:

```js
/**
 * Get alteration of a pitch.
 *
 * The alteration is an integer indicating the number of sharps or flats
 *
 * @param {Array|String} pitch - the pitch (either in scientific notation or array notation)
 * @return {Integer} the alteration
 * @example
 * alt('C#2') // => 2
 */
export const alt = prop((p) => Math.floor((p[0] + 1) / 7))
```

Our second property function will return the pitch letter:

```js
// get fifths simplified
function fifthsBase (num) {
  var i = (num + 1) % 7
  return i < 0 ? 7 + i : i
}

const LETTERS = 'FCGDAEB'
/**
 * Get the pitch letter
 *
 * @param {Array|String} pitch - the pitch (either in scientific notation or array notation)
 * @return {String} the letter
 * @example
 * letter('C#2') // => 'C'
 * letter([-7, 2]) // => 'C'
 */
export const letter = prop((p) => LETTERS[fifthsBase(p[0])])
```

```js
/**
 * Get accidental string from a pitch
 *
 * @function
 * @param {Array|String} pitch - the pitch to get the accidentals from
 * @return {String} the accidentals string
 * @example
 * accidentals('C##2') // => '##'
 * accidentals([-7]) // => 'b'
 */
export const accidentals = (p) => altToAcc(alt(p))
```

```js
// return if pitch has octave or not (is a pitch class)
export function hasOct (p) {
  return p && typeof p[1] !== 'undefined'
}
```

```js
const octOr = (v) => (p) => hasOct(p) ? p[1] + fifthsOcts(p[0]) : v
const octStr = octOr('')
const octNum = octOr(0)
export var oct = prop(octOr(null))
```

```js
/**
 * Convert a pitch in array notation to string
 *
 * @param {Array} pitch - the pitch to convert
 * @return {String} the pitch in scientific notation
 * @example
 * pitchStr([2, 1]) // => 'D2'
 */
export function pitchStr (p) {
  return letter(p) + accidentals(p) + octStr(p)
}
```
```js

export var sci = map([pitchStr, pitchParse])
sci.str = pitchStr
sci.parse = pitchParse
sci.regex = pitchRegex
```

## Intervals

```js
// build an interval
export function interval (num, alt, oct, dir) {
  return pitch(num, alt, oct).concat(dir)
}
```
```js
export function isInterval (i) { return isArr(i) && i.length === 3 }
```
```js

// get interval simplified number (0-based index)
var SIMPLES = [3, 0, 4, 1, 5, 2, 6]
export function simple (i) {
  return isArr(i) ? SIMPLES[fifthsBase(i[0])] : (i - 1) % 7
}

// get interval number from interval (1-based index)
export function number (i) {
  return simple(i) + 1 + 7 * octNum(i)
}

// get interval quality number
export function qNum (i) { return i[0] }

// get interval direction
export function direction (i) { return i[4] }

// get interval type (P = perfectable, M = majorable)
export function type (i) { return TYPES[simple(i)] }

var ALTER = {
  P: ['dddd', 'ddd', 'dd', 'd', 'P', 'A', 'AA', 'AAA', 'AAAA'],
  M: ['ddd', 'dd', 'd', 'm', 'M', 'A', 'AA', 'AAA', 'AAAA']
}
export function quality (i) {
  return ALTER[type(i)][4 + alt(i)]
}

// shorthand tonal notation (with quality after number)
var IVL_TNL = '([-+]?)(\\d+)(d{1,4}|m|M|P|A{1,4}|b{1,4}|#{1,4})'
// standard shorthand notation (with quality before number)
var IVL_STR = '(AA|A|P|M|m|d|dd)([-+]?)(\\d+)'
var COMPOSE = '(?:(' + IVL_TNL + ')|(' + IVL_STR + '))'
var IVL_REGEX = new RegExp('^' + COMPOSE + '$')

// get regexp to parse intervals
export function ivlRegex () { return IVL_REGEX }

var TYPES = 'PMMPPMM'
var QALT = {
  P: { dddd: -4, ddd: -3, dd: -2, d: -1, P: 0, A: 1, AA: 2, AAA: 3, AAAA: 4 },
  M: { ddd: -4, dd: -3, d: -2, m: -1, M: 0, A: 1, AA: 2, AAA: 3, AAAA: 4 }
}

// interval parse quality
function _ipq (simple, q) {
  return q === '' ? 0
    : q[0] === '#' ? q.length
    : q[0] === 'b' ? -q.length
    : QALT[TYPES[simple]][q]
}

export function ivlParse (str) {
  var m = IVL_REGEX.exec(str)
  if (!m) return null
  var num = +(m[3] || m[8])
  var sim = simple(num)
  var alt = _ipq(sim, m[4] || m[6] || '')
  var oct = Math.floor((num - 1) / 7)
  var dir = (m[2] || m[7]) === '-' ? -1 : 1
  return interval(sim, alt, oct, dir)
}

function dirStr (p) {
  return direction(p) === -1 ? '-' : ''
}

function ivlStr (p) {
  return dirStr(p) + number(p) + quality(p)
}

export var ivl = combine([ivlStr, ivlParse])
ivl.parse = ivlParse
ivl.str = ivlStr
ivl.regex = ivlRegex

/* ******** TRANSPOSE ******** */

export var tr = (a) => (b) => {
  var ac = sci.parse(a) || ivl.parse(a) || a
  var bc = sci.parse(b) || ivl.parse(b) || b
  var n = (isInterval(ac) && isPitch(bc)) ? trBy(ac, bc)
    : (isPitch(ac) && isInterval(bc)) ? trBy(bc, ac)
    : null
  return n ? sci.str(n) : null
}

export function trBy (ivl, p) {
  return p.length === 1 ? [ivl[2] * ivl[0] + p[0]]
    : [ivl[2] * ivl[0] + p[0], ivl[2] * ivl[1] + p[1]]
}
```

## Work with collections

Split strings:
```js

var SEP = /\s*\|\s*|\s*,\s*|\s+/
export function split (source) {
  return isArr(source) ? source
    : typeof source === 'string' ? source.trim().split(SEP)
    : (source === null || typeof source === 'undefined') ? []
    : [ source ]
}
```
```js
function combine (arr) {
  var fns = arr.slice().reverse()
  return function (e) {
    return fns.reduce(function (e, fn) {
      return fn(e)
    }, e)
  }
}
```
```js
export function map (fn, list) {
  if (arguments.length === 1) return function (l) { return map(fn, l) }
  if (isArr(fn)) fn = combine(fn)
  return split(list).map(fn)
}
```

```js
export function harmonize (list, tonic) {
  return split(list).map(tr(tonic))
}
```

Fin.