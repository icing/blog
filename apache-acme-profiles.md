# Apache ACME Profiles

As you might have heard, [Let's Encrypt will introduce certificates with shorter
lifetimes in 2025](https://letsencrypt.org/2025/01/09/acme-profiles/). They will
not go away from the 90 days they give you now. They'll add certificates with a 
duration of 6 days in addition.

**If you do nothing you will continue to get certificates as before!**

This out of the way, what do you have to do to get the shorter ones?

### How to get other Certs

First, as of now, you will not be able to use this new feature, as Let's
Encrypt is gradually phasing that in this year. You may use it on their
[staging (e.g. test) servers](https://letsencrypt.org/docs/staging-environment/),
however.

LE adds "Certificate Profiles" to the ACME protocol. In the "directory" web 
resource, they announce a "profiles" field in the "meta" section. That lists
the profile names and a short description.

Let's try this:

```
> curl https://acme-staging-v02.api.letsencrypt.org/directory
{
  "XS46oHzRdCU": "https://community.letsencrypt.org/t/adding-random-entries-to-the-directory/33417",
  "keyChange": "https://acme-staging-v02.api.letsencrypt.org/acme/key-change",
  "meta": {
    "caaIdentities": [
      "letsencrypt.org"
    ],
    "profiles": {
      "classic": "The same profile you're accustomed to",
      "tlsserver": "https://letsencrypt.org/2025/01/09/acme-profiles/"
    },
    "termsOfService": "https://letsencrypt.org/documents/LE-SA-v1.4-April-3-2024.pdf",
    "website": "https://letsencrypt.org/docs/staging-environment/"
  },
  "newAccount": "https://acme-staging-v02.api.letsencrypt.org/acme/new-acct",
  "newNonce": "https://acme-staging-v02.api.letsencrypt.org/acme/new-nonce",
  "newOrder": "https://acme-staging-v02.api.letsencrypt.org/acme/new-order",
  "renewalInfo": "https://acme-staging-v02.api.letsencrypt.org/draft-ietf-acme-ari-03/renewalInfo",
  "revokeCert": "https://acme-staging-v02.api.letsencrypt.org/acme/revoke-cert"
}
```

So, you see `classic` and `tlsserver` where they [link a blog describing these profile](https://letsencrypt.org/2025/01/09/acme-profiles/).

### Apache ACME

With the release v2.5.0 of [mod_md](https://github.com/icing/mod_md/releases/tag/v2.5.0), the Apache ACME
module supports profiles as well. When installed in your server, you would configure something like:

```
<MDomain "profile-test.mydomain.net">
  MDCertificateAuthority https://acme-staging-v02.api.letsencrypt.org/directory
  MDCertificateAgreement accepted
  MDProfile tlsserver
</MDomain>

<VirtualHost *:443>
  ServerName profile-test.mydomain.net
  SSLEngine on
  ...
</VirtualHost>
```
Reload the server and it will contact the Let's Encrypt staging server to get a 6 day certificate for you.

**Caveat**: these are early days. Thinks might go "sproink!", names might still change, etc.

![tire profile](./images/profile.jpg)

### Reliability

Is it a good idea to get 6 day certificates? For most people probably not. The main advantage of 
them is that, should a certificate escape your control, there are only a few days the baddies may use it.

However, there are also only a few days that your server has to renew them. With the default settings,
Apache will renew them when 33% of life is left, so 2 days before they expire. That is not a lot of
room should things go horribly wrong.

You choose.

### Disappearing Profiles?

If a profile you configure is one day no longer supported by an ACME server, Apache will just not use 
it and you get a default certificate. I chose this behaviour since, I believe, people do prefer to
have any certificate rather than none at all.

But you can override this. Configuring `MDProfileMandatory on` will make the renewal process fail
unless the server supports your profile.

### Conclusion

The 90 day certificates served many, many people well in the past and will continue to use them. But when
certificate revocation (which does not really work in practise) is a topic for you, the 6 day certificates
are a welcome new option. Now you can start playing around with them using Apache ACME and be ready, later
this year, to use them in production. 
