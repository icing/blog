# Apache httpd 2.4.49 release

(This is the first time I was release manager (RM) for Apache httpd. It would
be more aptly named *release caretaker* since it is mainly shuffling all the
puzzle pieces around to make the stars align.)

Some highlights from this release and what it brings you as a user. I can't touch
on everything since the improvements and fixes are done by many people. The complete
list is, as always, available in the [CHANGES 2.4](https://dist.apache.org/repos/dist/release/httpd/CHANGES\_2.4).

## CVEs

There are 5 security related fixes in this release. 2 are what is nowadays called a "classic C vulnerability", e.g. one out of bounds read and one write. One could be triggered from the outside,
the other on configuration data.

A third was a NULL pointer dereference which, triggered from outside, would make a child process crash. In my view, it is not a C weakness as such, as failed assumptions are present in most other languages and by default cause a process to terminate. The result is the same, people can DoS a server easily.

The fourth is a failed check on input which could let proxy requests open connections to servers you did not intend to allow. This leads into the realm of exploit vectors to other components in your server infrastructure. Apache httpd itself is not negatively impacted, but security is broken. These are the kind of security issue that no fuzzing or memory safety will solve, unfortunately.

The fifth, and that was already available and shipped as patch to 2.4.48, is a similar weakness in the HTTP/2 code. The HTTP/2 protocol did not check incoming method names (GET, POST, etc.) for correctness. When such a request was send via mod\_proxy to another server, it could possibly exploit a weakness in that server. Since Apache is used to front legacy systems, that may not be ready for such input, this was a unlikely, but nevertheless real scenario.

## Improvements

### core

mpm\_event now handles graceful restarts with open, idle connections faster. Connections where
the other side was no longer responding at all could lead to a timeout of 5 seconds, before
a child process was forcefully terminated. Also, mpm\_event fixed one issue where a child process was not 
stopped  on a graceful restart.

[StrictHostCheck](https://httpd.apache.org/docs/2.4/en/mod/core.html#stricthostcheck) is a new
directive that, when enabled, will deny any requests that do not exactly match a ServerName/ServerAlias
you configured.

The check on protocol components, e.g. a proper 'method' name, was changed so that all HTTP protocol
handlers (http/1.1 and http/2) use the same code. This addresses the CVE mentioned above.

The infrastructure to allow for more than one TLS/SSL handling module loaded is now complete. 
In 2.4.48 we shipped that for frontend connections and now it also supports proxy connections.
As an example, you can now serve some ports/backends by mod\_ssl and others by [mod\_tls](https://github.com/abetterinternet/mod_tls).

### mod\_ssl

You can configure the module to export private key material with SSLKEYLOGFILE and have a nice
integration with [Wireshark](https://www.wireshark.org) or other network analyzers to look into
your SSL connections.

The ALPN handling to backend servers has been tightened. When a certain protocol is requested,
exactly that is expected from the other end.  The connect will now fail if the backend does not agree to it. An exception is 'http/1.1' where the backend may not send back any protocol (as it does not implement ALPN). 
This change prevents certain attack types where responses from another service with the same certificate are used to answer a connection.

### mod\_unique\_id

Had a change in 2.4.48 that could result in duplicate ids giving to connections under load on Windows. This
was mostly reverted with some improvements.

### mod\_md

Improvements in regard to 'tls-alpn-01' challenge handling. Also, certificates and keys are checked
to actually match each other before being activated from a staged setting.

### mod\_proxy, mod\_dav

Several improvements here where I do not really know the details about.
