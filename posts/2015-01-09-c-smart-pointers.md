---
layout: post
categories: [C]
tags: [smart pointers, programming, C, article, howto, coding, code]
title: "[C] Implementing smart pointers for the C programming language"
share: true
comments: true
---

I am a big fan of C, but some part of me always yearn to have *just
enough* higher level constructs.

The impracticality of memory allocation in C is one of my pet peeves.
Being very easily distracted, I tend to forget to free my memory, or
close my resources, so you might guess that when I learned about
smart pointers in C++, I was immediately hooked.

As a freshman in my IT engineering school, our first semester was mostly
about programming in C. We were assigned pretty simple, solo projects,
until *the* big project came in --- we had to implement a POSIX bourne
shell, as a team of 4 people, and we were warned that if we leaked,
we would eat a pretty hardcore malus on our final grade. We also had
to use gcc 4.9 as our compiler.

Because I knew that my teammates and I were certainly not perfect, I
decided to try and make our life easier. I first thought about
[wrapping malloc][wrap-malloc] to register all allocated data, and
then free everything before exiting, but it wasn't a satisfying solution.

I went and read a little more about gcc's attributes, and found
`__attribute__ ((cleanup(f))`: according to the documentation,
this variable attribute would call the function `f` on the variable
before exiting the current scope.  
I was delighted.

## Implementing an auto-cleaning variable

I went on and figured that I would simply make some kind of
would-be type attribute to make the variable free itself after
going out of scope:

{% highlight c %}
#define autofree __attribute__((cleanup(free_stack)))

__attribute__ ((always_inline))
inline void free_stack(void *ptr) {
    free(*(void **) ptr);
}
{% endhighlight %}

The usage was pretty straight forward:

{% highlight c linenos %}
int main(void) {
    autofree int *i = malloc(sizeof (int));
    *i = 1;
    return *i;
}
{% endhighlight %}

But while it worked well for these tests, when I started refactoring
the existing code with the new keyword, it simply didn't prove to be
*that* useful, because most of the dynamically allocated data was
complex data, with (also dynamically allocated) members, and since you
can't slap a cleanup attribute on a struct member, I had to do better.

I needed some kind of destructor function mechanism.

I decided to prepend some metadata to the allocated memory -- this ought
to be the most non-intrusive way to work my way to the new goal:

    |------------------------|---------------------- // -----|
    | metadata    | padding? | actual data                   |
    |------------------------|---------------------- // -----|
    ^                        ^ returned address (word aligned)
     `- start of allocated
        block

## Thou shalt have my metadata

I needed two functions: one to allocate the memory, and one to free it.
Hence came into existence `smalloc` and `sfree`:

{% highlight c linenos %}
struct meta {
    void (*dtor)(void *);
    void *ptr;
};

static struct meta *get_meta(void *ptr) {
    return ptr - sizeof (struct meta);
}

__attribute__((malloc))
void *smalloc(size_t size, void (*dtor)(void *)) {
    struct meta *meta = malloc(sizeof (struct meta) + size);
    *meta = (struct meta) {
        .dtor = dtor,
        .ptr  = meta + 1
    };
    return meta->ptr;
}

void sfree(void *ptr) {
    if (ptr == NULL)
        return;
    struct meta *meta = get_meta(ptr);
    assert(ptr == meta->ptr); // ptr shall be a pointer returned by smalloc
    meta->dtor(ptr);
    free(meta);
}
{% endhighlight %}

Both functions are pretty straight-forward: `smalloc` allocates memory
to host both the requested data size and the metadata we need. It then
initializes said metadata and stores the destructor, and returns the pointer
to the start of the uninitialized user data.

`sfree` behaves exactly like `free`, in that it does nothing if `NULL` is
passed, and otherwise deallocates the memory. The only difference is that
it calls the destructor stored during the call to `smalloc` before the
actual deallocation, so that the cleanup step can be performed.

Given these two functions, I could rewrite my `autofree` macro:

{% highlight c %}
#define smart __attribute__((cleanup(sfree_stack)))

__attribute__ ((always_inline))
inline void sfree_stack(void *ptr) {
    sfree(*(void **) ptr);
}
{% endhighlight %}

I figured that this was kind of looking more and more like a smart pointer, so
I went ahead and renamed `autofree` to `smart`.

One of the immediate consequences of `sfree` running a destructor was that
`sfree` was the universal deallocator, akin to the `delete` keyword in C++.

This means that for any type managed by `smalloc` I could just call
`sfree` on it without worrying about the real destructor function, and I
would know that it would have been destroyed -- this was a **huge** improvement.

One drawback here is that even for simple types, we had to specify a valid
destructor, and this is not always desirable for simple types, so I allowed
`NULL` to be a valid parameter to mean that there is no destructor.

I also took the time to add a way to put user-defined data into the
metadata block, since it could have some nice applications, like
length-aware arrays, or extending existing library structures.

## On the path to unique\_ptr and shared\_ptr

With all of the above done, we pretty much have the needed foundations
in place for `unique_ptr` and `shared_ptr`. In fact, we currently have
`unique_ptr`.

The only thing left is `shared_ptr` then -- easier said than done.
Shared pointers rely on thread-safe reference counting. From there, two
choices: locks, or atomics.

Using locks is easy, but it makes smart pointer impossible to use in
signal handlers; not much of a problem, but still annoying.

A lock-free implementation would be ideal, but there is a big issue with
that: lock-free implementations rely on a compare-and-swap mechanism on
the wrapped pointer -- here, the pointer is not wrapped.
This sucks, but if we make the assumption that no destruction shall happen
during an async context, it's kind of fine (Oh boy, this will definitely
make people jump out of their seat).
On a side note, I have yet to see a proper solution to that, and I would
like to avoid having double pointed types to solve the issue -- if anyone
has an idea, feel free to send me a message or send a pull request on the
github repository.

For now, I included an atomic integer in the metadata, and a `sref`
function: each time it is called on a shared pointer, the internal
reference counter increments by 1, and each time `sfree` is called on
said pointer, it is decremented by 1.

We still need a way to make a difference between unique and shared
pointers, `smalloc`-wise. To do that, I added another parameter to
specify the kind.

Oh, well. That's a lot of parameters, now. We have the size, the
destructor, the user data, the size of said data, and now the kind of
resulting pointer. We started with a simple concept, but if I have
to specify five parameters for each allocation, I'd rather go back
to `malloc` and `free`.

## Macros to the rescue

One of the first improvements we could do, is to make `smalloc`
variadic --- after all, the destructor and the user metadata are
completely (and should be) optional. The issue with this is that
we need to pass some kind of indicator of the number of variadic
parameters. Some use NULL as a sentinel, but it's not that practical,
and some use a parameter to specify that number, but it's not
that better.

The solution here is to go with the additional parameter, and
wrap everything with a macro. The macro would count the number
of variadic parameters and dispatch them to the real `smalloc`
function.

Add two other macros for `unique_ptr` and `smart_ptr` to call
`smalloc` with the proper kind parameter, and we have a practical
smart pointer library:

If I'd like to have an unique pointer with a destructor, I'd use
`unique_ptr(sizeof (T), dtor);`.  
For a shared pointer with no destructor but user data,
`shared_ptr(sizeof (T), &data, sizeof (data));`...
and this is, at last, [absolutely satisfying][github-examples].

**Edit 15/01/2015:** On the advice of [/u/matthieum][reddit-matthieum]
on reddit, I changed the size parameter of my `unique_ptr` and
`shared_ptr` macros to use the actual type -- this makes the usage even easier.
On top of that, I implemented smart arrays, and managed to
take in array type parameters like `int[9]` and infer the size
of the element type and the length of the array.

## Wrapping everything up...

There we have it, smart pointers for everyone. I had fun pushing the
language a bit further to make it do things it normally would not,
although the project itself needs to mature a bit more before being
ready for some kind of production.

I have wrapped everything in a [github repository][repo], for everyone
to toy with. Installation instruction & usage samples are included in the
readme. Have fun reading through, curious developer!

[repo]: https://github.com/Snaipe/c-smart-pointers
[github-examples]: https://github.com/Snaipe/c-smart-pointers#examples
[wrap-malloc]: http://stackoverflow.com/a/3662951/1749566
[reddit-matthieum]: http://www.reddit.com/r/programming/comments/2s6h6u/implementing_smart_pointers_for_the_c_programming/cnmsvjv
