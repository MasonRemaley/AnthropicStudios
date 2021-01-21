---
layout: post
title: "Symmetric Matrices & Triangle Numbers"
date: 2020-03-30
tags: way-of-rhea tech
author: mason
excerpt_separator: <!--more-->
twitter: https://twitter.com/masonremaley/status/1245205608774520833
reddit: https://www.reddit.com/r/gamedev/comments/fst8i3/symmetric_matrices_triangle_numbers/
seo:
  links:
    - https://gamasutra.com/blogs/MasonRemaley/20200401/360417/Symmetric_Matrices__Triangle_Numbers.php
image: /assets/monsters-and-sprites/symmetric-matrices/player-orb.png
---

With Santa Cruz officially in lockdown due to COVID-19, I've been at home a lot working on [Way of Rhea](/way-of-rhea). Since I'm left without any excuse for not posting, here's a neat proof I happened across while working on some new puzzles. :)

## Symmetric Matrices

[Wikipedia](https://en.wikipedia.org/wiki/Symmetric_matrix) defines a symmetric matrix as "a square matrix that is equal to its transpose." In other words, a symmetric matrix has symmetry along its diagonal such that `m[row][col]` always equals `m[col][row]`.

Why should you care about symmetric matrices?

I dunno, you read the title and chose to click on this blog post, you tell me. **I'm** interested in symmetric matrices because this morning, as part of a puzzle I was working on, I added a layer system to Way of Rhea's physics engine.

The layer system essentially lets me define a number of layers rigid bodies can be placed on, and then for each layer-layer pair set a boolean that indicates whether or not collisions can occur between those two layers.

<figure style="position: relative">
  <a href="/assets/monsters-and-sprites/symmetric-matrices/player-orb.png">
    <img style="min-width: 100%; max-width: 100%; width: 100%" src="/assets/monsters-and-sprites/symmetric-matrices/player-orb.png" alt="player facing glowing orb">
    <figcaption style="
      position: absolute;
      bottom: 0;
      color: white;
      background: rgba(0, 0, 0, 0.5);
      width: 100%;
    ">
      <div style="margin: 5px">
        The pink orb should not collide with the player, but it should collide with the ground.
      </div>
    </figcaption>
  </a>
</figure>


<table style="display: table; font-size: 80%">
  <tr>
    <th><i>Layers</i></th>
    <th>Default</th>
    <th>Character</th>
    <th>Orb</th>
  </tr>
  <tr>
    <th>Default</th>
    <td markdown="span">`true`</td>
    <td markdown="span"></td>
    <td markdown="span"></td>
  </tr>
  <tr>
    <th>Character</th>
    <td markdown="span">`true`</td>
    <td markdown="span">`false`</td>
    <td markdown="span"></td>
  </tr>
  <tr>
    <th>Orb</th>
    <td markdown="span">`true`</td>
    <td markdown="span">`false`</td>
    <td markdown="span">`false`</td>
  </tr>
</table>

So for example, rigid bodies on `Layer::Default` can collide with rigid bodies on `Layer::Character`, but rigid bodies on `Layer::Character` cannot collide with rigid bodies on `Layer::Orb`. You'll notice that I didn't bother filling out the top right half of the matrix. That's because it's symmetric! I don't care whether an orb is intersecting with a character, or a character is intersecting with an orb; the result should be `false` either way.

In general, symmetric matrices can be used to create per-pair properties. Here's a couple other places I've had this come up:
* The [coefficient of restitution](https://en.wikipedia.org/wiki/Coefficient_of_restitution) is a property of object pairs, not single objects! It's pretty common to set it per object and then "combine" the two coefficients using some heuristic, which is totally fine most of the time, but if you want finer grained control for gameplay reasons you probably want a symmetric matrix!
* I believe that the same holds true for [friction coefficients](https://simple.wikipedia.org/wiki/Coefficient_of_friction). Regardless of the physical reality, it's again possible that you'll want fine grained control over this for gameplay reasons.

## Mapping to a Two Dimensional Array

The first two times this came up, I just hardcoded a giant match statement and kept the branches in sync manually. The third time it came up, with the layer system, I figured it was time to come up with a real solution.

At first I considered implementing symmetric matrices as 2D arrays. I figured I could make a triangular 2D array and swap the row/col indices when necessary to stay in the triangle, or I could just store each value twice.

This would have been fairly reasonable IMO, but something was nagging me...

<!--more-->

## Mapping to a One Dimensional Array

...it felt like there should be some way to map to a one dimensional array that didn't depend on the matrix's size, which if true, would be super neat! You'd just need to implement an index function, and then you could use any existing array type as a symmetric array with no additional bookkeeping. As a plus, if your underlying array type is growable, your matrix is too!

Well, as you probably guessed since I'm writing this blog post about it, it turns out that this is possible!

Consider the following numbering scheme, left to right top to bottom:

<table style="width:auto; display: table">
  <tr style="background-color: black">
    <th></th><th>0</th><th>1</th><th>2</th><th>3</th>
  </tr>
  <tr style="background-color: black">
    <th>0</th><td>0</td><td></td><td></td><td></td>
  </tr>
  <tr style="background-color: black">
    <th>1</th><td>1</td><td>2</td><td></td><td></td>
  </tr>
  <tr style="background-color: black">
    <th>2</th><td>3</td><td>4</td><td>5</td><td></td>
  </tr>
  <tr style="background-color: black">
    <th>3</th><td>6</td><td>7</td><td>8</td><td>9</td>
  </tr>
</table>

Our goal is to define a function that maps each row and column to the number in the corresponding cell. If the cell is unoccupied, we want to look at the cell opposite the diagonal from it (e.g. with the row and column swapped.)

For example, say that we want to calculate the index of **x**, at (2, 1). We can see above that this maps to 4, which is also the number of cells that occur prior to x in our ordering:
<table style="width:auto; display: table">
  <tr style="background-color: black">
    <th></th><th>0</th><th>1</th><th>2</th><th>3</th>
  </tr>
  <tr style="background-color: black">
    <th>0</th><td style="background:rgb(113, 190, 70)"></td><td></td><td></td><td></td>
  </tr>
  <tr style="background-color: black">
    <th>1</th><td style="background:rgb(113, 190, 70)"></td><td style="background:rgb(113, 190, 70)"></td><td></td><td></td>
  </tr>
  <tr style="background-color: black">
    <th>2</th><td style="background:rgb(113, 190, 70)"></td><td><b>x</b></td><td></td><td></td>
  </tr>
  <tr style="background-color: black">
    <th>3</th><td></td><td></td><td></td><td></td>
  </tr>
</table>

Above, I've highlighted the shape formed by the cells that come before **x** in our ordering. If we can come up with a formula for the area of that shape, we have a solution. It's a bit of an odd shape, so let's break it into two simpler pieces:
- The cells on **x**'s row leading up to it
- The triangle formed by the cells above **x**
<style type="text/css">
  #table-addition {
    display: flex;
    justify-content: center;
    align-items: center;
  }

  #table-addition table {
    width: auto;
    font-size: 50%;
    margin: 0;
  }

  #table-addition span {
    height: 100%;
    margin: 2em;
  }

  @media only screen and (max-width: 800px) {
    #table-addition {
      flex-direction: column;
    }
  }
</style>

<div id="table-addition">
  <table>
    <tr style="background-color: black">
      <th></th><th>0</th><th>1</th><th>2</th><th>3</th>
    </tr>
    <tr style="background-color: black">
      <th>0</th><td></td><td></td><td></td><td></td>
    </tr>
    <tr style="background-color: black">
      <th>1</th><td></td><td></td><td></td><td></td>
    </tr>
    <tr style="background-color: black">
      <th>2</th><td style="background:rgb(113, 190, 70)"></td><td><b>x</b></td><td></td><td></td>
    </tr>
    <tr style="background-color: black">
      <th>3</th><td></td><td></td><td></td><td></td>
    </tr>
  </table>

  <span>+</span>

  <table>
    <tr style="background-color: black">
      <th></th><th>0</th><th>1</th><th>2</th><th>3</th>
    </tr>
    <tr style="background-color: black">
      <th>0</th><td style="background:rgb(113, 190, 70)"></td><td></td><td></td><td></td>
    </tr>
    <tr style="background-color: black">
      <th>1</th><td style="background:rgb(113, 190, 70)"></td><td style="background:rgb(113, 190, 70)"></td><td></td><td></td>
    </tr>
    <tr style="background-color: black">
      <th>2</th><td></td><td><b>x</b></td><td></td><td></td>
    </tr>
    <tr style="background-color: black">
      <th>3</th><td></td><td></td><td></td><td></td>
    </tr>
  </table>

  <span>=</span>

  <table>
    <tr style="background-color: black">
      <th></th><th>0</th><th>1</th><th>2</th><th>3</th>
    </tr>
    <tr style="background-color: black">
      <th>0</th><td style="background:rgb(113, 190, 70)"></td><td></td><td></td><td></td>
    </tr>
    <tr style="background-color: black">
      <th>1</th><td style="background:rgb(113, 190, 70)"></td><td style="background:rgb(113, 190, 70)"></td><td></td><td></td>
    </tr>
    <tr style="background-color: black">
      <th>2</th><td style="background:rgb(113, 190, 70)"></td><td><b>x</b></td><td></td><td></td>
    </tr>
    <tr style="background-color: black">
      <th>3</th><td></td><td></td><td></td><td></td>
    </tr>
  </table>
</div>
<br>

The number of cells leading up to **x** on the same row as it is easy--that's just equal to **x**'s column, which we're given.

The triangle is a little trickier. Notice that regardless of the dimensions, it will **always** form an [equilateral triangle](https://en.wikipedia.org/wiki/Equilateral_triangle). If we can come up with an equation to calculate the number of cells required to form an equilateral triangle of a given size, then we're good to go!

## Triangle Numbers

Well, it turns out that this is called a [triangle number](https://en.wikipedia.org/wiki/Triangular_number). We could go try to dig through that Wikipedia link to see if it has the equation...but where's the fun in that? Let's derive it.

If we look at an example equilateral triangle, it's easy to see that we'll always take up half of the square it fits inside...sort of:
<table style="width:auto; display: table">
  <tr style="background-color: black">
    <td style="background:rgb(113, 190, 70);color:rgba(0, 0, 0, 0)">0</td><td></td><td></td>
  </tr>
  <tr style="background-color: black">
    <td style="background:rgb(113, 190, 70);color:rgba(0, 0, 0, 0)">1</td><td style="background:rgb(113, 190, 70);color:rgba(0, 0, 0, 0)">2</td><td></td>
  </tr>
  <tr style="background-color: black">
    <td style="background:rgb(113, 190, 70);color:rgba(0, 0, 0, 0)">3</td><td style="background:rgb(113, 190, 70);color:rgba(0, 0, 0, 0)">4</td><td style="background:rgb(113, 190, 70);color:rgba(0, 0, 0, 0)">5</td>
  </tr>
</table>

We have 9 cells in this square, half of that is...4.5. The answer we're actually looking for is 6. This is because we need to include the diagonal!

Alright, how can we do that?

Well, our square always has `n*n` cells, and our diagonal always has `n`. Let's just remove the diagonal from the square, halve that, and then add it back!

```
square = n * n
diagonal = n
triangle = (square - diagonal) / 2 + diagonal
```

With the substitutions made:
```
triangle = ((n * n - n) / 2) + n
```

Now that we have an equation, we can simplify it!

```
((n * n - n) / 2) + n
((n * n - n) / 2) + (2 * n / 2)
(n * n - n + 2 * n) / 2
(n * n + n) / 2
n * (n + 1) / 2
```

And there you have it, the equation for a triangle number! Feel free to plug in some numbers to see for yourself that it works.

## Putting it all together

We originally broke the shape whose area we're trying to calculate into two pieces, and we now have a format for both pieces separately: `row * (row + 1) / 2` and `col`. If we add those together, we get:

```
row * (row + 1) / 2 + col
```

Since the caller could pass in indices that point outside the triangle, we also need to flip the row and column if row is smaller than the column so that we always get a cell inside the triangle.

Here's my final implementation--note for those of you reading from the [Rust](https://www.rust-lang.org/) subreddit, this is written in a scripting language I wrote in Rust not Rust itself hence the Rust-like but sill unfamiliar syntax:

```rs
// Maps (row, col) or (col, row) indices into a symmetric matrix to a 1D index.
pub fn index(index_a: u16, index_b: u16) -> usize {
    // Get the low and high indices
    let low = ::math::usize_min(index_a as usize, index_b as usize);
    let high = ::math::usize_max(index_a as usize, index_b as usize);

    // Calculate the index (triangle number + offset into the row)
    let tri = high * (high + 1:usize) / 2:usize;
    let col = low;

    // Calculate the resulting index
    tri + col
}
```