---
layout: post
categories: [C]
tags: [programming, C, malloc, tutorial, article, howto, recoding, coding, recode, code]
title: "[C] On the Quest of recoding malloc(3)"
share: true
---

Back when I did not know anything about programing and started to learn C, I was first introduced to pointers (and other dreaded horrors that made me curl into a corner and cry) and dynamic memory in general.

I was baffled, troubled, yet fascinated by the basic explanation on how memory worked, and started to dread the time where I would need to manually create my char arrays for each and every sentences of my program; right before learning about string literals and feeling like an idiot.

It was then where I was learning about memory allocation and came upon a function that I would call for long the "magic function" : *malloc*.
Magic, because at that point I didn't know how it worked, let alone knew anything about memory other that it was a "chain of boxes for numbers".

Time passes and I recently had as an assignment to code malloc. This article is meant to share my experience on recoding the function, as well as some kind of howto since some people asked me how it was done.

## Preparing the journey

Recoding malloc is not the simplest of tasks, as its implementation will mostly depend on the kernel you are running -- as such, I am going to assume that we are running a Linux system, although this article should apply for any POSIX-compliant UNIX system. I am not familiar enough with Windows NT's heap functions to cover malloc's implementation for Windows.

You will **need GDB** if you have bugs -- no joking. You just don't have access to any function of the standard library as most will rely on their implementation of malloc.

I will consider that you already know the memory layout of an UNIX process, as well as what the program break is. If you don't, there is an addendum covering that at the end of the article.

Lastly, if you haven't yet, you should really read the manual pages for [malloc(3)](http://linux.die.net/man/3/malloc).

## The dumb allocator

We know from [sbrk(2)](http://linux.die.net/man/2/sbrk) that we can manipulate the program break, asking for or returning dynamic memory to the operating system.
Hence, the dumbest allocator does just that : 

{% highlight c linenos %}
#include <unistd.h>

void *malloc(size_t size) {
    void *c = sbrk(size);
    return c == (void*) -1 ? NULL : c;
}
{% endhighlight %}

### Trouble ahead

That's, however, not enough to run a program -- the `malloc` man page mentions `free`, `calloc`, and `realloc`. We need to implement them first before testing out the allocator.
`free` brings up our first issue with the allocator : we have no way in hell to know where the allocated segments are, and since our only mean to return memory to the system is to decrement the program break, `free` is just not feasible yet and will be empty.

{% highlight c linenos %}
void free(void* ptr) {}
{% endhighlight %}

`calloc` is pretty much straight forward : you call `malloc` and fill the memory section with zeroes.

`realloc` might cause some trouble, as we need to copy the old memory section to the newly allocated section, padding with zeroes if needed. Unfortunately, as for free, we have no way to know the size of the old segment. At that point, we are pretty much stuck with that model, and the best we might be able to do is to allocated fixed size blocks and pray that the running program will never want to allocate something bigger.

### The whole code

{% highlight c linenos %}
#include <unistd.h>
#define SIZE 1024

void *malloc(size_t size) {
    void *c = sbrk(SIZE);
    return c == (void*) -1 ? NULL : c;
}

void free(void *ptr) {}

void *calloc(size_t nmemb, size_t size) {
    void *ptr = malloc(size);
    char *b = ptr;
    for (size_t i = 0; i < SIZE; ++i, ++b) {
        *b = 0;
    }
    return ptr;
}

void *realloc(void *ptr, size_t size) {
    if (!size) return NULL;
    void *newptr = malloc(size);
    if (ptr) {
        char *b1 = ptr, *b2 = newptr;
        for (size_t i = 0; i < SIZE; ++i, ++b1, ++b2) {
            *b2 = *b1;
        }
    }
    return newptr;
}
{% endhighlight %}

This should work, given that the running program never tries to allocate more than 1024 bytes -- and here are the test results :

{% highlight bash %}
$ make debug
clang -Wall -Wextra -g -fPIC -c -o malloc.o malloc.c 
        && clang -shared -o libmalloc.so malloc.o
malloc.c:6:21: warning: unused parameter 'size' [-Wunused-parameter]
void *malloc(size_t size) {
                    ^
malloc.c:11:17: warning: unused parameter 'ptr' [-Wunused-parameter]
void free(void *ptr) {}
                ^
2 warnings generated.
$ LD_PRELOAD=./libmalloc.so /bin/ls
libmalloc.so  Makefile  malloc.c  malloc.o
$ LD_PRELOAD=./libmalloc.so /usr/bin/grep -R "" /
zsh: segmentation fault (core dumped)
{% endhighlight %}

As you noticed, /usr/bin/grep segfaulted, most likely because of the sizing issue.

## Fiat Stuctura

We know from the above that we need to store some metadata -- but where ? You can't really expect to call malloc in malloc.
There are actually two ways of doing that : since you are in control of the allocation process, it is completely possible to prepend the segment with some data, or to build a separate structure to store it all.
One of the most straight forward way to store and order the segments is by using a linked list, and I will first be covering this simple implementation.

Since we are going to prepend all of these to the actual segment, we also need a security to check if the pointer passed to free/realloc is actually valid; we will then add a pointer to the beginning of the data to the chunk metadata.

Thus, each memory chunk must at least "know" the following about itself :

* Its size
* Its predecessor
* Its successor
* If the space is marked free
* The pointer to the data following the chunk

We finally have a structure of the sort :

{% highlight c linenos %}
struct chunk {
    struct chunk *next, *prev;
    size_t size;
    int free;
    void *data;
};
{% endhighlight %}

The only thing left would be to set up the data structure. You start off with a sentinel to keep track of the start of the heap and deal with ease with chunk removal, then for each call of `malloc` you increment the program break by a word-aligned value of `sizeof(struct chunk) + size`, where `size` is the parameter passed to `malloc`.

This implementation is overall a bit trickier due to the pointer arithmetic galore you need to handle -- It's easy to get things wrong and corrupt memory. These bugs will typically backfire and cause unexpected behavior at any point of the program, and hence are really hard to track. This is why I would advise to start slow and incrementally add the new parts while making sure everything's working fine.

### Improving the model

Since memory allocation with `sbrk` is basically pushing further the program break, we might want to avoid calling `sbrk` when the call is, in fact, not needed : instead of always claiming dynamic memory, we iterate through the chunk list to find the first free chunk with sufficient space, and if none found **then** we allow ourselves to call `sbrk`.

As we reuse free spaces, the must here would be to split the reused chunks if they are oversized and insert a new chunk right after. In the same manner, it would be a shame to let free chunks congregate without merging them into one big free chunk, as it might cause issues with performance for programs that need to allocate large amounts of memory, as they would spawn a lot of chunks.

The resulting code with all of these improvements can be found on [the github project](https://github.com/Snaipe/malloc/).

## Beyond the call of malloc

Huzzah ! We managed to have a somewhat acceptable memory allocator !

We could consider that the objective is met, that we succeeded in our quest and shall return victorious with the head of the slain beast as a trophy.
Now that wouldn't be fun, wouldn't it ?

Truth to be told, this implementation sucks. There are some big issues with performance, and performance with the memory manager is critical as it could spawn a significant amount of overhead for all programs.

### Changing the structure

The first issue we have is that we use a linked list, and we constantly iterate through the list. This is excruciatingly slow, as in the worst case, you would need to iterate through the whole damn list before concluding that you need to allocate new memory; this might not really be an issue for small programs, but when there has been thousands of allocated variables, the performance would drop over time.

There are alternatives to the linked list, with among them :

* [Buddy blocks](https://en.wikipedia.org/wiki/Buddy_memory_allocation)
* [Slabs](http://en.wikipedia.org/wiki/Slab_allocation)
* [Memory pools](http://en.wikipedia.org/wiki/Memory_pool)

### Working with pages instead of the program break.

The program break is a lie.

The operating system manages memory using [pages](https://en.wikipedia.org/wiki/Page_(computer_memory)) rather than a program break -- what sbrk does is simply increment the program break, and if it crosses the boundary of the current page, it shall ask to the kernel for a new page. Since your program break is *somewhere* on the page, you may access memory beyond that position, given that you do not cross the page boundary.

Instead of using the program break, we might as well directly ask for pages to the system using [mmap(2)](http://man7.org/linux/man-pages/man2/mmap.2.html).

### Thread safety

As mentionned by [smcameron on reddit](http://www.reddit.com/r/programming/comments/226ran/some_friends_asked_me_how_one_would_implement/cgk3kbg), this implementation is absolutely not thread safe and will lead to undefined behavior without any synchronization.

## Conclusion

The choice of implementation remains with the programmer, as there are a lot of trade-offs to consider when choosing a particular strategy over another.

Writing a custom memory manager was fun, and I will recommend anyone with a bit of C experience to do the same, as it gives you a better understanding of the inner mechanisms of your system. I would not advise the production use of these unless you know what you are doing; chances are that the standard library has a much more efficient memory manager than your custom one. You might also check some other implementations such as [tcmalloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html) (a really interesting one indeed !)

I shall now return to my [cave](http://youtu.be/jBeDOvxatKw?t=2m6s) and do... things. Until then, bye.

# Addendum

## Debugging with GDB

Debugging with GDB might be tricky if you don't set it up properly. You just can't do something like LD_PRELOAD=./libmalloc.so gdb, because, well, gdb WILL use malloc.
Fortunately, GDB comes with the set exec-wrapper function :

{% highlight bash %}
$ gdb -q /bin/ls
Reading symbols from /bin/ls...(no debugging symbols found)...done.
(gdb) set exec-wrapper env 'LD_PRELOAD=./libmalloc.so'
(gdb) run -q -a
.  ..  libmalloc.so  Makefile  malloc.c  malloc.o
[Inferior 1 (process 25201) exited normally]
(gdb) quit
{% endhighlight %}

With this, you will be able to properly debug the library.

## The memory layout of an UNIX process

Typically, a process under a UNIX operating system will take 4GB of virtual memory space (3GB for the user, 1GB for the kernel, although some kernels support the hugemem feature that extends to 4GB for both spaces). I will not cover in detail how the memory is layed out on these 4 gigs, and will focus mainly on the stack & heap.

The stack and the heap are memory sections belonging to the 3GB of user memory space -- both will expand if necessary, although the stack will start from a high address in memory and expand downwards while the heap will start from a low address and will expand upwards. These sections are bounded by a delimiter, marking where the heap and stack ends. This delimiter is called the program break (or brk) for the heap and the top of the stack pointer (or %esp, from the assembly notation) for the stack.

For the visual people out there, [here](http://musingsofagator.files.wordpress.com/2013/03/virtual-mem-layout.jpg) is a diagram showing a simplified layout.

## Further readings

* [*ilsken* on Github](https://github.com/ilsken) provided links of the more complex implementations of [glibc's malloc](https://github.com/lattera/glibc/tree/master/malloc) and [jemalloc](https://github.com/jemalloc/jemalloc) (used by facebook).

* [*wm4* on Github](https://github.com/wm4) added a link to [a very good data structure and algorithm for malloc](http://g.oswego.edu/dl/html/malloc.html), along with it's [implementation](ftp://g.oswego.edu/pub/misc/malloc.c). He also posted a link to [musl's malloc implementation](http://git.musl-libc.org/cgit/musl/tree/src/malloc/malloc.c).
