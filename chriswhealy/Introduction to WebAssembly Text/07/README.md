# Introduction to WebAssembly Text

| Previous | | Next
|---|---|---
| [Arrangement of WAT Instructions](../06/) | [Up](/chriswhealy/introduction-to-web-assembly-text) | [Loops](../08/)

## 7: Conditions

WebAssembly includes the high-level flow control statements `if/then/else/end` for conditional branching.  By default, the `if` statement must complete without leaving any values behind on the stack.

### Simple Value Test

Using sequential notation, we can test if the local variable `$my_value` equals zero like this.   Place the `i32` value in question on top of the stack, then simply invoke the `if` statement:

```wast
local.get $my_value      ;; Stack = [5]

if
  ;; True if the top of the stack contains a non-zero i32
else
  ;; False if the top of the stack contains a zero i32
end
```

The point here is that any expression can be used to evaluate a condition as long as an `i32` value is left on the top of the stack.  The `if` statement then examines this `i32` value and if it is zero, this means `false`, and any non-zero value means `true`.

It's as simple as that.

We could also write the same condition as an S-expression.  But notice some important syntactical differences:

```wast
(if (local.get $my_value)
  (then
    ;; True if the top of the stack contains a non-zero i32
  )
  (else
    ;; False if the top of the stack contains a zero i32
  )
)
```

In sequential notation, the `then` keyword is not used but the `end` keyword is. However, if you fold an `if` statement into an S-expression then:

1. The entire `if` statement must be enclosed in parentheses
1. The `then` branch is mandatory, the `else` branch is optional
1. The `then` and `else` branches must be enclosed in parentheses
1. The `end` keyword is redundant now because its place has been taken by the closing parenthesis

### Simple Comparison Tests

Simply testing whether a local `i32` variable contains a non-zero value is not a very realistic use-case for `if`.  More realistically, we will need to compare two values: for instance, whether some counter has reached a particular limit.

The following code[^1] sample is part of a larger loop construct, but at the moment, the condition is the part that interests us:

```wast
(local $counter i32)

;; As long as the limit is greater than the counter
(if (i32.gt_u (i32.const 5) (local.get $counter))
  (then
    ;; True. Top of stack contains a non-zero i32
    ;; Do something here

    ;; Increment the counter
    (local.set $counter (i32.add (local.get $counter) (i32.const 1)))
  )
  (else
    ;; False. Top of the stack is a zero i32
  )
)
```

Remember that we have a choice over how we interpret integer values.

Consequently, all integer comparison statements must identify not only the type of comparison to be performed (`lt`, `gt`, `le`, `ge` etc), but must additionally specify whether the `i32` is to be treated as a signed or unsigned value &mdash; hence the comparison operations end with the additional suffix of `_u` or `_s` for unsigned or sign respectively.

In this case, it makes no sense to check whether we've gone round a loop a negative number of times, so there is no need to treat `$counter` as a signed value: hence we test for `i32.gt_u` (greater than, unsigned) as opposed to `i32.gt_s` (greater than, signed)

### Using `if` as an Expression

Up til now, the way we have used the `if` statement assumes that it does not leave a new value behind on the stack.

But what if we want to perform some sort of conditional assignment?

Consider this little block of JavaScript code that is actually used to optimize performance when calculating the Mandelbrot Set.  For some pixel location on the screen (given by `x` and `y`), we can avoid the expensive call to function `mjEscapeTime()` by checking whether this particular pixel is located within certain areas of the Mandelbrot Set.

```javascript
let iters = isInMainCardioid(x, y) || isInPeriod2Bulb(x,y)
            ? max_iters
            : mjEscapeTime(x, y)
```

The important point to understand here is that the value assigned to the variable `iters` is determined by the outcome of a condition.  Here, we are checking whether the current pixel at location `x` `y` falls within the Mandelbrot Set's main cardioid (the big heart-shaped blob in the centre) or within the period-2 bulb (the smaller circle to the left).  If it does, then we can bypass the expensive call to `mjEscapeTime()` and can arbitrarily set the value of `iters` to the maximum iteration value.

**Q**: That's nice, but how do we replicate this construct in WebAssembly?<br>
**A:** We can transform `if` from a *statement* into an *expression* by assigning it a return type

The implementation of functions `$is_in_main_cardioid` and `$is_in_period_2_bulb` is not important here, suffice it to say that these functions both return `i32` values that can be treated as Booleans.

```wast
;; Set $iters to whatever i32 value is returned from the if expression
(local.set $iters
  ;; The "(result i32)" clause declares that the if statement will leave an i32 value on the stack
  (if (result i32)
    ;; We can avoid running the escape time calculation if the point lies either
    ;; in the main cardioid or the period-2 bulb
    (i32.or
      ;; Main cardioid check returns an i32
      (call $is_in_main_cardioid (local.get $x) (local.get $y))

      ;; Period-2 bulb check returns an i32
      (call $is_in_period_2_bulb (local.get $x) (local.get $y))
    )
    (then
      ;; Yup, so no need to run the escape time algorithm
      (local.get $max_iters)
    )
    (else
      ;; Nope, we need to run the calculation
      ;; snip
    )
  )
)
```

Whatever `i32` value the `if` expression leaves on the stack, is then assigned to the local variable `$iters`.

<hr>

[^1]: From here on, we will always use the S-expression notation, because this is easier to read.
