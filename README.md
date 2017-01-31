human-asmjs
===========

NOTE: (asm.js will be probably abandoned in favour of WebAssembly)

[asm.js](http://asmjs.org/spec/latest/) is an optimizable subset of javascript that is generally compiled ahead-of-time (AOT). It's primarily intended as a target for compilers such as Emscripten, but it's possible for humans to write in asm.js as well. Unfortunately, there aren't many good examples and I ended up falling into a lot of pits from which I could only escape by trial-and-error. This readme contains some guidance to help others avoid these errors.

Contributions welcome. This document is based more on my observations than on my reading of the spec. Thus, there are potentially errors in this document originating from my misunderstanding of the spec.

#### TOC

- [0. Overview](#0-overview)
- [1. Types](#1-types)
- [2. Arrays](#1-arrays)
- [3. Operators and functions](#3-operators-and-functions)
- [4. Syntax](#4-syntax)
- [5. Foreign function interface](#5-foreign-function-interface)
- [6. Common errors](#6-common-errors)
- [7. Resources](#7-resources)

## 0. Overview

An asm function module consists of the `"use asm"` declaration, shared variable declarations, function(s) definition(s) and exported functions.

```javascript
function asmModule(stdlib, foreign, heap) {
  "use asm";

  // shared variables
  var sin = stdlib.Math.sin;
  var heap32 = new stdlib.Float32Array(heap);
  var size = 0;

  // function declarations
  function complexCalculation(size, factor) {
    size = size|0;
    factor = +factor
    ...
  }

  // export function
  return { complexCalculation: complexCalculation }
}
```

Compiled asm modules are [linked](http://asmjs.org/spec/latest/#linking-0) for invocation from javascript.

## 1. Types

Asm.js handles three numeric types:

- int: 32 bit integer type 
- double: 64 bit floating point type
- float: 32 bit floating point type

Strings and other built-in objects are not allowed. Boolean value representing true / false uses int type instead of boolean type (1 and 0).

### 1.1 Type declaration

There are three types of declaration types for variables:

- Specifying the type of parameter of function
- Specifying the type of shared / local variable
- Specifying return type type of function

The method of describing each type is different.

| Type | Variable declaration | Function parameter | Return type |
| ---- | -------------------- | ------------------ | ----------- |
| Int  | `var i = 0;` | `i = i|0;` | `return i|0` |
| Double | `var d = 0.0` | `d = +d` | `return +d` |
| Float | `var f = fround(f)` | `f = fround(f)` | `return fround(f)` |

Notice that float requires to use the `Math.fround` function accessed via `stdlib`.

Variables must be initialized to primitive types. They cannot be null, and they cannot be calculated:
```javascript
var a; // "Variable needs explicit type declaration via an initial value"
var a = (x/2)|0; // "Variable initialization value needs to be a numeric literal"
var a = 0; a = (x/2)|0; // works
```

### 1.2 Type coercion

You can coerce one numeric type into another:


From / To | Int  | Double | Float |
--- | --- | --- | --- |
Int | _(none)_ | `+(a|0)` | `fround(a|0)` |
Double | `~~floor(a)` | _(none)_  | `fround(a)` |
Float | `~~floor(+a)` | `+a` |  _(none)_ |

```javascript
function asm(stdin, foreign, heap){
	"use asm";

	var fround = stdin.Math.fround;
	var floor = stdin.Math.floor;

	function typeConversion (){
		var i = 0;        //int
		var d = 0.0;      //double
		var f = fround(0);//float
		i = i;            //int to int
		i = ~~floor(+f);  //float to int
		i = ~~floor(d);   //double to int
		f = fround(i|0);  //int to float
		f = fround(d);    //double to float
		f = f;            //float to float
		d = +(i|0);       //int to double
		d = d;            //double to double
		d = +f;           //float to double
	}
}
```


## 2. Arrays

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
  'use asm';
  var arr = new stdlib.Int8Array(heap);
  arr[0] = 1; // "asm.js must end with a return export statement"
  // ...
}
```

Instead do something like this:

```javascript
function MyModule(stdlib, foreign, heap) {
  'use asm';
  var arr = new stdlib.Int8Array(heap);
  function init() {
    arr[0] = 1;
  }

  return {
    init: init
  };
}
```

### 2.1 Typed Arrays

| Name | Element type | Numeric type | bytes per index |
| ---- | ------------ | ------------ | --------------- |
| Int8Array | Signed 8 bit integer | Int | 1 byte |
| Uint8Array | unsigned 8 bit integer | Int | 1 byte |
| Int16Array | Signed 16 bit integer | Int | 2 bytes |
| Uint16Array | unsigned 16 bit integer | Int | 2 bytes |
| Int32Array | Signed 32 bit integer | Int | 4 bytes |
| Uint32Array | unsigned 32 bit integer | Int | 4 bytes |
| Float32Array	| 32bit floating point number | Float | 4 bytes |
| Float64Array	| 64bit floating point number | Double | 8 bytes |

When extracting the content of typed array to variable or using it for calculation, it is necessary to specify the correct type:

```javascript
// valid
i = int8Array[0]|0;
// invalid
i = int8Array[0];
```

asm.js uses byte addressing of the heap, so depending on what typed array are you using, you might need bitwise shifting:

```js
heap32[ndx >> 2] = fround(i|0);
```

## 3. Operators and functions

Only math related operations are allowed inside asm.js code. That means that logical operators like "||", "&&" or object operators like "new" or "instanceof" throws an Exception. The valid operators are:

| Type | Symbol(s) | Description |
| ---- | --------- | ----------- |
| Unary | + | Unary addition: type conversion to double |
|  | - | Sign inversion: type correction required  |
|  | ~ | Bitwise complement: applies to integer values, the result is an integer |
|  | ! | Negation of a comparation |
| Arithmetic | `+ - * / %` | Align types required in both sides. Multiplication can __not__ be used with integers |
| Bitwise | `| & ^ << >>` | The result is an integer value |
| | `>>>` | Rigth shift: the result is an unsigned integer |
| Comparasion | `< <= > >= == !=` | Align types required. Integer value (1 true, 0 false) returned |
| Condition | `a ? b : c` | Align types required |

To multiplicate int types you must use the `Math.imul` function.

If you use comparison operators, etc.", you need to align the type of the left term and the right term:

```javascript
var b = 0;
b = (i|0) == (i|0);
b = +(i|0) == +d;
b = fround(i|0) == f;
```
When combining the comparison results, use bit operators instead of logical operators.

When using conditional operators, align the types of the then and else terms:

```javascript
var b = 0;
b = (a|0 == 5) ? 3 : 4;
```

### 3.1 Arithmetic

As with the operator, asm.js can only use arithmetic functions related to numerical operations and numerical constants related to them:

| Functions | Type signature |
| --------- | -------------- |
| `acos, asin, atan, cos, sin, tan, exp, log` | `(double) => double` |
| `ceil, floor, sqrt` | `(double) => double` or `(float) => float` |
| `abs` | `(double) => double` or `(float) => float` or `(int) => int` |
| `min, max` | `(double, double, ...) => double` or `(int, int, ...) => int` |
| `atan2, pow` | `(double, double) => double` |
| `imul` | `(int, int) => int` (Used for multiplication of int type) |
| `fround` | `(numeric) => float` (Used for type coercion) |

| Constants | Type |
| --- | --- |
| `Infinity, NaN` | Double (special values) |
| `E, LN10, LN2, LOG2E, LOG10E, PI, SQRT1_2, SQRT` | Double |

## 4. Syntax

#### `if`

align left and right types

#### `switch`

conditional expression of a switch statement must be of type int. In addition, the label must be numeric literals:

```javascript
  var i = 0;
  switch(i|0){
    case 0:
    break;
    case 1:
    break;
    default:
    break;
  }
}
```

#### `for`

Variables must be decaled outside `for` statement. Can __not__ use `++` or `--` unary operators to increment or decrement:

```javascript
var i = 0;
for(i = 1; (i|0) < 10; i = (i+1)|0) {
  ...
}
```

#### `do-while`

Do-while and while syntax, continue statement, and labeled break statement can also be used in asm.js.

#### `return`

The value returned by the function is required to be of the same type no matter what path it follows.

### Syntax not available

- `for-in` (Objects and strings are not allowed)
- `try-catch-finally`
- Change the scope with `{}`
- Any ES6 syntax

## 5. Foreign function interface

You can call any function passed as property of the second argument when linking the asm.js module. However, data that can be delivered on that mechanism is limited to numerical values, and processing will be left to the world with slow operation.

When defining the foreign function, note the following points.

- The number of parameters, may be implemented without regard for the type.
Therefore, also works correctly by performing the processing corresponding to the number of parameters passed.
- Since the return value is a value that goes inside asm.js, only the numerical value in principle should be passed.
- asm.js not perform the control using the try-catch syntax, so use a status code to propagate the abnormal termination of the process.

## 6. Common errors

| Message (Firefox) | Description |
| ----------------- | ----------- |
| TypeError: asm.js type error: Disabled by debugger | You have to close the development tool and reload |
| TypeError: asm.js link error: ArrayBuffer byte Length of 0x10000 is less than 0x6000000 (the size of the size implied by const heap accesses)	| The size of the heap area (ArrayBuffer) is insufficient, so that ArrayBuffer of sufficient size is passed. |
| TypeError: asm.js type error: Math not found in module global scope |	Did not explicitly the affiliation of the Math object: `var imul = Math.imul;` should be `var imul = stdin.Math.imul;` |
| TypeError: asm.js type error: Uint8Array not found in module global scope	| Did not explicitly the affiliation of the constructor. |
| TypeError: asm.js type error: unsupported import expression	| Pobably you have to separate the initialization and modification of a variable |
| TypeError: asm.js type error: one arg to int multiply must be a small (-2 ^ 20, 2 ^ 20) int literal	 | An attempt was made the multiplication of int value. rewrite in Math.imul function. |
| TypeError: asm.js type error: intish is not a subtype of int |	(1) Miss the type of the variable: `values[a + 1]` should be `values[(a + 1)|0]`  (2) Untyped array contents: `a = values[0]` should be `a = values[0]|0` |
| TypeError: asm.js type error: unsupported expression | Look for `++` unary operations and variable declarations inside `for` statements |
| TypeError: asm.js type error: arguments to a comparison must both be signed, unsigned, floats or doubles; int and int are given	| Comparison is the left-hand side and the right side of the operator is incomparable, or type is to align the different. Type type cast or corrected. |
| TypeError: asm.js type error: incompatible number of arguments (1 here vs. 0 before) | The number of arguments at function definition does not match the number of arguments at function call. |
| TypeError: asm.js type error: unexpected statement kind	| Tried to use a unsupported syntax. |
| TypeError: asm.js type error: int is not a subtype of extern | Problems with variable types using the FFI |

# 7. Resources

- Lot of this information taken from: http://defghi1977.html.xdomain.jp/tech/asmjs/asmjs.htm
- asm.js working draft: http://asmjs.org/spec/latest/
