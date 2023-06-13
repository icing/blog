# ACME Security

You might have heard about the ACME 0-day exploit in [acme.sh](https://github.com/acmesh-official/acme.sh) (fixed in the latest release) and Matt Holt, who discovered it, has written an excellent [blog about it](https://matt.life/writing/the-acme-protocol-in-practice-and-reality), where you can read all the details.

In the second part of his blog, he gives general security advice and opinions on technology in regard to ACME implementations and deployments. Given that he and me and on different spectrum in the Holy Campaign on [Memory Safety](https://en.wikipedia.org/wiki/Memory_safety), I feel some points misrepresented or left out.

Important aspects in ACME for [Apache httpd](https://httpd.apache.org/), where I did the implementation, are not mentioned. Let me add to Matt's post.

![ACME Bar and Pizza](images/ACME_logo.png)

## Isolation

Bugs happen to everyone. Catastrophic bugs also happen to everyone. There was the infamous Heartbleed, there was Log4j and now there is acme.sh. The first was in a memory-unsafe language, the other two happened with memory safety. This means, we have not found a technology where exploits do not happen. We argue about likelihoods. Those are good and worthy discussions, but using memory-safe languages is no excuse to stop thinking about your design.

The first principle in implementing code talking to outside agencies is **Isolation**.

Isolate the piece talking to foreigners by giving it the minimal access to the rest of your system that is possible (and usable). Assume that it can be compromised some day. What will your system look like when that happens?

### Example: Neverbleed

[Neverbleed](https://github.com/h2o/neverbleed) was implemented in response to the SSL Heartbleed disaster. It's purpose is to provide *isolation* of your TLS key handling from its hosting application. This allows a webserver to manage TLS connections with foreigners (you, for example) without access to private keys.

It does this by running in a separate process, spawned off at the start of the application. The application thereafter becomes "unprivileged" (more on the below) and talks through pipes with Neverbleed for signing handshakes.

This is an excellent design. It has nothing to do with the C language. It uses the user separation capabilities of the host Operating System. Ideally, every HTTP server would follow it.

### Privileges

Every user account on an Operating System has certain permissions to do things. There are different account types. "root" on Linux being infamous for its god-like rights. It is very convenient (usability!) to run things as root, because almost nothing will stand in your way. But for security, this is terrible.

Especially if processes talk to the outside. **Do not** run your ACME client as root (or another privileged user). Nor do this for your HTTP server, or SMTP/IMAP etc. People who write the default configuration in OS distributions know this, of course, and put in work to make this happen for you. It is always a balance between security and usability and distros vary in the weighing of one vs. the other.

### Unprivileged

When we say "becoming unprivileged", we mean a process that starts as a privileged user switcher over to another user with significantly less privileges, aka. permissions. This is a common pattern to solve the following problem:

A HTTP server needs to read configurations and TLS private keys when it starts, but once it has digested those, it is no longer necessary. So, it switches user afterwards *before* it accepts connections from the outside.

Apache httpd, in many setups, starts as root, configures and then spawns child processes to handle the traffic. Those child processes run as "www-data" user (debian). The user "www-data" has no permissions to read (or even write) your certificates and configuration files or other parts of the system.

This does not solve all security problems, but it is good design.

## ACME Isolation and Usability

Does your ACME installation manage privileges responsibly?

The litmus test for this is: *Do your new certificates get activated automatically?*

If yes, there is no proper isolation. Simple as that. Great usability, though!

See, to do that this the ACME client needs to manage your server process and write private keys to your storage. For that it needs privileges. If it gets compromised, as acme.sh did, terrible things happen. Using a memory-safe language does not free you from carefully designing your overall system architecture.

## Apache ACME

Managing privileges was part of the design of `mod_md`, the module handling ACME in Apache, from the start. The main points here are:

* All communication with ACME server happens in the unprivileged processes (e.g as `www-data`).
* httpd sets up special directories where `www-data` may write new keys and challenge files, the "staging" areas. Live certificates/keys etc. are not accessible.
* When (re-)started, the staging areas are scanned for new, complete configurations. These are read, checked and written into the live configuration then. They are not just copied over. A compromised staging file does not simply become a live one.
* ACME cannot control or restart your server. This is why the activation of new certificates only happens after a reload triggered from the outside.

Users of Apache ACME often ask why the server does not activate new certs automatically by itself. This is why. A usability vs. security trade off.

# Summary

Your overall architecture defines your systems security. The implementation language plays an important role, but it is not the only one. Nailing everything with the "memory-safety" hammer is sweeping aside other, equally important things. Not helpful.

Münster, 2023-06-13
Stefan

(Logo from [ACME Bar&Pizza](https://acmebarandpizza.com/). No affiliation.)






