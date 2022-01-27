# TLS-ALPN-01 Trouble

Let's Encrypt found some trouble with their certificate validation that used
the `tls-alpn-01' challenge scheme of the ACME protocol. It seems severe enough
that they decided to **revoke all certificates issues with this method on January 28, 2022**.

This is rather drastic, but they usually know what they are doing. But still, a 5 day
heads up on this has many people scrambling. They [offer help on renewing your certificates](https://community.letsencrypt.org/t/questions-about-renewing-before-tls-alpn-01-revocations/170449) on
their community forum. Which is a nice place with many helpful people.

If you use Apache ACME (`mod_md`) to manage certificates, below is some advice on
how to check if you are affected and what you can do.

## Are you affected?

### maybe not

Apache ACME, like EFF's `certbot`, uses the `http-01` challenge method by default. If
your server listens also on port 80, most likely that method is used for your certificates
and you will have to do nothing.

But how can you be sure?

### better check

Let's Encrypt states that this only concerns certificate issue in the last 90 days using
the `tls-alpn-01` method. Since their certificates have a lifetime of 90 days and Apache
ACME renews accordingly, it is relevant for all your LE certificates.

Some affected people have received an Email from them, but those seems not necessarily 
always be complete. Having received no email seems not a safe indicator.

Luckily, Apache ACME persists everything in the file system, including a log of the
renewals. This is the `MDStoreDir` which is by default the directory `md` in your
server root. On many systems this is `/etc/apache2/md`. If you have a recent Apache install,
just open a shell there and:

```
# Apache httpd 2.4.48+
md> fgrep -r 'message-challenge-setup' domains
...
./domains/mydomain1.org/job.json:        "type": "message-challenge-setup:http-01:mydomain1.org"
./domains/mydomain2.com/job.json:        "type": "message-challenge-setup:tls-alpn-01:mydomain2.com"
...
```

and you can see which ACME challenge type was used to verify your domains with Let's Encrypt.

In the output above you can see that `mydomain1.org` was verified by the `http-01` method
and `mydomain2.com` used the problematic `tls-alpn-01`. This means that your certificate
for `mydomain1.org` is fine and you should renew the one for `mydomain2.com`.

In older versions, logging was a bit different and you might use:

```
# Apache httpd 2.4.46 and maybe older
md> fgrep -r 'tls-alpn-01' domains
...
./domains/mydomain2.com/job.json:        "detail": "Setting up challenge 'tls-alpn-01' for domain mydomain2.com"
...
```


## How to Renew

### brute force 

The brute force thing is to just delete everything in your `md` directory and reload the
Apache server. But that is rather drastic and loses you also the previous information
that is kept in the `archive` sub directory. Not recommended.

But if you want to renew everything, you can just rename the `md/domains` directory to
something like `md/domains.bak` and reload.

You can also do this for a particular domain only by moving `md/domains/mydomain2.com` to
`md/domains/mydomain2.com.bak`.

**This means your site(s) will be temporarily offline**, as they'll use fallback certificates
that browsers will not trust while talking to Let's Encrypt.

### Uninterrupted

The better way is to tell your Apache to renew early. Since Let's Encrypt certificates are
valid for 90 days, just tell the server to renew certificates when less that 85 days are left:

```
MDRenewWindow 85d
```

and reload the server. It will start right away to renew your certificates that are older than
5 days. Wait until it is done (see [`MDMessageCmd`](https://github.com/icing/mod_md#mdmessagecmd) on how to monitor this)
and reload the server again. Your new certs are installed and used. Done.

**Remember to change the renew window back** or your server will renew every 5 days from now on.

If you have only a few domains affected and do not want to renew all of them, you can configure
the window for a particular domain:

```
<MDomain mydomain2.com>
  MDRenewWindow 85d
</MDomain>
```

so all other domains stay on the default.

# Conclusions

While this short-notice change by Let's Encrypt certainly sucks, they can hardly be blamed for
the excellent and free service they offer. Mistakes happen.

Other servers, like Caddy, have implemented auto-renewal when they detect a revocation. This is
certainly a great feature which I'd like to have in Apache as well. I just haven't gotten around
to implementing this.

I hope this blog helps you in avoiding issues with revoked certificates and unnecessary struggle
if you can check that you are not affected.
