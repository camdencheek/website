---
title: "Making of 'Social Distancing'"
date: 2020-04-19T15:12:54-04:00
math: true
markup: mmark
tags:
- generative
- python
---

One of the positive sides of being stuck at home all day is that I have much more time on my hands. Lately, I've been using some of that time to come back to my long-neglected interest in generative art. In honor of the pandemic, my first blog post documents the process behind my latest project: _Social Distancing_.

![Social Distancing](/images/social-distancing-1.svg)

# Overview
- Build a fully connected base graph
- Decompose the graph into cycles
- Remove any self-intersections
- Calculate a smooth path

[Source code](https://github.com/camdencheek/generative/tree/master/disjoint_cycles)

# Creating the base graph

The idea of the base graph is to represent all the _possible_ final edges between nodes before we factor it into cycles. In the above image, I started with a simple 20 by 20 grid where each node is connected to each of its direct neighbors (including diagonals).

For the graph-heavy code, I used python's [`networkx`](https://networkx.github.io/) library. In the code snippets, it's shortened to `nx`. I've included an example of generating a grid graph with diagonal connections.
```python
def graph_grid_with_diag(rows, cols):
    g = nx.Graph()
    for node in product(range(rows), range(cols)):
        g.add_node(node)
        for adj in adjacent_grid_with_diag(node, rows, cols):
            g.add_edge(node, adj)
    return g


def adjacent_grid_with_diag(node, rows, cols):
    for i in range(node[0]-1, node[0]+2):
        for j in range(node[1]-1, node[1]+2):
            if i >= 0 and i < rows and j >= 0 and j < cols:
                if i == node[0] and j == node[1]:
                    continue
                yield (i, j)
```

The choice of base graph can significantly change the characteristics of the final result. I've included a few examples to demonstrate. The first example is the graph type the original was generated from.

![Diagonal-connected graph](/images/debug-grid-diag-10.svg)
![Grid-connected graph](/images/debug-grid-nodiag-10.svg)
![Triangular-connected graph](/images/debug-tri-10.svg)

# Generate cycles from the graph

On my first iteration, I was just plotting random <a href="reddit.com">paths</a> through the base graph. After a little experimentation, I came up with a couple of requirements for pathing through the graph.
1) The set of paths needs to cover all nodes. The structure of the base graph is less prominent if we can just ignore a bunch of the nodes.
2) All paths need to be cycles. Dangling ends aren't pretty, and cycles are fillable.

It turns out my requirements have a more formal name: a [spanning 2-regular subgraph](https://en.wikipedia.org/wiki/Graph_factorization), also known as a 2-factor. Unfortunately, `networkx` did not provide an out-of-the-box algorithm for finding a k-factor of a graph. I [implemented one](https://github.com/networkx/networkx/pull/3925) based on the algorithm described in a paper[^1] I stumbled across. If you're interested in that kind of thing, I'd highly encourage you to read the paper. The algorithm is clearly described and fairly approachable.

Using that algorithm to generate cycles, we get something like this:

![Diagonal-connected graph lined and filled](/images/diagonal-cycles-10.svg)


Note: by default, this algorithm is deterministic. Some randomness can be added by randomly weighting the edges, then using those weights when finding the maximum matching in step 3 of the algorithm described in that paper. Conveniently, the `networkx` algorithms include [`max_weight_matching`](https://networkx.github.io/documentation/stable/reference/algorithms/generated/networkx.algorithms.matching.max_weight_matching.html), which allows us to choose a matching based on edge weights, allowing us to inject some randomness into the process.

# Remove any self-intersections

This style is interesting in its own right, and I had some fun playing around with it, but I decided I didn't like the self-intersections. Especially when smoothed (which I'll get to in a minute), it just looks a little out of place with the organic blobbiness I was going for.

I didn't generalize this algorithm to other types of graphs because, honestly, it seemed complicated. Instead, I went with a naive algorithm that just iterates through each square of four nodes and checks if both diagonals exist. If they do, we can safely assume that there are two opposing sides that do not have edges, and we can just delete the diagonals and add the opposing edges.

```python
def remove_intersections(g, w, h):
    for (i, j) in it.product(range(w-1), range(h-1)):
        n00 = (i, j)
        n01 = (i, j+1)
        n10 = (i+1, j)
        n11 = (i+1, j+1)

        if g.has_edge(n00, n11) and g.has_edge(n10, n01):
            g.remove_edge(n00, n11)
            g.remove_edge(n10, n01)
            if g.has_edge(n00, n01) or g.has_edge(n10, n11):
                g.add_edge(n00, n10)
                g.add_edge(n01, n11)
            else:
                g.add_edge(n00, n01)
                g.add_edge(n10, n11)
```

After removing edges from our sample graph, it looks like this:

![Diagonal-connected graph lined and filled](/images/diagonal-cycles-no-intersect-10.svg)

# Drawing it smoothly

There are a lot of ways to produce smooth lines between points. My personal favorite (and probably most common) is [cubic spline interpolation](https://en.wikipedia.org/wiki/Spline_interpolation). For image generation, they're especially nice because
1) They're equivalent to cubic Bézier curves
2) They math on them is _very_ convenient

Let me explain #2. For a cyclical set of nodes, we can create a set of cubic splines so that we have one spline between each of our nodes. Additionally, we can create those splines such that the first _and_ second derivatives of those splines are equal where the spline segments meet. This gives us that nice, smooth, organic shape.

So, how do we generate our cubic splines? Well, since I'm drawing them using [`pycairo`](https://pycairo.readthedocs.io/en/latest/), which supports cubic Bézier curves out of the box, we're going to start with the [formula](https://en.wikipedia.org/wiki/B%C3%A9zier_curve#Cubic_B%C3%A9zier_curves) for a cubic Bézier curve.

$$
B(t) = (1-t)^3P_0 + 3(1-t)^2tP_1 + 3(1-t)t^2P_2 + t^3P_3 \medspace , \medspace 0 \le t \le 1
$$

Note that$$P_0$$and$$P_3$$are the ends of the spline segment, and also the nodes in our graph we're drawing the spline between. Given$$B_n(t), 0 \le t \le 1$$is the spline between node$$n$$and$$n+1$$in our cycle, we get the following equations if we want[$$C^2$$](https://en.wikipedia.org/wiki/Smoothness#Order_of_continuity)continuity

$$
B_n(1) = B_{n+1}(0) \\
B_n'(1) = B_{n+1}'(0) \\
B_n''(1) = B_{n+1}''(0)
$$

Since it's 2020 and there's no need to differentiate these by hand, I'm going to use my good friend `sympy` to do the differentiation

```no-highlight
>>> from sympy import *
>>> t, p0, p1, p2, p3 = symbols("t p0 p1, p2 p3")
>>> b = (1-t)**3*p0 + 3*(1-t)**2*t*p1 + 3*(1-t)*t**2*p2 + t**3*p3
>>> b.subs(t,0)
p0
>>> b.subs(t,1)
p3
>>> diff(b,t).subs(t,0)
-3*p0 + 3*p1
>>> diff(b,t).subs(t,1)
-3*p2 + 3*p3
>>> diff(b,t,t).subs(t,0)
6*p0 - 12*p1 + 6*p2
>>> diff(b,t,t).subs(t,1)
6*p1 - 12*p2 + 6*p3
```

Substituting these into our equations above, we're left with a set of$$3k$$linear equations, where$$k$$is the number of nodes in our cycle.

$$
P_{3_n} = P_{0_{n+1}} \\
-3P_{2_n} + 3P_{3_n} = -3P_{0_{n+1}} + P_{1_{n+1}} \\
6P_{1_n} - 12P_{2_n} + 6P_{3_n} = 6P_{0_{n+1}} - 12P_{1_{n+1}} + 6P_{2_{n+1}}
$$

That first equation isn't very helpful since$$P_0$$and$$P_3$$are known in advance (they are our nodes). Additionally, in reality, each of these points represent two coordinates, so we can expand the four remaining equations into this:

$$
-3x_{2_n} + 3x_{3_n} = -3x_{0_{n+1}} + x_{1_{n+1}} \\
6x_{1_n} - 12x_{2_n} + 6x_{3_n} = 6x_{0_{n+1}} - 12x_{1_{n+1}} + 6x_{2_{n+1}} \\
-3y_{2_n} + 3y_{3_n} = -3y_{0_{n+1}} + y_{1_{n+1}} \\
6y_{1_n} - 12y_{2_n} + 6y_{3_n} = 6y_{0_{n+1}} - 12y_{1_{n+1}} + 6y_{2_{n+1}} \\
$$

Conveniently, for each spline, we now have four linear equations, and the four unknowns$$x_1, x_2, y_1, y_2.$$ I'm going to leave the solving of these linear equations to `numpy`. I'm not going to go into detail on constructing the matrix representation of the linear system, but if you're interested, you can take a look at the [`cycle_cubic_interpolate` function](https://github.com/camdencheek/generative/blob/adc37e2de39bc177c562f17146093ec0e76d8972/disjoint_cycles/graph.py#L50-L87).

We can use the points we solved for to draw each Bézier segment with `pycairo`'s [`context.curve_to()`](https://pycairo.readthedocs.io/en/latest/reference/context.html#cairo.Context.curve_to). Doing so yields the final style:

![Smooth interpolated](/images/smooth-final-10.svg)

All that's left from here is to scale it up to a 20x20 graph.

# Epilogue: Ideas for refinement and enhancement

- I think this could look really cool if the technique was applied to a base graph with radial symmetry. Circles make everything better.
- Currently, the cycle filling code doesn't take into account cycles contained in other cycles. The original has a couple of those. It would be neat if we could alternate white/black or something for layered fills.
- The style right now is very...stark. There is a lot of room to add color and texture
- The `max_weight_matching` algorithm in `networkx` is pretty slow. For my 20x20 graph, it took approximally 10 minutes to find the matching. The algorithm itself is pretty complex, but it might be possible to generate a spanning cycle graph without factoring another graph. Unsure if this is actually possible, but the complexity of the algorithm is a bottleneck for larger graphs right now.
- As suggested by [a Redditor](https://www.reddit.com/r/generative/comments/fzp2tp/social_distancing/fn5zrk8/) on my original post, it could be pretty sweet to 3D-ify this. Bézier surfaces and 3-factors exist, but it's not immediately clear what it might look like to generalize this to another dimension.

Other ideas or questions? Feel free to reach out. The easiest way would probably be on this [Reddit thread](asdf)


[^1]:  Meijer, Henk, Yurai Núñez-Rodríguez, and David Rappaport. "An algorithm for computing simple k-factors." Information processing letters 109.12 (2009): 620-625.
