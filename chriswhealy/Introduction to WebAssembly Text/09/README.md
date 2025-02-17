# Introduction to WebAssembly Text

| Previous | | Next
|---|---|---
| [Loops](../08/) | [Up](/chriswhealy/introduction-to-web-assembly-text) | [WASM and Shared Memory](../10/)

## 9: More About Functions

So far, we have only looked at a very simple WebAssembly function &mdash; one that takes no arguments and returns a single `i32` value.

> For the sake of simplicity, we will skip over certain details of function declarations such as interface types, as these are not required at the moment.

### Naming Functions

A function is declared using the keyword `func`.

> ***IMPORTANT***<br>
> WebAssembly functions can be given two, possibly different, human-readable names:[^1]
>
> 1. A name used when calling the function from **inside** the WebAssembly module
> 1. A name used when calling the function from **outside** the WebAssembly module

If we wish to call a WebAssembly function from inside the module, then the internal name is declared as follows:

```wast
(func $my_func       ;; The function's internal name

  ;; Function body goes here
)
```

If a WebAssembly function only needs to be called from outside the module, then the internal name can be omitted.  The external name is defined using the `export` keyword followed by some string that becomes the function's external (or exported) name:

```wast
(func
  (export "wasm_fn") ;; The function's external name

  ;; Function body goes here
)
```

It also might be the case that we need to call this function from both inside and outside the module.  In this case, specify both names:

```wast
(func $my_func       ;; The function's internal name
  (export "wasm_fn") ;; The function's external name

  ;; Function body goes here
)
```

> ***IMPORTANT***<br>
> It is entirely possible for the internal and external names of a function to be different!

### Function Arguments and Return Values

If a WebAssembly function needs to be passed any arguments, these must be specified immediately after the function name(s).  Similarly, if a function returns any values, these must be specified after any arguments have been defined.

Here's a real-life example.  Let's say we have a function that finds the magnitude (or hypotenuse length) of a complex number; that is, the distance from origin to the point on the complex plane.

This function takes a complex number as an argument in the form of two, 64-bit floating point numbers (`$real` and `$imag`) and returns a single 64-bit floating point number:

```wast
(func $mag           ;; Internal name
  (export "mag")     ;; External name
  (param $real f64)  ;; 1st argument is an f64 known as $real
  (param $imag f64)  ;; 2nd argument is an f64 known as $imag
  (result f64)       ;; One f64 will be left on the stack when the function terminates

  ;; Function body goes here
)
```

The implementation of this function simply uses Pythagoras' formula to work out the hypotenuse length of the right triangle having sides `$real` and `$imag`

[09-single-return-value.wat](/assets/chriswhealy/09-single-return-value.wat)
```wast
(func $mag           ;; Internal name
  (export "mag")     ;; External name
  (param $real f64)  ;; 1st argument is an f64 known as $real
  (param $imag f64)  ;; 2nd argument is an f64 known as $imag
  (result f64)       ;; One f64 will be left on the stack when the function terminates

  ;; Find the square root of the top value on the stack, then push the result
  ;; back onto the stack
  (f64.sqrt
    ;; Pop the top two values off the stack, add them up and push the result back
    ;; onto the stack
    (f64.add
      ;; Square the real part and push the result onto the stack
      (f64.mul (local.get $real) (local.get $real))

      ;; Square the imaginary part and push the result onto the stack
      (f64.mul (local.get $imag) (local.get $imag))
    )
  )

  ;; The f64.sqrt operation leaves a single f64 value on the stack; therefore, this
  ;; becomes the function's return value
)
```

After the `f64.sqrt` instruction has been executed, it pushes its result onto the stack and this implicitly becomes the function's return value.

You can run this function using `wasmer`:

```bash
wasmer 09-single-return-value.wat -i mag 3 4
5
```

### Functions with Multiple Return Values

WebAssembly functions can also return multiple values.  Here's a simple example in which we calculate the conjugate of complex number.  This is a very simple operation that transforms the complex number `a + bi` into `a - bi`.

The WebAssembly function is passed a complex number in the form of two, 64-bit floating point numbers, and it returns another complex number, also in the form of two, 64-bit floating point numbers.

[`09-multiple-return-values.wat`](/assets/chriswhealy/09-multiple-return-values.wat)
```wast
;; Conjugate of a complex number
;; conj(a+bi) => (a-bi)
(func $conj                     ;; Internal name
  (export "conj")               ;; External name
  (param $a f64)                ;; 1st argument is an f64 known as $a
  (param $b f64)                ;; 2nd argument is an f64 known as $b
  (result f64 f64)              ;; Two f64s will be left on the stack

  (local.get $a)                ;; Push $a. Stack = [$a]
  (f64.neg (local.get $b))      ;; Push $b then negate its value.  Stack = [-$b, $a]
)
```

After calling this function, the host environment pops these two values off the stack to obtain the result of the function call

You can test this by running using `wasmer` to run [`09-multiple-return-values.wat`](/assets/chriswhealy/09-multiple-return-values.wat)

```bash
wasmer 09-multiple-return-values.wat -i conj -- -5 3
-5 -3
```

> Notice the double hyphens `--` between `-i conj` and the function arguments.<br>
> This is necessary to prevent the shell from interpreting the minus sign in front of `-5` as a shell option

### NodeJS Limitation

Versions of NodeJS lower than 16 cannot invoke WebAssembly functions that return multiple values.  Earlier NodeJS versions will throw a runtime error if you attempt to instantiate a WebAssembly module that exports a function with multiple return values.

If you have Node 14 installed, try running [`09-multiple-return-values.js`](/assets/chriswhealy/09-multiple-return-values.js) from NodeJs.

```bash
node 09-multiple-return-values.js
(node:49384) UnhandledPromiseRejectionWarning: CompileError: WebAssembly.instantiate(): return count of 2 exceeds internal limit of 1 @+15
(Use `node --trace-warnings ...` to show where the warning was created)
```

In Node 16 or higher, this `.wasm` file runs fine as it will do in `wasmer` and from within a browser.

<hr>

[^1]: All functions (and local variables) are referenced by their index number, so you *could* choose not to use any human-readable function names; however, its now down to you to remember what function number `2` or `7` or `21` does.  Good luck with that one...
