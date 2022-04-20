# Vulnerabilities Blabbing

In this blog I blab a bit about vulnerabilities in software. There is no
definite conclusion to be had here. Should you chose to read it, don't 
blame me for wasting your time. You have been warned.

## Feasible Security

In our quest to make secure systems, the implied assumption often seems to be
that we should strive for making complex software machinery that has a defined
output for all possible kinds of input. That is, we can be "sure" it does the
"right" thing.

Formal software verification is one approach taken. One regards a piece of
code as a mathematical function and verifies that for all inputs the function
does what it should do (the "right" thing). This is very useful, if the code
can be described in its function from an outside perspective. Cryptographic
algorithms implementation are an excellent target for formal verification.

When it comes to networked code, this proves to be impossible. We cannot describe
the function of the internet, so we are unable to verify that the software 
running on the internet does the right thing. 

In truth, this is a desirable feature. We want the internet to surprise us
with something that we did not anticipate. Hopefully in a joyful way. But
people also seem to like doomscrolling or else news outlets would shape
their articles differently.

## Unverifiable Security

Bruce Schneier, the godfather of security, blogged recently about [Undetectable Backdoors in ML](https://www.schneier.com/blog/archives/2022/04/undetectable-backdoors-in-machine-learning-models.html). Machine Learning models
can be primed with special behaviour that do not prevent their normal operations. E.g. a face recognition
system would work as you want it to, but on showing it something special, it would always let you pass.

And this is "Undetectable", e.g. it is computationally impossible to verify if an ML model has such
a backdoor or not. It is not paranoid to assume that the NSA is very interested.

## Vulnerability, who are you?

So, what is a vulnerability?

1. *Unfit for Purpose*: we were able to clearly state what we expect software to do and it failed to do that. In practice, this is often a missing test case. So you add that test case, fix the software and it will never appear again.
2. *Garbage in, Garbage out*: input is "garbage" if it was never defined what the software was supposed to do with it. The outcome is regarded as undesirable. The obvious fix is to *extend the definition of purpose* and go to square 1. 
4. *Unreliable*: the software does what it should do and denies unwanted input. But that reaction is degrading its usability. E.g. answers to correct input take too long. A "Denial of Service".
3. *Unstable*: a software does what it is supposed to do, but not in all situations. "It works on my machine" is the cliche for this. Formally, one could say that this is a lack in the definition of possible circumstances. And as a result, a lack of test cases. In practice, there is a lot of hindsight involved.

These categories are not mutually exclusive. Since systems are interconnected, the denial of service in one component may trigger an unfitness in another.

Often, vulnerabilities appear when crossing systems and their definitions. System A handles user input, does some checks and forwards it to system B that only handles URLs. If A and B do not fully agree on what a valid URL is, both may be correct in their own domains, but their combination is vulnerable.

This is a practical example. As you might know, [*there is no common definition of what an URL is*](https://curl.se/docs/url-syntax.html) on the internet.

## Causation

So, what causes a vulnerability?

Someone finds it.

The root cause of all CVEs is that someone detects behaviour in a piece of software classifies as a vulnerability. E.g. that does some (or a lot of) harm or, *now that it is known*, could be exploited by someone to do harm.

(In the beginning of the internet, software makers tried to manage this situation by *making vulnerabilities unknown again*. Which proved disastrous and is now commonly regarded as the wrong approach.)

Of course, the vulnerability existed since the release of the software, some would argue. But was there Log4Shell in 2015? Yes and no. This quickly becomes philosophical. For practical purposes, Log4Shell is a vulnerability from 2021. It affects installations as far back as 2013, sure.

If you want to argue why this was not prevented by the people of 2013, so that we do not have to deal with it now. Well, that quickly leads to silliness. 

We could also discuss why the people of Ancient Rome did not do anything about the Corona virus. They did not know about it? They did not have the technology of today? Exactly my point.

## Weakness

Almost all software has weaknesses. The formally verified, functional pieces may be the only exception. But they are verifiable present in all others. Because vulnerabilities are detected every day every where.

How weak a piece of software is depends on a variety of things:

1. If formal verification is possible.
2. If its purpose is fully defined and understood.
3. If its fitness is constantly tested on changes.
4. How vulnerabilities are used to improve the points above.

How this is done depends heavily on project/company culture. Availability of infrastructure and other resources like time and money play an important role.

The technology used contributes heavily to the feasibility of this. The programming language is a dominating factor. The correctness of C code is way more tricky to verify than Java or Rust or Go. And thus, the push to use the later more.

Less weakness, less vulnerabilities. No one can argue with that. However, weakness is not the only factor in selecting a technology. The technology of the current internet has been chosen in the past and we need to deal with the situation. Until we have come up with better solutions.

Language wars are and always have been mostly silly.

## And Beyond

The growing numbers of vulnerabilities found paint a rather dark picture. Or doesn't it?

Well, for one, we have so many vulnerabilities because so many people are looking. There is
fame and money to be had. Also, there are tools available that help finding them. 

Don't expect us to run out of vulnerabilities to find soon. Or ever. As mentioned above, we can
now design undetectable backdoor, e.g. vulnerabilities. Expect security research to continue even after we replaced the last C code on the net. People fuzz Rust code successfully as well.

Also, security research can be seen as an investment after the fact. There has been, compared to the numbers of people
working on it, little investment made in the development of the net. That is why it spread so fast and easily. And now companies invest in the quality of those things.

Fortunately, the vast majority of vulnerabilities are of minor importance. Not unimportant, but not really threatening
our lives. Few are and those now get the attention they deserve. Starting with the OpenSSL heartbleed as a wake-up call over Log4Shell, Spectre and others to the latest [Java Crypto Oops, e.g. Psychic Signatures](https://neilmadden.blog/2022/04/19/psychic-signatures-in-java/).

After all, life on Earth is 2 billion years old and we still have virus infection vectors that allow re-programming our cells.