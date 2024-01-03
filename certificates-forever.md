# Certificates Forever

Apache ACME gives you easy access to certificates via [mod_md](https://github.com/icing/mod_md), as you might know. It handles obtaining and renewing them by the Apache server itself. No other 3rd party scripts needed. There are meaningful defaults and quite some flexibility in letting you control (and monitor!) how it works. But with the recent [v2.4.26](https://github.com/icing/mod_md/releases/tag/v2.4.26) release, we are going one step further.

### Revocations

Certificates may get revoked. By the CA (Certificate Authority) that gave them to you. This may get trigger by yourself (your setup has been compromised and your keys may have been exposed) in which case you already know about it and may have triggered renewals.

But it may also happen for reasons beyond your control. Just 2 years ago Let's Encrypt needed to [revoke many certificates](https://www.theregister.com/2022/01/26/lets_encrypt_certificates/). You might have heard about it in time. Or not. Or the time was not sufficient to update all your deployments. Happily, with [mod_md v2.4.26](https://github.com/icing/mod_md/releases/tag/v2.4.26) your Apache can take care if this situation itself.

![ACME Logo](https://www.ietf.org/media/images/ACME-Horiz-FullLockup.original.png)

### OCSP Stapling

Revocation information is published by CAs via OCSP responders (e.g. HTTP servers) that can be queried for a certificate's status. The original idea was the clients to this, but the usability of that is not good, so most clients do not. Instead "Stapling" was invented, so the server would retrieve the OCSP response and send it to the client on connect. Since all this is digitally signed by the CA, the client can trust the stapling that comes from the server. Usability is good, since no additional connections are needed (which may fail or time out).

`mod_md` implements this for Apache as alternative to the one provided by `mod_ssl`. It offers several benefits like caching and status information. And now it is used to trigger certificate renewals as well. The minimal setup for that would be something like:

```
MDStapling on
```

in your Apache configuration to activate it. If it gets a **revoked** OCSP response, the next certificate check will notice this and start renewing.

### Timing

OCSP responses carry a "best used before" time. That varies across CAs. Let's Encrypt usually gives it 3-4 days. Apache ACME uses this and, by default, tries to get a new OCSP response when a third of that time remains. Which for Let's Encrypt means that OCSP status is retrieved every 2 days.

Since a revocation may happen at any time, you might need to check this more often. This depends on your site setup. Since most clients do not look up OCSP themselves, they will continue to believe the valid status response from before. So, you'll not have failed connections right when the CA revokes. Since more frequent checks place a burden on the OCSP responder (e.g. the CA) be kind in choosing your timings.

If you have decided to go for daily checks, add the following to your Apache config:

```
MDStaplingRenewWindow 1d
```

Now, due to how Apache ACME is implemented internally, the stapling and certificate checks run on independent interval. Certificate checks are done every 12 hours with a variation of +/- 50% (many restart their servers at midnight, CAs see traffic spikes at full hours). So the OCSP revocation may trigger a renewal much later than you'd want to.

Luckily, checking the certificates will find that there is nothing to do most of the time. Once you have the certs, they'll be valid for many days and the checks will not trigger much cpu load and not bother the CA servers at all. So, you may do checks more frequently, like every hour via:

```
MDCheckInterval 1h
```

Please remember that the server will require a reload to activate new certificates. See ["How to Manage Server Reloads"](https://github.com/icing/mod_md#how-to-manage-server-reloads) in the documentation.

### Conclusion

By using the OCSP Stapling in the Apache ACME solution, you'll have a server that reacts on revocations on its own. How timely that happens can be controlled by you. The defaults will work for many sites. As long as your CA, e.g. Lets Encrypt, lives, you will have valid certificates forever.






