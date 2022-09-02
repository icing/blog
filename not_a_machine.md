# Not a Machine

Someone recently said on IRC "You're a machine!". This was intended as a compliment on
the speed of my work and I took it as such (compliments are nice!). But it somehow stuck
in my head and now it wants to get out. It's Friday and I am winding down for the weekend, so
I thought I'd give it a shot.

## Machines

A machine is, generally speaking, something made for a *purpose*. It is a *good* machine if
it does that whenever we want it to. Bonus points for speed and efficiency.

Computers are often referred to as machines. However, they are more *meta* machines, e.g.
a machine that allows to build new machines, we call programs or applications. A computer without 
applications is of little use to anyone but a programmer. And even they
prefer computers with pre-installed machines they call operating system, compiler, linker etc.

Now, the really surprising thing is that none of our ancestors, e.g. my grandparents and before, 
would have regarded a computer as a machine. Because **it does not do anything!**. It does not
lift, hammer, drill or mill. It does not feed the horses and neither does it plow the fields. It 
is totally useless! To them.

So, what changed? We changed.

The early examples for the usefulness of computers were about *accounting* (halls of people
calculating what is today a single spreadsheet) and *moon landings* (the famous image of who
programmed for the Apollo mission). The use case everyone could maybe relate to was *printing*.

Is that why everyone has a computer nowadays? Certainly not.

We are excited because they bring our minds closer together, let us share our ideas and creativity
better, find others with common interests, have access to knowledge, etc. I have not data to prove
this, but I would say we are all using our brains much more than previous generations did on 
average.

Now we find that we are also getting very close to edge lords, lies and hate. We have not found
a good way of dealing with that. But the overall effect is still positive, I'd say.


## Programmers

Programmers are brain users. While typing on a (mechanical!) keyboard can be very soothing, the
real kick comes from creating *useful* machines. There are at least two aspects here:

1. it was tricky to create the machine and required some nice thinking. How smart we are!
2. the machine now does something we otherwise would have to do ourselves. Something *mind-numbing*. Now the computer does it, fanning happily. What a pleasure!

Programmers often talk about the first. And I can relate to that as well. But most of my personal satisfaction comes from the second.

You see, in order to do something innovative, I really need all the help I can get. Something
to watch my sloppiness and inexactitudes. Reading a text of mine
ten times or more will **not** allow me to discover the spelling mistake in the first sentence. I need help!

And that is why I like the computer taking over all those things I am not really good at.

### Testing

Automated testing is the prime example for this. Writing test cases is *satisfying*. I know people
who write good machines without them. Some. Well, very few, actually. But somehow they can do it. But
I can not. 

In need tests to produce something good. Because when they work, I *know* that what I changed is
at least as good. When I add test cases, I am sure that it is *better* than before.

When the test suite is really covering the essentials, then I am ready for the real pleasure 
in programming: ripping everything apart and putting it together in a new, improved way. Making
it simpler or more efficient or both. Fun!

### Debugging

I hate it. In fact, I almost never do it. It is one of those *mind-numbing* things. To step
trace through a program, inspecting variables. Bleh! Sure, it let's you find out what is going on.
But on the next bug, you'd have to do all this again! Yikes!

### Logs

Give me logs instead. In case they do not show the problem, add log statements to the code. On
a recurring problem, raise the log level and you will find out what happened. Make your test
suite flexible in running particular tests with higher log levels. Easy!

One reason I love using `pytest` is that it allows to run any particular test case alone. Giving
you logs of just that run. How much time that saves.

If logs are large and not easy to read, write a log analyzer that shows you aggregated data. Automation!


## Not at all

So, I am not a machine at all. I just like automation. For the purpose of letting me
being good and efficient at all the other things. And for catching all the mistakes I do
very early and very quickly. So, at best, no one else will see them. Only the working results. 

And they may say "wow, you are a machine!", but I certainly am not.






