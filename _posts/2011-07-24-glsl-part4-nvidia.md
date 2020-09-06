---
title: Heavy computing with GLSL
subtitle: NVIDIA optimizations
tags: [opengl, qt, shader]

---

In my recent post about double emulation in GLSL I tested the shader code on my ATI GPU. That was a mistake... Mauna pointed out that the double emulation shader does not work on NVIDIA GPUs. I checked that and it proved that this was not a single problem on a specific card.

I reckon that it must be some weird optimization on NVIDIA cards that break the emulation code. So I searched for a way to disable these optimizations. After some googling I found an [old blog entry by Cyril Crassin](http://www.icare3d.org/blog_techno/gpu/nvidia_glsl_compiling_options.html) (check out [his new blog](http://blog.icare3d.org/) if you are into 3D graphics) which helped me out with this problem.

## Solution

To turn off the NVIDIA optimizations I had to add the following lines to the shader code.

```glsl
    #pragma optionNV(fastmath off)
    #pragma optionNV(fastprecision off)
```

Now it should work fine with NVIDIA GPUs. Let me know if there are still some issues.

Download updated version: [GLSL_EmuMandel-nv.zip](/files/GLSL_EmuMandel-nv.zip)

I have also found a handy tool called [NVEmulate](http://developer.nvidia.com/nvemulate) to examine the GLSL compiler output and other stuff to analyze GLSL assembly on NVIDIA GPUs.

## Add-Ons

Jethro provided the division function:

```glsl
    vec2 ds_div(vec2 dsa, vec2 dsb)
    {
    vec2 dsc;
    float c11, c21, c2, e, s1, s2, t1, t2, t11, t12, t21, t22;
    float a1, a2, b1, b2, cona, conb, split = 8193.;

    s1 = dsa.x / dsb.x;

    cona = s1 * split;
    conb = dsb.x * split;
    a1 = cona – (cona – s1);
    b1 = conb – (conb – dsb.x);
    a2 = s1 – a1;
    b2 = dsb.x – b1;

    c11 = s1 * dsb.x;
    c21 = (((a1 * b1 – c11) + a1 * b2) + a2 * b1) + a2 * b2;
    c2 = s1 * dsb.y;

    t1 = c11 + c2;
    e = t1 – c11;
    t2 = ((c2 – e) + (c11 – (t1 – e))) + c21;

    t12 = t1 + t2;
    t22 = t2 – (t12 – t1);

    t11 = dsa[0] – t12;
    e = t11 – dsa[0];
    t21 = ((-t12 – e) + (dsa.x – (t11 – e))) + dsa.y – t22;

    s2 = (t11 + t21) / dsb.x;

    dsc.x = s1 + s2;
    dsc.y = s2 – (dsc.x – s1);

    return dsc;
    }
```    