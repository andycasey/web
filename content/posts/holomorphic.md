---
title: Holomorphic? Hell no!
description: "A short tale of gradients of complex functions in JAX."
date: 2025-07-03
authors:
  - name: Andy Casey
    link: https://astrowizici.st
    image: ../../me.png
tags:
  - jax
  - python
  - complex numbers
  - gradients
  - holomorphic
  - autodiff
---

This is a short post about a problem I hit while working with JAX and complex numbers. It's mostly a reminder to myself, but is niche enough that it might appear in Google searches and help someone else.

## The problem

I am using JAX to optimise a function $f: \mathbb{C} \to \mathbb{R}$ that takes a complex number as input and returns a real value.

I'm using JAX's automatic differentiation to compute the gradient of this function. Now when you're dealing with complex numbers and autodifferentiation (in JAX, and probably elsewhere) there are different notations for the gradient. The [JAX Autodiff cookbook](https://jax.readthedocs.io/en/latest/notebooks/autodiff_cookbook.html#Complex-Differentiation) is a great resource for this.

I thought my function was holomorphic, so I promised this to JAX by telling it to compute the gradient with `holomorphic=True`:
```python
grad = jax.grad(f, holomorphic=True)
```

Now this is just one part of my big problem that I am trying to optimise, so there were many other things that I could fiddle with. But through experimentation I was getting very strange results, and no amount of fiddling with other things would get me something sensible.

That was when I realised my problem was not [holomorphic](https://en.wikipedia.org/wiki/Holomorphic_function), but I had promised JAX it was. 

As a result, I was computing the wrong gradients.

If you're not sure if your problem is holomorphic or not, there is an easy workaround:
```python
from jaxtyping import Array, Complex, Float


# f_complex is my original function that takes a complex number
# and returns a real value. I thought it was holomorphic, but it isn't.
def f_complex(Z: Complex[Array]) -> Float:
    return ...
    
def f(Zw: Float[Array]) -> Float:
    # Zw has shape (2, ...): the real and imaginary parts
    return f_complex(Zw[0] + 1j * Zw[1])

g = jax.grad(f)

# If Z is a complex number, let's represent it as a 2D real array
Zw = jnp.array([jnp.real(Z), jnp.imag(Z)])

initial_loss = f(Zw)
initial_grad = g(Zw)
```

Don't assume holomorphicity! Holomorphism? Holomorphotics? We'll never know.