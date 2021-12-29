# Social Java

This is a mini series of blogs about the social aspects of programming languages
and how they allow for interaction between developers and deployments. It does
not cover language details. This article focuses on Java, what worked and what did not.

(Caveat: this reflects on my experiences with the Java of 2000-2010. There might be newer,
relevant changes that I am unaware of. Read it as a retrospective
on what was there at the time and what impacts it had.)

## System

The Java language was started in 1991 when the world of computing was very different
from today. The version 1.0 was released in 1996. The slogan was to "develop once, run
everywhere". There were many systems and architectures around at the time, and Java
provided a *Virtual Machine* (VM) on each platform where application code would run unchanged,
even without re-compilation.

The delivery format to the VM is *compiled* code,
e.g. byte code instructions for the VM, so called *class* files. Since each *class* file is
generated from one *source* file, class files are too small for meaningful distributions. The
`jar` file was invented to bundle several class files. A jar is basically a `tar` file with
a defined substructure and, most importantly, a `manifest` file that carries meta data. Among
that is a name and a version number.

Jar version numbers initially were only informative. There is nothing in the VM
concerned with these and code needing specific versions had no way to mention this. Instead,
the burden of solving this problem was left to the people *deploying* a Java application.

(The VM and class files have a version identifier to check if the byte code is compatible,
 but that has nothing to do with the version of the compiled software.)

## Deployment

Each Java application bootstraps from a Java Runtime Environment (JRE) which provides the VM
and the Java Standard Library, basically a set of standard Jars with base functionality. Everything
beyond that was to be provided by the installed application.

There was no way to add a component to the JRE other than getting it adopted by the JRE
maintainers in a future release. There was a process for this and the Java Runtime continued
to grow, but it was a centralized decision, taking time.

As an application developer, when you wanted to use a Jar from someone else, you had to *copy*
it into your application. This can be done either by including the files in the bundle you
ship or load it from the network. Network loading was done in the Java plugin for browsers, but other
applications preferred copying since network access was not universally available, reliable
enough or with sufficient bandwidth (think deployment on an enterprise server that is
not allowed to connect outside for good reasons).

Copying Jars freezes them in time. From deployment view this has the same properties as a
statically linked C library. To provide any fixes for such an included jar, one needs to
re-bundle the application with an updated copy and ship/re-install that.

Each Java application was its own island, containing dynamically loaded Jar libraries
for its own purpose. A system with several Java applications was likely to have many copies
of common Jars and in different versions. Whatever was recent when the application was
last updated (at best).

With the security needs of today, this is a problematic situation. Admins are scanning their
systems for vulnerable Jar versions. This is not as easy as it sounds. For various reasons,
Jars are not only loaded from the file system, but might come from the network or be buried
inside databases (`URLClassLoader` is a standard feature that can load class file from any
URL format. Even custom ones by sub classing).

This makes it really hard to assert a system no longer uses a vulnerable component.

![](https://uproxx.com/wp-content/uploads/2020/09/starwars-the-phantom-menace-liam-neeson-jar-jar-binks_lucasfilm-brightened-wide.jpeg?w=1024&h=428&crop=1)

## Versions

Version Management for Jar files is not part of the VM or JRE. It existed for *NIX
shared libraries, but was not picked up in Java. 
The standard class loading just used the first Jar with the correct names it found. Every
incompatibility was a Runtime Exception.

Developers of large Java systems (read enterprise applications and people building set-top
boxes and telephones) soon found out the problems with that. Components you want to integrate
into the same VM came with incompatible dependencies to the same Jar. 

Fortunately (or not), Java could be tricked to load *both* versions of the same classes. This
worked as long as both versions never met, e.g. both versions were only used purely *inside*
the different components that needed them. Otherwise, you could run into *apples not being apples*
because the first were instances of Apple1.0 and the latter of Apple2.0 and the VM
threw Runtime Exceptions.

```
Runtime Exceptions are, for all practical purposes, like Segmentation Violations in your C
program. A crash. One can catch them in Java, but your application is then in an undefined
state and you cannot be sure that your customer *really* ordered those 100 refrigerators.
```

Smart people in major companies realized at the end of the 90s that they had a problem. And
this became one of the motivators for the [OSGI Working Group](https://www.osgi.org) to develop
a standard for Java components *inside* a deployment to manage dependencies, versioning and
lifetimes.

OSGI is a huge standard because it needs to manage a highly complex situation. If you get it
right, it works nicely, but when it does not it can be very tricky to analyze and even more
tricky to resolve a situation.

## Outcomes

Having breezed over the Java deployment situation (and probably missed several things), what 
is there to learn in regard to the social aspect of this technology?

The obvious is that in the first ten years of Java, the development of shareable components
outside the Java Runtime was simply not an issue. The
focus was on managing the situation where a system company (Sun Microsystems) provided
infrastructure to Enterprise developers for their applications (my interpretation).

There was no code sharing between enterprises at the time. There was not even a sourceforge yet. 

However, by making the technology open and available to everyone, Sun Microsystem did
encourage the propagation of open source beyond the *NIX crowd. A lot of Java open source
projects found their home at the Apache Software Foundation and are still active there. 

Open Source Java frameworks proved their value and were picked up by Enterprises because
of it. This helped break the mistrust in those companies that was very widespread.

The social consequence of that success, the lack of management in third party developments,
left problems in deployments. Recently revisited during the `log4j` security issues.

## Conclusion

If you like Java as a language or not, it seems indisputable that it played an important
role in the development and spread of free code sharing and helped open source (intentionally
leaving out the Oracle acquisition and licensing disaster here).

Unfortunately, it never solved the aspect of how exactly component developers work together 
with deployment in an innovative, productive way. It did not impose *enough* discipline
to make the integration easy. Every deployment situation was unique, a highly
complex system like OSGI became necessary to manage all possibilities.

Compared to this, *NIX distributions like Debian, Ubuntu, Alpine, FreeBSD and the myriad 
others **reduce** complexity in deployment. Although they are many, the variations in are
several orders of magnitude lower than in the Java case.

If code has outdated dependencies, it will not become a package on the distros. If it already
is and is not maintain those, it will get questions or be replaced.
The task that integrators do is important and, in its result, beneficial to developers
and users alike.


