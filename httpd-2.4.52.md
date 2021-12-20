# Apache httpd 2.4.52

On 2022-12-20 the project released version 2.4.52 of Apache httpd. 2 CVEs have been fixed, several bugs have been addressed. The server has been tested with OpenSSL 3.0.1. A new, experimental for Rust based TLS and ACME EAB support has been added.

In detail:

## CVEs

### mod_lua

A buffer overflow, a classic `C` weakness, has been identified in multipart POST handling inside `mod_lua`. If you do not have that module loaded into your installation, you are not affected. The overflow can be triggered by an outside request. No exploit is known, but certainly someone will analyze and craft one.

### forward proxying

If you use `mod_proxy` in a forward configruation (`ProxyRequests on`), it is possible to trigger a Null-Pointer dereference in the code. This is a denial of service attack surface. If your installation has no proxy forwarding, your server is not affected.

# OpenSSL 3.0.1

We added OpenSSL version 3.0.1 to our CI testing and verified that this combination works for us. With version 3 containing quite some changes in OpenSSL, and SSL configurations via `mod_ssl` being highly customizable, deployment deserves some attention. We do not want to discourage anyone from using it, but it has not the same production maturity as OpenSSL 1.1.x.

# Rust based TLS (experimental)

Sponsored by the Internet Security Research Group (ISRG, the parent company of Let's Encrypt), an alternative to `mod_ssl` has been developed in their [Prossimo](https://www.memorysafety.org/about/) project and was donated to the httpd project.

[`mod_tls`](https://httpd.apache.org/docs/2.4/mod/mod_tls.html), the new module, uses [rustls](https://github.com/rustls/rustls), a TLS library completely written in Rust to provide TLS v1.2 and v1.3 for incoming `https:` and outgoing proxy connections.

While the integration into the Apache httpd infrastructure is still written in `C`, all TLS related operations are handled by Rust code. Compared to `mod_ssl` the functionality is limited, but complete for common `https:` serving setups. More about this in the [module's documentation](https://httpd.apache.org/docs/2.4/mod/mod_tls.html).

The status as `experimental` means that the project makes no guarantee to keep this module backward compatible in future releases. Your feedback on if and how you make use of this will influence future direction in integrating memory-safe code into the server.

In our testing, we see promising results in regard to efficiency and performance.

# Bug Fixes

Non-security related fixes have been made, as usual, which are described in the `CHANGES` file. 

A common theme are fixes in regard to dynamic traffic handling, e.g. process creation and reductions involving proper connection shutdown and lingering close. It is easy to start a server, but quite hard to shut down resources and keep everyone happy.

# ACME EAB Support

The ACME standard allows a CA to require special information by clients to connect them to existing accounts. This `external account binding` (EAB) feature is now supported by Apache ACME in `mod_md`. Please see the [here](https://github.com/icing/mod_md#a-key-to-bind-them) for more details.

This work has been supported by [Sectigo](https://sectigo.com) and information on the integration of Apache ACME with their Certificate Manager is expected early next year.

# Not Included

Well, I wanted to deliver improvements in the HTTP/2 implementation. Those are in trunk and released on [mod_h2's github repository](https://github.com/icing/mod_h2), but an issue was detected that led to an infinite loop, hogging cpu. This has been fixed in v2.0.2, but some more testing in a production environment seems wise.

Luckily, there are people like @codeyy and @Adam7288 who help with that. Invaluable!