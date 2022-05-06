# tailscale and Apache

[tailscale](https://tailscale.com) describes itself as "zero config VPN". It has a free tier for personal use. Its business is around giving enterprises their own network using the public internet (my words). 

I gave this a try (it works nicely and the "zero config" is indeed almost zero) after hearing that the Go based [caddy web server](https://caddyserver.com) did [some integration](https://tailscale.com/blog/caddy/) with it. Later, tailscale offered an [auth integration for nginx](https://tailscale.com/blog/tailscale-auth-nginx/) and I wondered what Apache can offer here.

This blog gives a summary of the Apache tailscale integration I did so far.


!["theme pic"](images/tailchase.png "Photo by Tim Mowrer, flickr")
Photo by [Tim Mowrer, flickr](https://flickr.com/photos/mekin/281791343)

## My Own Tailnet

I installed tailnet on my phone, home machine and a virtual private server I rent on a German hosting provider. Let's call these machines `fone`, `home` and `vps`.

With tailscale enabled, I can ping all machines from any one of them. If I switch WLAN off one `fone`, it remains reachable. Also, my `home` machine is reachable without any fiddling on my router. According to tailscale, connections are done peer-to-peer in almost all cases with some backup relay in case you reach a really obscure part of the internet. tailscale has [excellent documentation](https://tailscale.com/blog/how-nat-traversal-works/) on how this all works. And their code is open.

There are some features that you might want to enable on your tailnet:

1. `MagicDNS`: which takes over resolving names for your tailnet machines and forwards all other DNS queries to a provider of your choosing (from a list for now). Some people might (reasonably) object to this, but for me it's fine.
1. `HTTPS`: which gives you a public `*.ts.net` subdomain and obtains 'real' Lets Encrypt certificates for your machines.

### Tailscale HTTPS

Let's say, I chose `headless-chicken.ts.net` as my subdomain, then tailscale can get LE certificates for `vps.headless-chicken.ts.net`, for example. On `vps` runs a Apache httpd, so I add a `VirtualHost` for the domain and the certificate/key and - voila - I can open `https://vps.headless-chicken.ts.net` in my browser (or curl it) and everything is fine and secure. No complaining.

While you can use the tailscale cli to retrieve the certificate and key, copy it to a safe location and configure the Apache with those paths, there is a better way. Coming in the next release of httpd, the [ACME integration supports tailscale certificates](https://github.com/icing/mod_md/tree/master#tailscale).

In our example, we'd do:

```
<MDomain vps.headless-chicken.ts.net>
  MDCertificateProtocol tailscale
  MDCertificateAuthority /var/run/tailscale/tailscaled.sock
</MDomain>

<VirtualHost *:443>
  ServerName vps.headless-chicken.ts.net
  SSLEngine on
  ...
</VirtualHost>
```

When you reload Apache, it will manage the domain just like other ACME domains, obtain the certificate, renew it when necessary and trigger the events. Only, it talks to the tailscale demon on your machine instead of the Lets Encrypt servers (for this very domain).

This is not tied to the machine having a public IP address, like my `vps` does. It can also be done for the `home` machine. 

#### Is it secure?

If you set this up on your `home` machine that is not accessible from the public internet, then yes. Except when you have guests in your WLAN. On the `vps` machine it is definitely not secure, as anyone can fake the ALPN domain name. What to do?

### Tailscale AUTH

I wrote [`mod_authnz_tailscale`](https://github.com/icing/mod_authnz_tailscale) as authentication and authorization provider for Apache httpd. Installing that module allows you to configure which tailscale machines/users can access locations on your Apache.

I did the following:

```
<VirtualHost *:443>
  ServerName vps.headless-chicken.ts.net
  SSLEngine on
  <Location />
    Require tailscale-user icing@github
  </Location>
  ...
</VirtualHost>
```

which allows only connection from one of my tailscale installations to get through. All others get a "403 Access Denied". 

In case your Apache hosts an application that is user based, you may configure something like:

```
  <Location /myapp>
    AuthType tailscale
    Require valid-user
  </Location>
```

That would set `REMOTE_USER` to `icing@github` on my requests. This works for *all* connections from my machines, no matter if you use a browser or curl. SSO without the hassle and pitfalls.

More information can be found in the README of the module.

### Production?

While the certificate provisioning will be part of the next Apache release, the authnz module needs some more discussion the the tailscale people around edge cases. I opened an [issue](https://github.com/tailscale/tailscale/issues/4605) for the question what tailscale considers as a `LoginName` for a client `address:port` tuple. It does not really matter for a default tailscale installation, but once you are in a team or start sharing with other tailnets, this becomes relevant.

## Conclusion

I like the ease of use of tailnet and it is an interesting space to explore. For 'internal' hosting of services that you only want to use yourself or that you'd like to share in a selected group of friends, it


