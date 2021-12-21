# Social C

The programming language `C` is being under heavy attack in recent years as it is prone
to memory vulnerabilities, allowing very serious exploits, making systems we all rely on
insecure.

To be clear, the language `C` itself is secure. A `C` program does exactly what
the programmers told it to do. The problem is that it is hard to express in `C`
what exactly you want it to do. Compilers for `C` are being improved all the time to
give better *warnings* in case it seems unclear what the purpose of your C statements
is.

However, there is a limit in how expressive you can be in `C`, especially when it comes
to dynamic memory use, lifetimes of allocated objects, concurrent operations and many
things a modern application has to cope with.

The problem is so urgent because `C` is *universally everywhere*. The ever growing business
towers in our internet have it all in their foundations. And it is still spreading at incredible
speeds into toothbrushes and Mars helicopters.

Why?

"It is irresponsible to write anything in C!" is being stated as a serious expert opinion. But
programmers continue doing it, myself included. Are we all tone deaf?

Speaking just for myself, there are two main reasons for me continuing this in the foreseeable
future:

1. **Maintenance**: someone needs to maintain and improve the systems we all rely upon (and we we all use to 
carry the debates, btw). Apache `httpd`, one of the elder components of the internet, is still in heavy
use around the world. It would server no one, if the project would just stop caring about it.
2. **Fitness for Purpose**: I see no alternative that could replace `C` in its entirety (yet). That deserves some more explanation of what I mean with *entirety*.

![](images/aquaeduct.jpg)

## Culture

The social aspect of `C`,  its culture, defines the ways different people, teams and organizations
work with each other. There are defined contracts how it is all fit together to create what we commonly
call the systems on the internet.

When you run a server, locally or somewhere in the cloud, you commonly use an OS put together out
of thousands of parts, from hundreds of thousands of people which, seemingly miraculously, work
well together. At least well enough that you chose to use it.

Over the decades, they way this is done, has been refined (and continues to evolve). There are contracts,
teams and heavy tooling. Between programmers and users, there are many steps and responsibilities
being taken care of.

This culture evolved *around* the `C` language. Its ways are less defined by the *language* C as
by its tools for *interface definitions* (FFI) and infrastructure for *assembling and
provisioning* (linking and packaging).

This **social** aspect of `C` allows me to make a bugfix in `httpd` that reaches the users
in a matter of hours. It's a big accomplishment of this culture. One that happens day after
day after day.

The technological centerpiece of this is the *shared library* system invented at Sun Microsystems in
the early years of their SunOS development. 

Libraries, functional components developed in different teams, had already existed for ages. The
innovation of *shared* libraries brought two main things:

1. **Memory efficiency**: non-shared, e.g. *static*, libraries are loaded again and again into memory for each executable using them. This became particularly burdensome with the advent of the X11 window system and its huge libraries (comparatively at the time). One more X11 app started and your system was coming to an almost stop.
2. **Libraries as Deliverables**: a shared library became an identified system component. It was installed
and *updated* separately from all the apps using it. A **versioning** system was introduced that defined backward
compatible or new updates. Different major versions could exist together on a system, if needed.

This second aspect of the shared library innovation made the web of dependencies between libraries and other components so much more simple. Teams had a stable interface for the components they use. And library developers had a safe way to *evolve* their component, e.g. incrementing major versions, without breaking apps.

Without this smart management of dependencies modern software development would not exist.

## Conclusion

So, when you think about an alternative to `C`, have this whole Social aspect in mind. For now, I 
do not see a real contender in this space, yet. That may be my ignorance. 

My point is: if you propose a technology as a substitute for the `C` *system*, you need to cover all of this. 





