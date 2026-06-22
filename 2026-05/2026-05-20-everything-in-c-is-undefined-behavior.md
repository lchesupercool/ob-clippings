---
title: "Everything in C is undefined behavior"
source: "https://blog.habets.se/2026/05/Everything-in-C-is-undefined-behavior.html"
author: "Thomas Habets"
published: "2026-05-19"
saved: "2026-05-20 20:47:10 +0800"
tags: [c, cpp, undefined-behavior, programming, security, clipping]
---

# Everything in C is undefined behavior

    <script async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js"></script>
    <ins class="adsbygoogle horizontal-ad"
         style="display:inline-block;width:320px;height:100px"
         data-ad-client="ca-pub-4148956918309843"
         data-ad-slot="2882811709"></ins>
    <script async src="/static/ads.js"></script>

2026-05-19, Categories: programming, security

If he had been a programmer, [Cardinal Richelieu](https://en.wikiquote.org/wiki/Cardinal_Richelieu) would have said “Give me six lines written by the hand of the most expert C programmer in the world, and I will find enough in them to trigger undefined behavior”.

Nobody can write correct C, or C++. And I say that as someone who’s written C and C++ on an almost daily basis for about 30 years. I listen to C++ podcasts. I watch C++ conference talks. I enjoy reading and writing C++.

C++ has served us well, but it’s 2026, and the environment of 1985 (C++) or 1972 (C) is not the environment of today.

I’m definitely not the first to say this. I remember reading a post by someone prominent about a decade ago saying that a good case can be made that use of C++ is a [SOX](https://en.wikipedia.org/wiki/Sarbanes%E2%80%93Oxley_Act) violation. And while I was not onboard with the rest of their rant (nor their confusion about “its” vs “it’s”), I never disagreed about that point.

With time I found it to be more and more true. WAY more things are undefined behavior (UB) than you’d expect.

Everyone knows that double-free, use after free, accessing outside the bounds of an object (e.g. array), and accessing uninitialized memory is UB. After all, C & C++ are not memory safe languages. And yet we as an industry seem to be unable to stop making even those mistakes over and over.

But there’s more. More subtle. More illogical.

## It’s not about optimizations

Some people seem to think that as long as they don’t compile with optimizations turned on, undefined behavior can’t hurt them. They believe that the compiler is somehow being deliberately hostile, going “AHA! UB! I can do whatever I want here!”, and without optimizations turned on it won’t.

This is incorrect.

UB doesn’t mean that the compiler can take advantage of your sloppiness. UB means that the compiler can assume that your code is valid. It means that the intention of your code that’s oh so obvious when read by a human, *doesn’t even have a way to be expressed* between compiler stages or modules.

UB means that the compiler doesn’t even have to implement some special cases in its code generation, because they “can’t happen”.

The compiler, and really the underlying hardware too, is playing a game of telephone with your UB intentions. It may end up with what you wanted, but there’s no guarantee for now or in the future.

## UB is everywhere

The following is not an attempt at enumerating all the UB in the world. It’s merely making the case that UB is everywhere, and if nobody can do it right, how is it even fair to blame the programmer? My point is that **ALL** nontrivial C and C++ code has UB.

### Accessing an object which is not correctly aligned

As an example of this, take this code:

```c
int foo(const int* p) {
   return *p;
}
```

If this function is called with a pointer not correctly aligned (probably meaning on an address that’s a multiple of `sizeof(int)`, but who knows), this is UB. C23 6.3.2.3.

On Linux Alpha, in *some* cases this would merely trap to the kernel, which would software emulate what you intended. In other cases it would (probably) crash your program with a SIGBUS.

On SPARC it would cause a SIGBUS.

Sure, on x86/amd64 (henceforth just “x86”) this is likely fine. Hell, it’s probably even an atomic read. x86 is famously extremely forgiving about cache coherency subtleties.

So here we have three cases:

1. kernel gave a helping hand (Alpha for *some* loads)
2. crash (other Alpha loads, and SPARC)
3. not a problem (x86)

What about ARM, RISC-V, and others? What about future architectures? A future architecture could even have special `int-pointer registers` that do not populate the lowest bits, because such pointers cannot exist.

Even if it works, maybe the compiler one day changes from using one load instruction to another, and suddenly that’s no longer fixed up by the kernel.

Because *the compiler is not obligated to generate assembly instructions that work on unaligned pointers*. Because it’s UB.

Or how about this:

```c
void set_it(std::atomic<int>* p) {
        p->store(123);
}
int get_it(std::atomic<int>* p) {
        return p->load();
}
```

Is this operation atomic when the object is not correctly aligned? That’s the wrong question to ask. [Mu](https://en.wikipedia.org/wiki/Mu_(negative)), unask the question. It’s UB. (but also yes, in practice this can easily be an atomicity problem)

If you want to get even more convinced, you can try thinking about what happens if an object you thought you were reading atomically spans [pages](https://en.wikipedia.org/wiki/Memory_paging). But don’t think too much about it, or you may conclude that “it’s fine”. It’s not. It’s UB.

### Actually, it was UB even before that

Don’t blame the `foo()` function, above. The act of dereferencing the pointer wasn’t the problem. Merely *creating* the pointer was enough to be a problem.

Example:

```c
bool parse_packet(const uint8_t* bytes) {
        const int* magic_intp = (const int*)bytes;   // UB!
        int magic_raw = foo(magic_intp);  // Probably crashes on SPARC.
        int magic = ntohl(magic_raw); // this is fine, at least.
        […]
}
```

That cast is the problem, not `foo()`.

It’s perfectly valid for the compiler to assign specific meaning, such as garbage collection or security tagging bits, to the lower bits of an `int*`.

### isxdigit() on char input

```c
bool bar(char ch) {
        return isxdigit(ch);
}
```

`isxdigit()` is a simple function that takes a character and returns `1` if it’s a hex digit. 0-9 or a-f. It can also take the value `EOF`. Uh, ok. What value is `EOF`? Per [C23](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3088.pdf) 7.4p1 we know it’s an `int`, and we can infer that it’s not representable by `unsigned char`.

`isxdigit()` therefore takes an `int`, not a `char`. All values of `char` fit inside `int`, so we should be fine. Casting from `char` to `int` fits, so per section 6.3.1.3 we’re fine, right?

No. Because if `bar()` is called with a value other than 0-127, **and** on your architecture `char` is `signed` (implementation defined, per 6.2.5, paragraph 20 in [C23](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3088.pdf)), then the integer value ends up negative.

And the following is a valid implementation of `isxdigit()`, that would cause a read of who-knows-what memory. It could even be I/O mapped memory, triggering things to happen that is more than merely getting a random value or crash. It could cause the motor to start. Less likely in an application running in a desktop operating system than in an embedded system, sure. But there are user space network drivers (for performance), so even user space won’t protect you.

```c
int isxdigit(int c) {
        if (c == EOF) {
                return false;
        }
        return some_array[c];
}
```

### Casting from float to int

```c
int milliseconds(float seconds) {
        int tmp = (int)(seconds * 1000.0); /* WRONG */
        return tmp + 1; /* WRONG separately (signed overflow is UB) */
}
```

```c
When a finite value of real floating type is converted to an integer type[…]If
the value of the integral part cannot be represented by the integer type, the
behavior is undefined.
— 6.3.1.4
```

And, by omission, it’s also UB if the float is a non-finite value.

So how do you compare a float to `INT_MAX`? Do you cast the float to `int`? No, that’s the UB you want to avoid. So you cast `INT_MAX` to float? How do you know it can be represented exactly? Maybe casting `INT_MAX` to `float` rounds to a value not representable in `int`, and your comparison becomes non-representative?

Maybe the following works? You’ll miss out on representing some really high values, but maybe that’s OK?

```c
int milliseconds(float seconds) {
        const float ftmp = seconds * 1000.0f;
        if (!isfinite(ftmp)) {
                // or other error reporting.
                return 0;
        }
        if ((float)(INT_MIN + 1000) > ftmp) {
                // or other error reporting.
                return 0;
        }
        if ((float)(INT_MAX - 1000) < ftmp) {
                // or other error reporting.
                return 0;
        }
        // Now safe to convert.
        const int tmp = (int)ftmp;
        if (INT_MAX == tmp) {
                // or other error reporting.
                return 0;
        }
        // Now safe to add.
        return tmp + 1;
}
```

I just wanted to convert a float to an int. :-(

I bet there’s lots of code out there that take a value in seconds, and convert it to integer milliseconds, by just multiplying and casting.

### Object at address zero

Most programmers won’t have to deal with this, but I don’t think there’s any C standards compliant way in practice to put an object at address zero. This can come up in OS kernel and embedded coding.

By 6.3.2.3 an integer constant zero (which is convertible to a pointer) and `nullptr` are the “null pointer constant” (which I’ll just call `NULL`). [C doesn’t specify that the actual pointer `NULL` points addr machine address zero](https://c-faq.com/null/confusion4.html), because the C standard only talks of the C abstract machine, not about hardware.

All C guarantees is that if you *compare* `NULL` to zero you’ll see them equal. But for all you know that’s because the zero is converted to the native platform’s `NULL`, which happens to be `0xffff`.

It also explicitly says that dereferencing a null pointer, no matter what the value, is undefined behavior. It’s *the* example of UB under 3.4.3.

This also means that you can’t assume that `memset(&ptr, 0, sizeof(ptr));` will create a `NULL` pointer! You cannot initialize your structs this way and assume member pointers are `NULL`! And this *does* apply to most programmers.

And yes, [some historic machines used non-zero NULL pointers](https://c-faq.com/null/machexamp.html).

But let’s say you have a modern machine, where `NULL` is a pointer to address zero, and you actually have an object there.

Again, C 6.3.2.3 says that `NULL` compares unequal to “any object or function”. So this is UB:

```c
void (*func_ptr)() = NULL;
func_ptr();
```

C says “there is no function there”. For all you know the compiler has no internal way to even express your intention here. You may argue that “but surely it’ll just emit a call instruction to the bit pattern of all zeroes? Nothing else seems reasonable.

What is “all zeroes”, though? On 16bit x86, is it `0000:0000`? Is it `CS:0000`?

### Variable arguments and types (e.g. printf with %ld instead of %lld )

This is UB:

```c
execl("/bin/sh", "sh", "-c", "date", NULL);     /* WRONG */
execl("/bin/sh", "sh", "-c", "date", 0);     /* WRONG */
```

This is not:

```c
execl("/bin/sh", "sh", "-c", "date", (char*)NULL);
```

Because the argument needs to be a pointer, and the `NULL` macro may be misinterpreted as an integer zero.

Similarly, this is UB:

```c
uint64_t blah = 123;
printf("%ld\n", blah);  /* WRONG */
```

It needs to be:

```c
uint64_t blah = 123;
printf("%"PRIu64"\n", blah);
```

So how do you print an `uid_t`? Well, you could cast them to `uintmax_t` and print them using `PRIuMAX`. But is `uid_t` even unsigned? Oh well, worst case you get a nonsense value printed instead of `-1`, I guess.

## Divide by zero is UB

Sure, you probably knew this. But did you consider the security aspects of it? It’s not rare for the denominator to come from untrusted input.

And there’s so much more. The C23 standard contains 283 uses of the word “undefined”. And that’s not even including the things that are undefined by omission.

## Bonus non-UB

Nobody can apply integer promotion rules at code skimming speeds. **Nobody**.

This post is already long enough, but as a start:

```c
unsigned char a = 0xff;
unsigned char b = 1;
unsigned char zero = 0;
bool overflowed = (a + b) == zero;
// overflowed is set to zero, not one.

unsigned char a = 0x80;
uint64_t b = a << 24;     // Bonus UB(?)
// b is now 18446744071562067968 (ffffffff80000000), not 2147483648 (0x80000000).
// even with all our variables unsigned.
```

## LLMs are better than us at this

Point an LLM at ANY C code, asking it to find UB, and it will. And it’ll be right almost all the time, nowadays.

I felt a bit bad after it correctly found ones in my code, so I thought I’d point it at the mature and pedantically written OpenBSD. I just picked the first tool I could think of, `find`, and it spit out a bunch.

I sent the project a patch for [an out of bounds write](https://marc.info/?t=177910871100010&r=1&w=2) (and also for [a non-UB logic bug](https://marc.info/?l=openbsd-tech&m=177910856447778&w=2)). I didn’t send them patches for the UB that was left and right, partly because the OpenBSD project has not been very receptive in the past for bug reports, my sense of “this is probably fine, in practice”, and that if OpenBSD wants to weed out UB from their code base, then that’s a major project that should be done in a better way than me just being the middle man between the LLM and them for a patch here and there.

## So what do we do now?

We can’t just throw away our C and C++ code bases. But leaving them inherently broken is also not an option.

We need some way of fixing UB at scale, without committing AI slop nor overwhelming human reviewers.

This too is not a new opinion, nor a great revelation.

But yes, writing C or C++ in 2026 without an LLM supervising you for UB should probably be seen as a SOX violation, and just plain irresponsible. If OpenBSD people can’t find these problems given 30+ years, what chance do the rest of us have?

It may not scale to large code bases, but for my own projects I’ve asked the LLM to find UB, if necessary explain it, and fix it. And then stare at the output until I can confirm the issue and the fix.

A problem with this is that in order to confirm the findings, you’ll need an expert human. But generally expert humans are busy doing other things. This is janitor work, but too subtle to leave to the junior programmers who have traditionally been assigned janitor work.

## Related

- [No way to parse integers in C](https://blog.habets.se/2022/10/No-way-to-parse-integers-in-C.html)
- [Integer handling is broken](https://blog.habets.se/2022/11/Integer-handling-is-broken.html)
- [UB in the Linux kernel](https://pooladkhay.com/posts/first-kernel-patch/)
- [Integer promotion](https://www.123microcontroller.com/en/cpp-integer-promotion/)

 This is what I've been using for years and years  
disqus has started showing ads. :-(
Showing (probably incomplete) comments in a static read-only view. Click button to be able to leave comments.
Show comments despite ads

 
  <noscript><a href="https://blargh.disqus.com/?url=ref&https" ping="/ping">View the comments.</a></noscript>
  <script async type="text/javascript" src="https://disqus.com/forums/blargh/embed.js?https"></script>
  
 This is just a test, and while it works, what comes up is entirely empty
  <div id="disqus_thread"></div>
  <script>
      /**
       *  RECOMMENDED CONFIGURATION VARIABLES: EDIT AND UNCOMMENT THE SECTION BELOW TO INSERT DYNAMIC VALUES FROM YOUR PLATFORM OR CMS.
       *  LEARN WHY DEFINING THESE VARIABLES IS IMPORTANT: https://disqus.com/admin/universalcode/#configuration-variables    */
        var disqus_config = function () {
                this.page.url = "https://blog.habets.se/2026/05/Everything-in-C-is-undefined-behavior.html";
                this.page.identifier = "/2026/05/Everything-in-C-is-undefined-behavior.html";
        };
        (function() { // DON'T EDIT BELOW THIS LINE
            var d = document, s = d.createElement('script');
            s.src = 'https://blargh.disqus.com/embed.js';
            s.setAttribute('data-timestamp', +new Date());
            (d.head || d.body).appendChild(s);
        })();
   </script>
   <noscript>Please enable JavaScript to view the <a href="https://disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
