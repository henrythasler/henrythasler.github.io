---
title: Heavy computing with GLSL - Part 2
subtitle: Emulated double precision
tags: [opengl, qt, shader]

---

In my last post I introduced a simple [mandelbrot fractal](http://en.wikipedia.org/wiki/Mandelbrot_set) shader with GLSL. ---Unfortunately--- Intentionally, the shader code uses single precision floating point variables which ensures great performance but limits the zoom factor to about 17 before the [lack of accuracy](http://en.wikipedia.org/wiki/Floating_point#Accuracy_problems) of the floating point variables takes over and all you get is a block-ish image of some sort at greater zoom levels:

![](/img/blog/Mandelbrot-App-simple-blocks.png)

Since I am very interested to discover the beauty of our mandelbrot set in close detail I will improve the existing shader with emulated double precision variables (aka: double-single) and see how far I can push it.

## Basics about precision emulation

Since I didn't find any GLSL code samples using emulated precision I decided to use the [DSFUN90 library](http://crd.lbl.gov/~dhbailey/mpdist/) by David H. Bailey which is written in [Fortran](http://en.wikipedia.org/wiki/Fortran). Fontran is no good for GPUs so I had to convert the parts I needed to GLSL.

Single precision floats can hold up to 8 digits and an exponent. Say, when you want to store the numer **<span style="color: #3366ff;">0.48881298</span><span style="color: #339966;">19481270</span>** in a single float variable you will get **<span style="color: #0000ff;">4.8881298</span><span style="color: #993300;">e-1</span>** (8 digits and an exponent). The remainder (the green part) is lost. On the other hand you can store the number **<span style="color: #808080;">0.00000000</span><span style="color: #339966;">19481270</span>** in a single float without having any trouble (**<span style="color: #339966;">1.9481270</span><span style="color: #993300;">e-9</span>**). Check out [this page](http://babbage.cs.qc.edu/IEEE-754/Decimal.html) to convert any number to their single or double precision counterparts and see what happens.

You may have figured out by now, that you can store the number **<span style="color: #3366ff;">0.48881298</span><span style="color: #339966;">19481270</span>** as the sum of **<span style="color: #0000ff;">4.8881298</span><span style="color: #993300;">e-1</span>** and **<span style="color: #339966;">1.9481270</span><span style="color: #993300;">e-9</span>** and that it is possible to store each of these two parts in a single floating point variable. So we just split the double precision value in two single precision variables as shown above and we are fine. Well, we're almost fine, since the functions to do the basic math stuff like addition or multiplication get a bit more complicated but that's the point where Mr. Bailey's library comes in and helps us out with his emulated double precision arithmetic.

## Main Application

Preparing the double precision variables before transferring the to our shader works as described above:

1. take a double (**<span style="color: #3366ff;">0.48881298</span><span style="color: #339966;">19481270</span>**) and convert it to single float (**<span style="color: #0000ff;">4.8881298</span><span style="color: #993300;">e-1</span>**). Store it.
2. convert the single float back to double (**<span style="color: #3366ff;">0.48881298</span><span style="color: #339966;">00000000</span>**) and subtract it from our original value.
3. Store the result (**<span style="color: #3366ff;">0.00000000</span><span style="color: #339966;">19481270</span>**)  **<span style="color: #3366ff;"> </span><span style="color: #339966;"> </span>** in the second float (**<span style="color: #339966;">1.9481270</span><span style="color: #993300;">e-9</span>**).

```glsl
    vec2[0] = (float)xpos;
    vec2[1] = xpos - (double)vec2[0];
    ShaderProgram->setUniformValue("ds_cx0",  vec2[0]);
    ShaderProgram->setUniformValue("ds_cx1",  vec2[1]);
```

The blue and green parts can be seen as High- and Low-Part of our emulated double value.

## Emulated arithmetics

The emulated double precision values (double-single) can be stored as vec2 in GLSL. This makes the code short and readability is improved (`vec2(ds_hi, ds_lo)`).

To evaluate our mandelbrot formula (`z = vec2(z.x*z.x - z.y*z.y, 2.0*z.x*z.y) + c`) and do the other stuff to create a cool looking image, we need the following arithmetics:

* Convert to/from emulated double precision (double-single)
* Addition, subtraction
* Multiplication
* Comparison

Conversion to double-single is easy since you just copy the value to the High part of the double-single (DS) variable.

```glsl
    vec2 ds_set(float a)
    {
    vec2 z;
    z.x = a;
    z.y = 0.0;
    return z;
    }
    vec2 ds_two = ds_set(2.0);
```    

To create a single float from our DS variable we just use the High-part and leave the (much smaller) Low-part out.
    float s_two = ds_two.x;
Addition is a bit more complex since you have to take care of some carry over from low to high part.

```glsl
    vec2 ds_add (vec2 dsa, vec2 dsb)
    {
    vec2 dsc;
    float t1, t2, e;

    t1 = dsa.x + dsb.x;
    e = t1 - dsa.x;
    t2 = ((dsb.x - e) + (dsa.x - (t1 - e))) + dsa.y + dsb.y;

    dsc.x = t1 + t2;
    dsc.y = t2 - (dsc.x - t1);
    return dsc;
    }
```

Multiplication is even more weird...

```glsl
    vec2 ds_mul (vec2 dsa, vec2 dsb)
    {
    vec2 dsc;
    float c11, c21, c2, e, t1, t2;
    float a1, a2, b1, b2, cona, conb, split = 8193.;

    cona = dsa.x * split;
    conb = dsb.x * split;
    a1 = cona - (cona - dsa.x);
    b1 = conb - (conb - dsb.x);
    a2 = dsa.x - a1;
    b2 = dsb.x - b1;

    c11 = dsa.x * dsb.x;
    c21 = a2 * b2 + (a2 * b1 + (a1 * b2 + (a1 * b1 - c11)));

    c2 = dsa.x * dsb.y + dsa.y * dsb.x;

    t1 = c11 + c2;
    e = t1 - c11;
    t2 = dsa.y * dsb.y + ((c2 - e) + (c11 - (t1 - e))) + c21;

    dsc.x = t1 + t2;
    dsc.y = t2 - (dsc.x - t1);

    return dsc;
    }
```

## Smooth coloring

I have also improved the coloring method to smooth, continuous coloring as described in this [post by Linas Vepstas](http://linas.org/art-gallery/escape/escape.html) or on [wikipedia](http://en.wikipedia.org/wiki/Mandelbrot_set#Continuous_.28smooth.29_coloring).

```glsl
    if(length(z) > radius)
        {
        return(float(n) + 1. - log(log(length(z)))/log(2.));
        }
```

Don't be scared of all that logarithm stuff. Most modern GPU can handle these very well.

## Result

Starting with our block example above the emulation shows excellent details for zoom levels up to 42 before the precision of our emulated doubles is spent:

![](/img/blog/Mandelbrot-App-emu.png)

## Performance

I get the following framerates in benchmark mode on my desktop ATI HD4870. They show that the emulation is round about 4 times slower than single precision mode. But still qualifies for realtime rendering.

![](/img/blog/Mandelbrot-App-emu-perf.png)

## Conclusion

Compared to single precision with a maximum resolution of ![](https://latex.codecogs.com/gif.latex?59\cdot10^{-9}) units in the complex plane per pixel the emulated doubles perform well up to ![](https://latex.codecogs.com/gif.latex?1.7\cdot10^{-15})
units per pixel.

Emulated double precision is a cool thing to do and works quite well on modern GPUs. Let's see if there can be done something to improve accuracy further...

## Qt Sourcecode

Sourcecode: [GLSL_EmuMandel.zip](/files/GLSL_EmuMandel.zip)
