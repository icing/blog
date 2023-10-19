# Apache httpd 2.4.58

On 2023-10-19 the project released version 2.4.58 of Apache httpd. 
3 CVEs have been fixed, 2 low and 1 moderate. I describe the changes
done by me below.

## CVEs

The interesting one is CVE-2023-45802 which I described in 
[my blog about the HTTP/2 Rapid Reset](./h2-rapid-reset.md).

## Changes

### `mod_http2`, HTTP/2 frontend

#### WebSockets

I added support for bootstrapping WebSockets via HTTP/2, as described in RFC 8441. A new directive 'H2WebSockets on|off' has been added. The feature is not enabled by default for compatibility reasons.

If you host WebSockets applications with Apache as reverse proxy, you should be able to turn this on and allow HTTP/2 clients to use this without further changes in your application. More information available at [How to WebSocket](https://github.com/icing/mod_h2#how-to-websocket).

This work has been sponsored by [Axis Communications](https://www.axis.com/). Many thanks!

#### Early Hints

We support the HTTP 103 "Early Hints" from the beginning. Now, you can configure explicit header values that should be send to the client. See [How to Early Hint](https://github.com/icing/mod_h2#how-to-early-hint) for instructions.

#### Proxying

If you use Apache as a forward proxy, you can now enable HTTP/2 on it as well. You need to configure `H2ProxyRequests on` in such a setting.

#### Tweaks

The server lets you configure how large HTTP/2 DATA frames should be. Use `H2MaxDataFrameLen n` to limit these to at most `n` bytes of payload. By default, we use the maximum size possible (around 16KB). It depends on your application if smaller sizes are more beneficial.

#### Fixes, Fixes and Fixes

See the `CHANGES` file for all the bugs we fixed. Several around stability, some in statistics/log reporting.

### `mod_md`, ACME

You can now configure the command used for `DNS-01` challenges per Managed Domain. Also, there is a new `MDChallengeDns01Version 2` setting that will give `DNS-01` commands in `teardown` the challenge to remove from the DNS. This is useful when you have more than one ACME client using this challenge type on a DNS domain.

You can restrict the "auto-matching" of the module with `MDMatchNames all|servernames`. Giving you better control when using the same `ServerAlias` name in several virtual hosts.


@icing@chaos.social