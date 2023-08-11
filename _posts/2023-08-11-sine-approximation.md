---
layout: post
title: Approximating a sine wave without sin()
tags: 
categories: 
---

If we want to make a sinusoidal signal, we need a sine wave. sin() is part of the standard C math library, but unfortunately this is not trivially available on all microcontrollers. I went looking for a suitable approximation, and there is a truly ancient one: [Bhāskara I's sine approximation formula](https://en.wikipedia.org/wiki/Bh%C4%81skara_I%27s_sine_approximation_formula), dated to 7th century CE. For phase angle x in turns (1 turn = 2 $ \pi $ radians = one full cycle), $ 0 \le x \le 1/2$, it is

$$
\sin(x) \approx \frac{32x(1-2x)}{5 - 8x(1-2x)}
$$

![error plot](https://upload.wikimedia.org/wikipedia/commons/9/96/BhaskaraSineApproximation1.png)

The error is mostly in the tenths of a percent, which is not too bad. Evaluating this requires two quadratics and a division, which ought to be quick on even minimal floating point hardware. Yes, a lookup table and interpolation might be a better answer (faster, more accurate) sometimes, but this is not too wrong and very easy. Thanks, Bhāskara!

A related question of approximating sine is how frequently we need to update the value of the output in order to have adequate smoothness. Typical standard for power quality require output [total harmonic distortion](https://en.wikipedia.org/wiki/Total_harmonic_distortion) of &lt 5%. I asked ChatGPT to do the math and simulations to give a result at 1% or better. It claimed 182 samples per cycle, or about 2 degrees per step. It sounds plausible.

![piecewise constant sine](/assets/piecewise-constant-182sample-sine.png)

The result doesn't look too bad either, but could be improved a bit by centering the steps. Note it's not just the total error which is relevant: also important is moving the errors up in frequency, so that they are easier to filter out in analog. The errors here are at 10 kHz and above, which is getting to the level of switching frequencies we'd be using. 
