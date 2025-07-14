# HPACK Bombing Apache

Let's talk a bit about
[CVE-2025-53020](https://www.cve.org/CVERecord?id=CVE-2025-53020),
a moderate security vulnerability in Apache httpd that was fixed with
the recent release 2.4.64. It was found by security researcher
Gal Bar Nahum and reported to the project on June 18th 20205.

Gal did real real research, understanding the protocol, reading our code
and spotting where a client could pierce through our defenses. What
you'd expect from a real security researcher. Not one of the new
AI sloppers clogging projects everywhere.

Let's dive in.

## Part 1, HTTP/2 + HPACK

HTTP/2 has over the years had multiple *protocol* vulnerabilities which is
a bit of a misnomer. The protocol is not vulnerable by itself, but
implementing it in a safe manner turns out to be very hard.

The protocol makes it very cheap for a client to cause a lot of work 
or other resources for a server. In 2015, when it was standardized, 
this was an *intentional* feature. Mobile phones were in the hands
of everyone, mobile data was not as good and page load times were
considered as too slow.

While there is a lot of talk about performance in downloading *responses*,
in order to get those, one needs to send a *request* first. For those
to be fast, they needed to be small and numerous. Servers should allow
100 of them at the same time and compression was added to minimize their
size (via HPACK).

HPACK works so well, that a client can repeat a header it already had
sent on a connection by sending a single *reference byte* again in a new
request. And headers may repeat. So, a client can send a request with
8000 headers using ~8KB of data on the wire.

Gal's proof-of-concept did this using a header with a 4KB long name and
and an empty value. This turns an 8KB HTTP/2 request into a 32MB HTTP/1.1
request. Since a client can send 100 requests at the same time, would
a server naively unpack these into HTTP/1.1, these could consume over
3 GB of memory form 800KB incoming data on a single connection.

A classic, reverse ZIP bomb, you could call it.

![NASA Supernova Image](./images/supernova.jpg)

But Apache is not such a naive server. It has its limits.

## Part 2, Apache's Request Limits

Apache has configurable 
[limits on request resources](https://httpd.apache.org/docs/current/server-wide.html#resource) 
in place for ages to protect against misbehaving clients:

* `LimitRequestFieldsize`: the maximum length of a request header (`name: value`)
   with a default of 8190 bytes.
* `LimitRequestFields`: the maximum amount of request headers allowed, defaults to 100.

These are obeyed by its HTTP/1.x and also by the HTTP/2 implementation. However,
the HTTP/2 implementation had an unintended weakness which Gal discovered: *it
counts duplicate, empty headers as a single header*.

The reason for that was that in HTTP, those are the same. Consider a HTTP/1.1
request like

```
GET /file.html HTTP/1.1
aaaa: 1
aaaa: 1
aaaa: 2

```
is semantically the same as:

```
GET /file.html HTTP/1.1
aaaa: 1, 1, 2

```
and that's exactly what HTTP/2 does when it converts an incoming request into the internal representation.

Now, if a client sends such headers with values, the concatenated values will just get longer
and eventually hit `LimitRequestFieldsize`. The request would be denied, the resources freed. 

But sending *empty* headers, the concatenated field would not grow in length. So, Gal's PoC could
send thousands of empty header repeats without triggering it. But what is the harm? If they all 
are collapsed into a single header, the memory needed for that stays low, right?

Yes, and no.

## Part 3, Memory Pools

The `nghttp2` library that Apache uses delivers request headers as `char *` with a `size_t` length. Those
are not 0-terminated. Since Apache's internal data structures operate on 0-terminated strings, the
HTTP/2 code needs to copy those strings before they could be processed.

The copy was done using an allocation into the request's *memory pool*. The memory pools in Apache's
runtime are a special design. They delay the free of the memory to the end of the request, meaning code
does not have to bother with it. Request processing just allocates, frees happen automatically.

This turned into a foot gun in this case. While the HTTP/2 code folds the repeated, empty request headers
nicely into a single one for processing, *it allocated memory for each repeat*. So, the memory of a PoC
request did allocate the complete HPACK decompressed size.

But at the end of the request, this all got cleaned up. Or did it? Well, again, yes and no. 

Another feature of the 
memory pools is that you can have a hierarchy of them. This is a great feature to avoid memory leaks. If
someone fails to clean up a pool, at least cleaning up the parent will take care of it. Meaning, if you
start with a pool for each connection and make all new pools children of that one, cleaning up the
connection will ultimately release all resources.

Nice. However, by default the parent pool keeps the allocated memory of the sub-pool around to be used
again for the next pool. When the server processes one request after another on the same connection, the
same system memory gets re-used over and over again. Which is good.

But for HTTP/2 with up to 100 requests at the same time, this makes connection pools
up to 100 times larger than those for HTTP/1.1 connections.

And this was what Gal found. Sending many request with repeated, single-byte HPACK header that blow up
large, escape the field limits and you end up with a connection that reserves a serious amount of
system memory. Up until it is closed, which a client can prevent by now and then sending another request.

## The Mitigation

Three changes were done in Apache's HTTP/2 implementation to fix this:

1. Repeated headers are counted against `LimitRequestFields` now.
2. The 0-terminated copy of header names and values is done in a single "scratch" buffer held by the connection.
3. HTTP/2 request pool's system memory is returned at the end of the request. This shrinks memory again to minimum on idle HTTP/2 connections.

## Conclusions

Nice work by Gal Bar Nahum, finding how the combination of seemingly good individual design decisions led
to a denial of service attack vector overall. I believe we need security researchers for that. With the PoC
code it was rather easy to understand the flaw and fix it. But someone had to come up with it first.

The attack vectors of HTTP/2 keep on delivering. It's a classic benefits here/trade-off there design decisions that
were made 10 years ago with, in retrospect (there were concerned voices at the time), too much focus on
client benefits, underestimating the server complexity.

QUIC, and HTTP/3, have addressed some of this (and introduced other attack vectors in UDP space). People
are now considering to bring over the good bits into a TCP based protocol. Depending how that will actually
look in the end, it might become a comparatively easy replacement for HTTP/2. We'll see.
