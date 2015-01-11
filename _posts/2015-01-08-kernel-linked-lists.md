---
layout: post
title:  "Linux kernel linked lists"
date:   2015-01-09 00:01:23
categories: kernel linked lists linux
---

Recently I came accross the `container_of` macro defined in the linux kernel.
This macro is used to get the address of the container (a structure) from
the address of one of its members. Let's see how it works, and how it can
be useful to implement linked lists a clever (but weird) way.

For example, take a look at this code:

{% highlight c linenos %}
struct useless_structure
{
  int  member1;
  char member2[255];
  long member3;
} s;

if (&s == container_of(&(s.member1), struct useless_structure, member1))
  printf("SUCCESS\n");
else
  printf("FAILURE\n");
{% endhighlight %}

Here we can see that knowing the address of a member (here `&(s.member1)`),
the name of the structure (here `struct useless_structure`), and the name of
the member in the structure (here `member1`), we can get the address of the
structure (which is `&s`).

## How `container_of` works
Here is how `container_of` is defined:

{% highlight c linenos %}
#define container_of(ptr, type, member) ({ \
                const typeof( ((type *) 0)->member ) *__mptr = (ptr); \
                (type *)( (char *) __mptr - offsetof(type, member) );})
{% endhighlight %}

Before we can understand this macro, we have to know a little bit more about
the `offsetof` macro that it uses.

### The `offsetof` macro

`offsetof` returns the position
in bytes of a member of a struct relative to the address of the struct.
In our example, `offsetof(struct useless_structure, member1)` would be
0, and `offsetof(struct useless_structure, member3)` would be `255 +
sizeof(int)` because `member3` is just after `member2` (255 bytes)
in memory which is just after `member1` in memory (`sizeof(int)` bytes).

The `offsetof` uses a really clever trick. It casts `0` to a pointer to
the structure (so the address of this structure is 0) and the get the
address of the desired member of its structure. This code is safe
since we do not dereference the null pointer, in the sense we do
not access its data directly, we only get its address.

{% highlight c linenos %}
#define offsetof(st, m) ((size_t) (&((st *) 0)->m))
{% endhighlight %}

Some implementations of `offsetof` substracts the value of the null
pointer to the result in case the null pointer does not compare to
0 (which is really rare).

### How `container_of` uses `offsetof`

The `container_of` macro just takes the address of a member and substract
from it the address of this member relatively to the structure. Since elements
in a structure are contiguous, it works well.

For type-safety, `container_of` creates a pointer to the member so that
the compiler can generate a warning the the member is not part of the structure.
This trick is optionnal, but is a good practice. The macro then cast this pointer
to a char pointer so that the subtraction will work as expected
(pointer arithmetic would have massed up with types that occupy more than one
byte in memory).

## Linked lists

Now, here is the question that came to my mind and that should come to yours
too: when is `container_of` useful, if it is at all?

The Linux kernel primarily uses it to implement linked lists, but instead of
having the classical

{% highlight c linenos %}
struct  list
{
  void        *data; /* pointer to data */
  struct list *next; /* pointer to next node */
  struct list *prev; /* pointer to previous node */
};
{% endhighlight %}

where each node of the list holds a pointer to the data, Linux makes the
data hold a pointer to the node:

{% highlight c linenos %}
struct list
{
  struct list *next; /* pointer to next node */
  struct list *prev; /* pointer to previous node */
};

struct  contact
{
  char  *name;
  char  *phone;
  struct list *node; /* pointer to the node */
}
{% endhighlight %}

we can then access the data of a node using `container_of(ptr_list, contact, node)`
which points to our `contact` "instance".

One of the advantage of this method is that we cannot have two different linked lists
containing the same structure (which can be a problem when deleting an element from the list
frees it, since it will create a ghost pointer in the other list), without being aware
of it, because it can be seen clearly in the structure definition.