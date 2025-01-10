# curl SSL sessions and early data

In  the upcoming `curl` version 8.12.0 we'll add the *experimental* feature of SSL session import/export. 

You can specify a file where sessions are imported from at the start of `curl` and to which they are exported again once `curl` is done. Like in:

```
> curl --ssl-sessions my-session-file https://curl.se
```

This will work for all TLS backends that support SSL sessions in a format suitable to storage: GnuTLS, OpenSSL (and variants), wolfSSL, bearssl, mbedTLS.

Now, what is this good for? I'll explain some TLS details below, but who wants to read walls of text if they can look at an image?

***Update***: I added measurements for "Time To First Byte" which explains observed numbers better.

### Benchmark (ymmv)

I cannot predict how this will work on your system. This very much depends on the latency and bandwidth to the server. I did measurements from my machine to `curl.se` (fronted by Fastly's CDN). I did 100 runs of 

```
> curl -s https://curl.se -o /dev/null -w '%{time_appconnect} %{time_total}\n'
```
Then I added `--ssl-session file` and did 100 runs. Finally, I added `--tls-earlydata` (my curl is built with GnuTLS where curl has early data support). And then I did all of it again with `--http3-only`.

![Subjective Measurements of curl TLS sessions and early data performance by author for curl.se](./images/curl-sess-early-curl.se.png)

The `time_total` is the overall time from when `curl` starts connecting to when it has downloaded the ~10KB of the page. The transfer time stays basically the same when importing sessions and is noticeably faster when early data is used. Now, `curl.se` has a ping time of about ~18ms for me. Let's try a host somewhat further away, like `nghttp2.org` with a ping time of ~260ms:

![Subjective Measurements of curl TLS sessions and early data performance by author for nghttp2.org](./images/curl-sess-early-nghttp2.org.png)

We see that HTTP/3 saves one round-trip in each case. That is the initial TCP connection establishment. But when we use TLSv1.3 early data on HTTP/2, we compensate for that. HTTP/3 is still faster in delivering the first byte of the response, but the total time for the complete transfer can be better in HTTP/2. This depends on the overall latency of the connection, where we seen that QUIC+HTTP/3 can show what it is made for.

### SSL Sessions Effect

You can think of an SSL session as a token a server sends to a client: "Use this to contact me again and I know it is you and you know it is me!". TLS speak is that this "resumes" the previous session.

When resuming a session on a new TLS connection, the handshake is kept smaller. Specifically the server will not send its certificates (and the chain certificates) again. The client had already seen those.

This saving of bandwidth has no effect in my measurements as the bandwidth is large enough. Sending a few KB of certificate data makes no difference. On a mobile network, this might give more benefits.

The picture really changes when the server supports "Early Data" in an SSL session (TLSv1.3 is needed for that). And this means...

### Early Data Effect

TLSv1.3 sessions (or more precisely named "session tickets") can announce "Early Data" support by the server. Normally in SSL, a client can only send application data (like a HTTP request) *after* the handshake is done. 

With TLSv1.3, the client may - when resuming a session - already send data *before* it has seen any reply from the server. This saves a round trip. The server can start processing the HTTP request right away.

And this is why the benchmark shows a significant improvement with early data. Because I have some latency to curl.se from my home. And this explains also why *you* may see different numbers.

(Note that early data is only implemented for GnuTLS for now. We'll add support for that in other TLS backends in the future.)

### Security Aspects

Now, why doesn't `curl` do this automatically all the time? Well, there are some security aspects to consider. The session file contains data that you might not want to have stored automatically on a machine where you use curl.

`curl` uses salted hashes instead of hostnames in the file. This prevents anyone who can read the file to easily see what hosts you have been talking to. This works similar to `ssh`'s 'known_hosts' file, if you are familiar with that. 

But the salted hash does not prevent someone from checking if you have connected to `www.furries.net` in the recent past. Or try a brute-force attack on guessed names.

**Also important**, and more relevant, the session tickets itself is stored unencrypted (only base64 encoded). The ticket is coming form the TLS backend used and contains host specific information. GnuTLS stores the hostname and certificate information there. So, by base64 decoding that part of a session file, it is easy to see where the session ticket came from. (Another reason why this feature is experimental for now. We might have the trade offs between usability and security not fully correct yet.)

Early Data has other security implications. It is not as secure as other TLS traffic. It's a trade off. And it depends what kind of requests you do with it. There are many articles about it by people more knowledgeable than me, if you are interested.

### Experimental

We made this feature *experimental* in curl, because we need feedback from our users if the current implementation is good enough. The privacy/security concerns around storing TLS sessions in a file are of special concern. Let us know what you think!
