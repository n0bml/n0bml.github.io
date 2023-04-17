---
layout: post
title: Mathematics and Markdown
---

As part of [Sharpening the Saw](https://www.franklincovey.com/habit-7/) I'm going to start writing about things I'm studying.  To get started I wanted to make a post with some mathematics notation.  It will be a good way to make sure all the pieces are in place.

First, one of my favorite equations: $$ y = mx + b $$.

Now for something a bit more complex:

$$
\begin{align}
x=\frac{-b\pm \sqrt{b^2-4ac}}{2a}
\end{align}
$$

Thanks to the hard work of the folks behind [MathJax](https://www.mathjax.org/)
it was very easy to add mathematical equations to my blog.

## Proof That One Equals Zero (Not-Really)

Given that $$a$$ and $$b$$ are integers such that $$a=b+1$$, prove that $$1=0$$.

$$
\begin{align}
a            &= b + 1                && \text{Given} \\
(a-b)a       &= (a-b)(b+1)           && \text{Multiplication Prop. of =} \\
a^2 - ab     &= ab + a - b^2 - b     && \text{Distributive Prop.} \\
a^2 - ab - a &= ab + a - a - b^2 - b && \text{Subtraction Prop. of =} \\
a(a-b-1)     &= b(a - b 1)           && \text{Distributive Prop.} \\
a            &= b                    && \text{Division Prop. of =} \\
b+1          &= b                    && \text{Transitive Prop. of =} \\
Therefore, 1 &= 0                    && \text{Subtraction Prop. of = } \\
\end{align}
$$
