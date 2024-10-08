---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.16.2
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---

# Appendix: Background

## O notation

Throughout this chapter and the rest of the book, we will describe the
asymptotic behavior of a function using $O$ notation.

For two functions $f(t)$ and $g(t)$, we say that $f(t) \le O(g(t))$ if
$f$ is asymptotically upper bounded by $g$. Formally, this means that
there exists some constant $C > 0$ such that $f(t) \le C \cdot g(t)$ for
all $t$ past some point $t_0$.

We say $f(t) < o(g(t))$ if asymptotically $f$ grows strictly slower than
$g$. Formally, this means that for *any* scalar $C > 0$, there exists
some $t_0$ such that $f(t) \le C \cdot g(t)$ for all $t > t_0$.
Equivalently, we say $f(t) < o(g(t))$ if
$\lim_{t \to \infty} f(t)/g(t) = 0$.

$f(t) = \Theta(g(t))$ means that $f$ and $g$ grow at the same rate
asymptotically. That is, $f(t) \le O(g(t))$ and $g(t) \le O(f(t))$.

Finally, we use $f(t) \ge \Omega(g(t))$ to mean that $g(t) \le O(f(t))$,
and $f(t) > \omega(g(t))$ to mean that $g(t) < o(f(t))$.

We also use the notation $\tilde O(g(t))$ to hide logarithmic factors.
That is, $f(t) = \tilde O(g(t))$ if there exists some constant $C$ such
that $f(t) \le C \cdot g(t) \cdot \log^k(t)$ for some $k$ and all $t$.

Occasionally, we will also use $O(f(t))$ (or one of the other symbols)
as shorthand to manipulate function classes. For example, we might write
$O(f(t)) + O(g(t)) = O(f(t) + g(t))$ to mean that the sum of two
functions in $O(f(t))$ and $O(g(t))$ is in $O(f(t) + g(t))$.

## Python


