| Previous | | Next
|---|---|---
| [2: Initial Implementation](../../02%20Initial%20Implementation/) | [Top](/chriswhealy/plotting-fractals-in-webassembly) | [4: Optimised WAT Implementation](../../04%20WAT%20Optimised%20Implementation/)
| [3.4: Escape-Time Algorithm](../04/) | [3: Basic WAT Implementation](../) | [3.6: Displaying the Rendered Fractal Image](../06/)

## 3.5: Calculating the Mandelbrot Set Image

Now that we have a bare-bones function to calculate the value of a single pixel, we can simply call this function for every pixel in the image.

```wast
;; - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
;; Plot Mandelbrot set
(func (export "mandel_plot")
      (param $width i32)     ;; Canvas width
      (param $height i32)    ;; Canvas height
      (param $origin_x f64)  ;; X origin coordinate
      (param $origin_y f64)  ;; Y origin coordinate
      (param $ppu i32)       ;; Pixels per unit (zoom level)
      (param $max_iters i32) ;; Maximum iteration count

  (local $x_pos i32)
  (local $y_pos i32)
  (local $cx f64)
  (local $cy f64)
  (local $cx_int f64)
  (local $cy_int f64)
  (local $pixel_offset i32)
  (local $pixel_val i32)
  (local $ppu_f64 f64)
  (local $half_width f64)
  (local $half_height f64)

  (local.set $half_width (f64.convert_i32_u (i32.shr_u (local.get $width) (i32.const 1))))
  (local.set $half_height (f64.convert_i32_u (i32.shr_u (local.get $height) (i32.const 1))))

  (local.set $pixel_offset (global.get $image_offset))
  (local.set $ppu_f64 (f64.convert_i32_u (local.get $ppu)))

  ;; Intermediate X and Y coords based on static values
  ;; $origin - ($half_dimension / $ppu)
  (local.set $cx_int (f64.sub (local.get $origin_x) (f64.div (local.get $half_width) (local.get $ppu_f64))))
  (local.set $cy_int (f64.sub (local.get $origin_y) (f64.div (local.get $half_height) (local.get $ppu_f64))))

  (loop $rows
    ;; Continue plotting rows?
    (if (i32.gt_u (local.get $height) (local.get $y_pos))
      (then
        ;; Translate y position to y coordinate
        (local.set $cy
          (f64.add
            (local.get $cy_int)
            (f64.div (f64.convert_i32_u (local.get $y_pos)) (local.get $ppu_f64))
          )
        )

        (loop $cols
        ;; Continue plotting columns?
          (if (i32.gt_u (local.get $width) (local.get $x_pos))
            (then
              ;; Translate x position to x coordinate
              (local.set $cx
                (f64.add
                  (local.get $cx_int)
                  (f64.div (f64.convert_i32_u (local.get $x_pos)) (local.get $ppu_f64))
                )
              )

              ;; Store the current pixel's colour using the value returned from the following if expression
              (i32.store
                (local.get $pixel_offset)
                (if (result i32)
                  ;; Does the current pixel hit max_iters?
                  (i32.eq
                    (local.get $max_iters)
                    ;; Calculate the current pixel's iteration value and store in $pixel_val
                    (local.tee $pixel_val
                      (call $escape_time_mj
                        (local.get $cx) (local.get $cy)
                        (f64.const 0) (f64.const 0)
                        (local.get $max_iters)
                      )
                    )
                  )
                  ;; Yup, so return black
                  (then (global.get $BLACK))
                  ;; Nope, so return whatever colour corresponds to this iteration value
                  (else
                    ;; Push the relevant colour from the palette onto the stack
                    (i32.load
                      (i32.add (global.get $palette_offset) (i32.shl (local.get $pixel_val) (i32.const 2)))
                    )
                  )
                )
              )

              ;; Increment column and memory offset counters
              (local.set $x_pos (i32.add (local.get $x_pos) (i32.const 1)))
              (local.set $pixel_offset (i32.add (local.get $pixel_offset) (i32.const 4)))

              br $cols
            )
          )
        ) ;; end of $cols loop

        ;; Reset column counter and increment row counter
        (local.set $x_pos (i32.const 0))
        (local.set $y_pos (i32.add (local.get $y_pos) (i32.const 1)))

        br $rows
      )
    )
  ) ;; end of $rows loop
)
```

When looking through the coding, the following points are important:

1. The `mandel_plot` function is not called from anywhere inside the WebAssembly module; therefore, it does not need an internal name, only an exported name
1. This function writes to shared memory; therefore, it has no need for a `result` clause
1. Since this function works using two different frames of reference (pixels within the image, and coordinates on the complex plane), it needs to know:
    1.  How to transform an image pixel location to a corresponding coordinate in the complex plane.  Therefore, to link these two frames of reference, we must supply the arguments `$origin_x` and `$origin_y` that identify the X and Y coordinates of the central pixel in the image
    1. At what zoom level is the image being rendered?  To answer this question, we must supply the argument `$ppu` (or pixels per unit).  To start with, each unit on the complex plane will subdivided into 200 pixels (`$ppu = 200`) since this is small enough to show the entire Mandelbrot set on a canvas that is 800 by 450 in size.
1. The basic structure of the WAT function follows the structure used in the JavaScript implementation; that is, a pair of nested loops.
1. Certain optimisations have been implemented in order to avoid calculating the same value multiple times.  Hence the need for intermediate values such as `$half_width`, `$half_height`, `$cx_int` and `$cy_int`
1. It is very important to remember that the `$x_pos` and `$y_pos` loop counters are integers, but the escape-time algorithm requires coordinate values.  This means two things:
    1. Each index value must be transformed from a pixel location on the image, to a coordinate in the complex plane
    1. Coordinates are floating point numbers, not integers.  Therefore, we must employ type-conversion instructions (such as `f64.convert_i32_u`) to convert unsigned 32-bit integers into 64-bit floating points.

        This also explains why although the function receives argument `$ppu` as an `i32`, before it can be used, that value must first be converted and stored as an `f64` local variable.
1. Just before the call to `$escape_time_mj`, we use the instruction `i32.tee`.  This instruction is useful here because it does two things at once:
    1. It stores a value in a local variable (in this case `$pixel_val`), and
    1. It leaves the stored value on the top of the stack, thus saving us from needing to perform a `local.set ...` followed by a `local.get ...`
