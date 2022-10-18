# TLS Server Weirdos

TLS servers are weird. There are not many people who work on them, so few people think about the challenges. Notably, I had discussion over the years with *designers of TLS stacks* that did not understand the needs. Instead of explaining it again and again, I decided to write this blog. Just to drop a link next time.

### Clients have it easy

A client knows where it wants to go, what it expects to see, etc. This is all configured before the connection starts:

```
host = 'www.something.com'
ssl = SSL_new()
ssl.set_options(...)           # optional, protocol versions, ciphers
ssl.set_sni(host)              # the server to talk to, certificate must match
ssl.set_alpn('h2')             # the protocol to talk over the conn
connect(host)
ssl.do_handshake()
# talk h2 to server
```

Most TLS libraries follow this pattern closely in their APIs. And offer it to servers also.

### Fantasy TLS Server

```
accept()
ssl = SSL_new()
ssl.set_options(...)           # optional, protocol versions, ciphers
ssl.load_certificate('somthing.cert')
ssl.set_alpn('h2', 'http/1.1') # the protocols offered
ssl.do_handshake()
if ssl.get_alpn() == 'h2':
    # talk h2 to client
else:
    # talk h1 to client
```

But if you want to handle many connections, this is too wasteful. So TLS libs offer 'Context' objects that can keep a fixed set of parameters and can be prepared in advance.

```
ctx = SSL_CTX_new()
ctx.set_options(...)           # optional, protocol versions, ciphers
ctx.load_certificate('somthing.cert')
ctx.set_alpn('h2', 'http/1.1') # the protocols offered
...
accept()
ssl = SSL_new(ctx)
ssl.do_handshake()
if ssl.get_alpn() == 'h2':
    # talk h2 to client
else:
    # talk h1 to client
```
This shifts a lot of work outside the connection phase and also saves memory. The downside is that you need a separate ip address/port for each domain you want to offer. IP addresses are rare and only a few ports work through firewalls.

### More Realistic Server

But since the client sends the server name it wants to talk to in the SNI extension (see `ssl.set_sni()` above), the server could maybe switch its answers based on that? How would that look?

```
ctx1 = SSL_CTX_new()
ctx1.load_certificate('somthing.cert')
ctx1.set_alpn('h2', 'http/1.1') # the protocols offered on www.something.com
ctx2 = SSL_CTX_new()
ctx2.load_certificate('another.cert')
ctx1.set_alpn('http/1.1')       # the protocols offered on another.org
...
accept()
ssl = SSL_new(ctx1)             # Issue1: which ctx to use, or none?
ssl.do_handshake(on_sni(ssl, x) {
   if (x == 'www.something.com'):
      ssl.set_ctx(ctx1)
   elif (x == 'another.org'):
      ssl.set_ctx(ctx2)
   else:
      raise Error
})
if ssl.get_alpn() == 'h2':      # Issue2: 'another.org' does not talk h2, can we get here?
    # talk h2 to client
else:
    # talk h1 to client
```

There are 2 issues in this code:

1. That settings (or 'ctx') do we start the handshake with? Before we see the SNI, neither one is correct. If we do not select one, the `ssl` we create will have no certificate loaded. Some TLS libraries are fine with that, some fail.
2. How was the ALPN selected? Was it done before the SNI callback or afterwards? Can it select a protocol that is not supported by the selected domain? Again, TLS libraries differ.

Some TLS libraries (no, not all) offer also an alpn callback. So the handshake call may look like this:

```
ssl.do_handshake(on_sni(ssl, x) {
   ...adjust to sni
  }, on_alpn(ssl, alpns){
   ...select alpn protocol
  })
```

However, some TLS libraries do not guarantee the order in which these are called. Or that the interesting properties are available to the callback. A server might be asked to select the ALPN protocol without the SNI being known.

But is it not a really exotic configuration when ALPN differs for SNIs?

### The ACME protocol enters the scene

The ACME protocol (you know, the one that gives you the free certificates?) offers an ALPN specific challenge to verify that you are indeed in possession of the domain name. For that, it connect to the server, gives the domain in the SNI and asks for the ALPN protocol `acme-tls/1`. The server then needs to respond with a *specific* certificate that, most likely, did not exist at the time it started.

The server code would look like this:

```
ssl.do_handshake(on_sni(ssl, x) {
   ...adjust to sni
  }, on_alpn(ssl, alpns){
   if alpns == 'acme-tls/1'):
     if have_challenge_cert(ssl.get_sni()):
       ctxn = SSL_CTX_new()
       ctxn.load_certificate('sni_challenge.cert')
       ctxn.set_alpn('acme-tls/1')
       ssl.set_ctx(ctxn)
       return 'acme-tls/1'
      else:
        raise Error
    ...
  })
```

This only works when the SNI is available at that time.

Ok, that is just ACME. But then...

### QUIC arrives

With QUIC separate sets of SNI and ALPN configuration become the norm. It is highly unlikely that different services using QUIC will use different ports, like it was with TCP. Instead, the ALPN will be get a much more diverse use. While `h3` is the ALPN value for HTTP connections, there is now `DoQ`, [DNS over QUIC, rfc9250](https://www.rfc-editor.org/rfc/rfc9250.html). Others will follow.

### TLS Lib Adjustments

As the separate SNI/ALPN callbacks proved to be a bit weak and made things complicated, OpenSSL in version 1.1.1 added a new callback, the `client_hello_callback`. This one is invoked *after* the complete ClientHello has been parsed, but *before* the TLS libraries really starts the handshake processing.

This allows that callback to inspect *all* values the client sent and configure *everything* in the `ssl` just right for the subsequent processing. This gives a stable API that does not need to change when new TLS extensions are added and are of interest to a server. Not all TLS libraries support this callback yet.

In the Rust world, the [rustls](https://github.com/rustls/rustls) crate **broke** its API to accommodate for this (among probably other reasons). With version 0.20, there is an `Acceptor` that reads the first data from a client and produces a `ClientHello` struct that exposes *some* values like SNI and ALPN. So a server can decide which `ServerConfig` (the ctx) it wants to use for the `ServerConnection` (the ssl).

## Summary

It is not easy to fit new feature after feature into an API over the years. Servers need to make it work, often
with different TLS stacks that vary in capabilities and behaviour. So, you will not be surprised to see many `#ifdef`s
in such code.

I hope this makes you understand some of the weird requirements that server implementors have.

