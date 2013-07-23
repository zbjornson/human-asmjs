human-asmjs
===========

[asm.js](http://asmjs.org/spec/latest/) is an ahead-of-time (AOT) compiler for javascript. It's primarily intended as a target for compilers such as Emscripten, but it's possible for humans to write in asm.js as well. Unfortunately, there aren't many good examples and I ended up falling into a lot of pits from which I could only escape by trial-and-error. This readme contains some guidance to help others avoid these errors.

Contributions welcome. This document is based more on my observations than on my reading of the spec. Thus, there are probably errors in this document originating from my misunderstanding of the spec.

### Variables

<b>Basic Types</b>

Variables must be initialized to primitive types. They cannot be null, and they cannot be calculated:
```javascript
var a; // "Variable needs explicit type declaration via an initial value"
var a = (x/2)|0; // "Variable initialization value needs to be a numeric literal"
var a = 0; a = (x/2)|0; // works
```

To coerce a double to an int, you must use `~~`, as Vyacheslav Egorov points out [here](https://twitter.com/Zbjorn/status/336974148964343808). Trying to use the perhaps more obvious `|0` will fail with the message <i>Operands to bitwise ops must be intish</i>. Recent builds of Firefox Nightly further require `~~+floor(x)` or `~~+floor(x)|0`.

<b>Arrays</b>

Plain arrays, and by extension typed arrays created inline from plain arrays, are not allowed:

```javascript
var arr = [1,2,3]; // "Unsupported import expression"
```

```javascript
var arr = new stdlib.Int8Array([1,2,3]);// "Argument to typed array constructor must be ArrayBuffer name"
```

ArrayBuffers cannot be created in the module either:

```javascript
var ab = new stdlib.ArrayBuffer(4); // "Argument to typed array constructor must be ArrayBuffer name"
```

Instead, you have to use the heap (or another external arg) and views into typed arrays. Note that these typed arrays cannot be modified outside of a function:

```javascript
function MyModule(stdlib, foreign, heap) {
  var arr = new stdlib.Int8Array(heap);
  arr[0] = 1; // "asm.js must end with a return export statement"
  // ...
}
```

Instead do something like this:

```javascript
function MyModule(stdlib, foreign, heap) {
  var arr = new stdlib.Int8Array(heap);
  function init() {
    arr[0] = 1;
  }
  
  return {
    init: init
  };
}
```
