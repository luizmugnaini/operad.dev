+++
title = "2D Topological Quantum Field Theories and Commutative Frobenius Algebras"
date = 2023-03-03
updated = 2025-01-06

[taxonomies]
tags = ["mathematics"]

[extra]
math = true
+++

As a final project for my [mathematical-physics class](https://sites.google.com/view/cristian-ortiz/usp2022-math-physics)
I studied the categorical equivalence between 2-dimensional topological
quantum field theories (TQFT) and the category of commutative Frobenius algebras.
This led to an interesting study paper, which I'm now making publicly accessible.

{% math(caption="Universal property of monoidal natural transformations") %}
\begin{CD}
F(a \otimes b)               @>{\eta_{a \otimes b}}>>            G(a \otimes b) \cr
@V{\iso}VV                                                       @VV{\iso}V \cr
F a \widehat\otimes F b      @>>{\eta_a \widehat\otimes \eta_b}> G a \widehat\otimes G b
\end{CD}
{% end %}


<!-- more -->

The paper is divided into 9 sections:

1. **Monoidal categories**: This section serves as the foundation language of the
   paper, monoidal categories and their properties.
2. **Braided & symmetric monoidal categories**: Here, we briefly delve into a
   special class of monoidal categories that permit commutative behaviour with
   respect to the associated bifunctor product.
3. **Cobordisms**: A central idea in topological quantum field theories is the
   gluing of smooth compact manifolds without boundary, which is what cobordisms
   are all about.
4. **Elements of Morse theory**: We provide a very introductory discussion of Morse theory,
   just showing some key results required for the rest of the paper.
5. **The category $n$-$\cat{cob}$**: One of the building blocks of the main theorem of
   the paper is the category of cobordisms of a given dimension $n$,
   made up of compact oriented $(n - 1)$-dimensional manifolds.
6. **Monoidal structure of $n$-$\cat{cob}$**: As expected, the category of cobordisms
   exhibits some good properties, and in particular it admits a monoidal
   structure.
7. **Frobenius Algebras**: Within the framework of monoidal categories, we study
   our second main character: Frobenius algebras.
8. **Topological quantum field theories**: In this section we can finally define
   what we mean by a TQFT.
9. **Equivalence theorems**: Reaching the climax of the paper, we delve into the
   theorems that motivated its creation in the first place.

The full paper is [available as a PDF](/2d-tqft-frobenius.pdf).
