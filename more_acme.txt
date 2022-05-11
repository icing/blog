# More ACME

ACME is really one of the greater inventions of the last years for the web. Everyone likes
it, there is great support via Let's Encrypt and all the implementations out there. It is
integrated into more and more components. People find new ways to benefit for it. Take as
example the certificates provided for you VPN hosts that tailscale offers.

If you want to name one thing where it struggles, it is the revocation of certificates. While
it has functionality to revoke a certificate, the timely spread of this information to all clients
in the world remains an unsolved problem. Also, servers who use a revoked one often have
no idea that they need to do something.

For the later part, Let's Encrypt is proposing an extension to ACME, the [ACME Renewal Information (ARI)](https://www.ietf.org/id/draft-aaron-acme-ari-02.html). This will tell ACME clients when the CA thinks
they should renew a certificate. Very nice, since ACME clients check with the CA regularly anyway.

However this does not help clients. For them, OCSP is the only real alternative and that
has been not so successful. For various reasons covered in various articles on the web.

!["theme pic"](images/double-decker.png)

## Shorter, Faster

As an alternative to OCSP it has been proposed to just reduce the certificate lifetimes. If a certificate
takes a couple of days to become effectively revoked (OCSP information is cached), why not make
if valid a couple of days only? That would make revocation for most people unnecessary. A compromised
certificate would just wither away anyway.

It would mean that certificates need to be issued more often. If you reduce 90 days to 5, you have
a higher renewal frequency. More work for the CA infrastructure, for CT logs etc. But from
what I heard, this would still be doable.

The real road block here are the reliability requirements that a CA MUST meet to stay
recognized. Should it fail those, it is in danger of being excluded by the browsers.

If a CA becomes unavailable for 6 hours due to a disaster, this should only affect a certain
percentage of customers negatively (my reading of this). If certificate lifetimes are reduced, the 6 hours will affect a proportionally higher percentage of customers. Short enough, and this becomes a death wish.

## More Dices

The likelihood of throwing only 1s (in D&D a certain miss) goes down rapidly if you add dices. In
the recent Apache ACME (mod_md v.2.4.16), you may therefore configure two or more CAs for your
certificates.

```
MDCertificateAuthority letsencrypt buypass
```

would use Let's Encrypt a number of times and when switch to buypass. If that also fails
for some time, if goes back to Let's Encrypt etc. etc. 

You may configure how many attempts are made before the renewal falls over to another CA. The
defaults should make that about half a day of attempts. It is not exact since Apache ACME also
jiggles retry delays +-50% all the time.

If I remember my math correctly, this means if a CA had a 1% failure rate, 2 CAs would
have a 0.01% probability of both failing.

In many situations it does not really matter, where exactly your valid certificate is coming from. Having
more than one CA has only upsides then. And to be clear: this behaviour in Apache ACME does not put additional burden on the CA. Apache does not get *more* certificates. So there is no real downside for a CA either.

## Chill

All this does not really matter *right now*. The common 90 day lifetimes leave ample of room to
survive prolonged CA failures. 

However, if your are not hooked to a particular one, why not
configure more?