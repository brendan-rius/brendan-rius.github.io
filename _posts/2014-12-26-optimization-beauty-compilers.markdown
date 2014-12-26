---
layout: post
title:  "Optimization, the beauty of compilers"
date:   2014-12-26 02:10:23
categories: c compilers optimization
---

A few months ago I was asked for an exercise to create a C code to swap the
endianess (i.e. the order of the bytes in a word) of a 32 bits int representing
an ARGB color.
Nothing that much complicated. While the natural way of doing this would be
to write something like:

{% highlight c %}
uint32_t swap_endianess(uint32_t value)
{
  uint32_t r, g, b, a;

  /* We get each byte by isolating him, and masking others */
  r = (value >> 24) & 0xFF;
  g = (value >> 16) & 0xFF;
  b = (value >>  8) & 0xFF;
  a =  value        & 0xFF;

  return (a << 24) | (b << 16) | (g << 8) | r;
}
{% endhighlight %}

the exercise added a constrainst: I had to use a union. For those who are
not familiar with the concept of union, a union is a data structure that
represents a location in the memory that can be filled by multiple types
of data (thus taking the size of the biggest data type). For example:

{% highlight c %}
union	useless_union
{
  int	int_data;
  char	char_data[4];
}
{% endhighlight %}

Here, `int_data` and `char_data` share the same location in memory, and thus
the same address. This way, is the int is 4 bytes long (which is typically
the case on all modern computers), you can access the n-th byte of the int
with `char_data[n]`.

The solution to this exercise was something ugly like:

{% highlight c %}
union	swapping_union
{
  uint32_t	int_data;
  char		char_data[4];
};

uint32_t	swap_endianess(uint32_t color)
{
  union swapping_union	u_col;
  char			tmp;

  /* We fill the memory with the int */
  u_col.int_data = color;

  /*
  ** We the leftmost and the rightmost bytes (respectively char_data[0] and
  ** char_data[3], because the array share the same location than the int.
  */
  tmp = u_col.char_data[0];
  u_col.char_data[0] = u_col.char_data[3];
  u_col.char_data[3] = tmp;

  /* We repeat the proccess with the two "middle" bytes */
  tmp = u_col.char_data[1];
  u_col.char_data[1] = u_col.char_data[2];
  u_col.char_data[2] = tmp;

  /* We return the new "int" */
  return (u_col.int_data);
}
{% endhighlight %}

Although we do not clearly identify what the code is supposed to fo,
this method has the advantage of performing less operations (5 assignements only)
than the first one (6 bitshifts, 4 ANDs and 3 ORs).

After having performed some benchmarks on my i7 with no compiler optimization, swapping
endianess of 2,000,000,000 random integers takes ~1.04 seconds with the first method and
~1.72 seconds with the second one

While you can see the first way is a little bit fastest, the best is yet to come:
compiling with optimization (`-O2` with gcc), makes the code runs super-fast.

If we take a look at the compiled code of the first method we will see

{% highlight asm %}
mov	%edi, %eax
bswap	%eax
{% endhighlight %}

the `bswap` instruction here is a processor builtin operation to reverse the byte
order of a 32 bit register (which contains our value): *the compiler understood
what the code was doing, and did it in a much better way*: swapping 2,000,000,000
integers now take less than one clock tick (less thans 0.000001 second).

Meanwhile, the second method is so exotic that the compiler cannot understand it,
and this cannot optimize it this way[^2].

The moral of the story is that you should write your code in a way that makes sense
to you and leave the optimizations to the compiler, because nowadays, compilers
will perform a much better job in optimizing our code than ourselves.
Note that by "optimizations" I do not mean algorithms. Algorithms is what developers
should focus on, because there is the real gain of performance. You can use
as many tiny optimizations such as using bifhits instead of multiplications[^1], if you
use [bogosort][bogosort] instead of merge sort, you'll certainly end up with a slower code ;).

### Notes

[^1]: arithmetic operators tend to be heavily optimized by compilers (for obvious reasons), so this kind of optimizations is not useful anymore. For example gcc will automatically transform a multiplication in addition of bitshifts if it can, and that without `-O` (with `-O` it would have tried to pre-compute the result at compilation time).
[^2]: In fact, even if the compiler does not compile the code down to the `bswap` instruction, it can optimize it a way that it takes nearly the same time
[bogosort]: http://en.wikipedia.org/wiki/Bogosort
