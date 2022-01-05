# Social Swift? On APIs and ABIs

As feedback to
my earlier posts about [Social C](social_c.md) and [Social Java](social_java.md) I got a link from
[Cory Benfield](https://github.com/Lukasa) to an essay [How Swift Achieved Dynamic Linking Where Rust Couldn't](https://gankra.github.io/blah/swift-abi/) by Aria Beingessner. This proved *very* interesting for several reasons which I try to layout below.

## API stability, ABI stability, what are those, really?

Aria, who works at Mozilla on Rust, talks about this in depth in her article. And as usual, when
listening to someone *who really knows a thing* you find out that your own knowledge has been rather shallow 
so far. 

## API stability 
From a programmer point of view, **API** stability is often measured by "it continues to compile". Meaning, when you change the public types/functions in a component, do other components continue to work unchanged? **API** stability is useful because it reduces waste. If every change required changes in everything else, we'd never get anything done.

```
/* a sample API, version 1.0.0 */
typdef struct {
  int n;
} stat_t;

void stat_init(stat_t *s);

/* code using it */
stat_t s;

stat_init(&s);
printf("stat n: %d", s.n);
```

Above is a small API in C and code using it. With API stability we mean the ability to change our API in ways that keeps the code using it working unchanged. Like this:

```
/* a sample API, version 1.0.1 */
typdef struct {
  int n;
  long l;
} stat_t;

void stat_init(stat_t *s);
```

This change is compatible. We can re-compile the using code against v1.0.1 and it continues to work. However, the change will break ABI stability. We'll discuss this below.

## ABI stability

The critical difference between API and ABI stability is that the first applies *before* code compilation and the other *afterwards*. In our small example, this means one can take the compiled usage of API v1.0.0 and mix that together with API v1.0.1 and it continues to work.

> We'll leave the reasons why this is desirable aside for now. I've covered this in my blog about [Social C](social_c.md) and you'll find some more below in the Swift ABI chapter.

Staying in our C example, the change from v1.0.0 to v1.0.1 breaks the **ABI** because the caller passes a pointer to a `stat_t` it has allocated (on the stack) to the `stat_init()` function. That code was compiled using the old definition of `stat_t` and has no room for the new member `l`. The v1.0.1 `stat_init()` will however assume that there is and use memory that it should not.

There are many ways to break ABI stability in C and, since the language was not designed with this in mind, only
the programmer's discipline and attention will help here. People designing C APIs have learned what to avoid and 
with some care, the problem is manageable. 

To make compiled components work with each other, the ABI needs to include definitions of 

1. How Types are layed out in memory. How large they are, if they have restrictions on address alignment, how offsets in composite types (`struct`) are calculated, etc.
2. How Types are passed as arguments to function calls and returned as results.
3. Where arguments to function calls are stored (heap, stack, register).
4. Where returned Types are stored and can be picked up by the caller.
5. What kind of context information needs to be saved/restored/is free to mess with between caller and callee.

The exact details vary from platform to platform and are embedded into the compilers, linkers, debuggers and all sorts of runtime infrastructure. At the base is the format defining linkable components, so dynamic linkers can stitch them together at runtime. 

On linux and bsd systems [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) is the most common definition for the objects their ABI is based on. ELF has also been adopted in other operating systems and is used on devices such as Sony Playstations, Nintendos and Android mobile phones.

macOS and its siblings have their `Mach-O` as their equivalent to ELF in the [Darwin](https://en.wikipedia.org/wiki/Darwin_(operating_system)) OS. That has its roots in the BSD and Mach projects and began in 1989 as `NextSTEP`. (Linux first official release was 1991). ELF was first standardized in 1997 as ["System V Application Binary Interface"](http://www.sco.com/developers/devspecs/gabi41.pdf) and over the years extended for different architectures.

Quite a success story and decades of major development under the hood.

### ABIs and C and beyond

The ABIs that exist now on operating systems were invented after C had become the major system programming language. This explains why there is nothing about ABI management in the language C. And also why those ABIs are describing mechanisms that fit well onto the C way of types, structs and functions.

But C is not needed to generate ABI compatible objects such as executables or dynamic libraries. Other languages can do that as well - as long as their requirements do not exceed what the ABIs can do.

Surprisingly, the first major language that kind of broke the ABI limits was C++.

Well, kind of. One can implement dynamic libraries in C++, of course. But here is some special sauce involved. For
example function and operator overloading and namespace are features that need careful mapping into the flat ABI namespace. The mapping of C++ APIs onto ABI objects began really to creak ominously when Templates were added to the language in the late 90s.

The Standard Template Library (STL) is a collection of *generics* that allow to apply code recipes to new types, in a type safe way. For example the maximum of two type values is defined as:

```
template<typename T> T max(T a, T b) { return a > b ? a : b; }
```

When a program, using the STL, has statements like

```
    cout << max(1, 7);
    cour << max(0.6, 1.2);
```

the C++ compiler uses the `max` template from the STL to *generate an implementation* (for common types it might already have one, but one can use the template also for application specific ones). In the first compiler versions, this was done in a very straightforward way and blew up the size of C++ executables considerably. Later came optimizations, but the basic truth holds: *template implementations are owned by the user, not the library offering them*.

This means, when a template has a bug and is fixed in a patch release, this does not fix the code already built. One needs to re-compile all usage of the template for the fix to have an effect. Because C++ templates are not mappable into the ABIs that exist. There simply is no generic binary code inside the STL that can be linked here. The generics exists only in the API.

This is not easily fixed. No magic extension of ELF or other ABI standards could provide all possible implementations of `max()` in a dynamically loadable STL.

> The equivalent of generics in C is, of course, a macro. If you fix a macro in your C API, this will only have effect in applications when they are re-compiled. 

The term for all this is a `monomorphized generic`, e.g. the template is used to copy+paste the code for the type needed. The alternative is a `polymorphic generic`, where one implementation can be used for all sorts of types. Then the implementation code is inside a library and can be fixed/enhanced in patch releases.
 
Monomorphized generics are used in C++ and Rust. Polymorphic generics are offered by Swift. Swift may choose to implement a generic as monomorphized, for example when it is certain that it all happens inside the same component (this allows for all sorts of code optimizations). But if a generic is public, it will provide the polymorphic implementation for linking.

### Swift Stable ABI

With Swift 5 Apple introduced a [Stable ABI](https://www.swift.org/blog/abi-stability-and-more/) for macOS, iOS, watchOS and tvOS. The 'Stable' in their announcement refers to their promise that it will be backward compatible in future Swift versions. That you will be able to dynamically link components generated with Swift 5 to code produced with later versions of Swift.

That ABI has capabilities way beyond what ELF and its friends can do. The details are well described in [Aria Beingessner
's blog](https://gankra.github.io/blah/swift-abi/). In short: it allows backward compatible evolution of an API in many desirable features. Not only generics, but also for composite types and other things. The full extend can be read in Apple's [Swift Library Evolution](https://github.com/apple/swift/blob/main/docs/LibraryEvolution.rst) document.

Why does Apple make this big investment? Well, this is speculation, but my money would be on:

1. For memory safety reasons, Apple wants to replace more and more of their code with Swift implementations. (Read Paul Kehrer's report about [2021 in Memory Unsafety - Apple's Operating Systems](https://langui.sh/2021/12/13/apple-memory-safety/) - there is a possible pun here with "Swift on Security".)
2. Dynamic Linking is the most cost effective way to update iOS components with fixes. Apps are signed binaries made outside Apple as far as I know.
3. Dynamic Linking still saves a significant amount of memory on Apple devices. Less memory used also means longer battery life.

Sadly, there is no initiative to bring this ABI onto other platforms. While the Swift compiler is available on Linux and Windows, the ABI requires more than that. As noted above, this is a decision by the platform. Given Apple's track record, one cannot expect them to invest in this outside their own OS.

## Conclusions

While I look at dynamic linking from a Social aspect, e.g. how we all can work productive together, the Swift ABI shows that there is real business interest in this as well. It is not obsolete in the area of cloud stacks and docker images we live in.

Is there someone working on a similar thing for the *nix world? I don't know. Who would that be? I don't know. But I am sure that I would like to use it.
