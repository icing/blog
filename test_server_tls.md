# Local Test Server with TLS

I use `curl` in my tests for the Apache httpd server. I like end-to-end testing and `curl`
is not only the most common HTTP client outside of browsers, it is also very versatile.

Since `https` is now used for everything, tests should use that too. This blog describes
how to do a setup for that and how to use `curl` in this case, so that a full TLS
validation is done and all is pretty close to what the usage in a production setup would be.

[Caveat: there are several ways to do this, this is just one. Also, I use this for 
development, so I want this to work on the local machine without any other infrastructure.]

There are three things to do:

1. Create certificates using your own 'CA'.
2. Setup you Apache server
3. Invoke `curl` with the necessary arguments

![](images/stooges.png)

Let's assume, for brevity, we setup our test in the directory `/test` and our tests
need to talk to 2 hosts, `a.test.net` and `b.test.net`.

## The CA

There are several tools out there to create a local Certificate Authority (CA) and
issue certificates. Since I use Python in my test code (`pytest` is excellent), I
recommend [trustme](https://github.com/python-trio/trustme).

It makes no sense to repeat their documentation here. Basically, look at their
cheat sheet to run something like the following:

```
import trustme

ca = trustme.CA()
a_cert = ca.issue_cert("a.test.net")
b_cert = ca.issue_cert("b.test.net")

ca.cert_pem.write_to_path("/test/ca/ca.pem")
a_cert.private_key_and_cert_chain_pem.write_to_path("/test/ca/a_cert.pem")
b_cert.private_key_and_cert_chain_pem.write_to_path("/test/ca/b_cert.pem")
```

This gives you the following files:

```
> ls /test
ca.pem         # the certificate of your 'CA'
ca.key
a_cert.pem     # the cert for 'a.test.net', issued by the CA
a_cert.key
b_cert.pem     # the cert for 'b.test.net', issued by the CA
b_cert.key
```

## The Server

Setup Apache httpd to use these certificates. We make a local Apache setup that is just for this test
in `/test/apache`.

```
test> mdkdir -p apache/conf apache/htdocs apache/logs
test> cat > apache/conf/httpd.cond <<EOF

Listen 5002                    # local port for https:
Include conf/modules.conf      # load all modules that you need

<VirtualHost *:5002>
    ServerName a.test.net
    SSLEngine on
    SSLCertificateFile /test/ca/a_cert.pem
    SSLCertificateKeyFile /test/ca/a_cert.key
    DocumentRoot htdocs/a.test.net
</VirtualHost>

<VirtualHost *:5002>
    ServerName b.test.net
    SSLEngine on
    SSLCertificateFile /test/ca/b_cert.pem
    SSLCertificateKeyFile /test/ca/b_cert.key
    DocumentRoot htdocs/b.test.net
</VirtualHost>
EOF
```
And start it like:

```
> apachectl -d /test/apache -f /test/apache/conf/httpd.conf -k start
```

And you have a running Apache, listening on `localhost` port 5002 for incoming `https:` connections. The host
has the 2 DNS names defined and maps those to files in `/test/apache/htdocs/a.test.net` and `/test/apache/htdocs/b.test.net`.

[Caveat: this is not 100% copy&paste running, but shortened for brevity.]

## The `curl` Incantations

In our test cases, `curl` is used then to talk to our servers. But the server names are not known anywhere! We 
*could* add them to our `/etc/hosts` file, but this is awkward. First, we might not have sufficient privileges to fully automate that (and if we forget, the tests fail) and there *might* be interference with other software that uses the resolver.

Luckily, `curl` has it's own, internal resolver, where one can stuff things in that are used first. This makes our tests independent of any OS or network specifics. You invoke it via:

```
> curl --resolve a.test.net:5002:127.0.0.1  https://a.test.net:5002/
```

and this tells curl "Whenever you see a request to a.test.net on port 5002, do not bother with the DNS, use this address for it". 

`curl` will talk to our test Apache as if it were a "real" server on the internet. It will use `a.test.net` as hostname in TLS SNI as well as 'Host:' header. As it would do in production. Which will also mean that `curl` does not trust our test server - yet.

To make `curl` trust our test server certificates, we need to feed it the certificate of our test CA:

```
> curl --cacert /test/ca/ca.pem --resolve a.test.net:5002:127.0.0.1  https://a.test.net:5002/
```

and we have a trusted HTTP interaction between curl and our server, just like in "real" life!

## Wrapping Up

This is an easy recipe on how to create a test setup that does real `https:` on your local machine without
anything you need to setup beforehand. 

My test suites for HTTP/2 and ACME and Rust TLS in Apache httpd all make use of that. In a bit more
elaborate form, since they need to check more variations in TLS and certificates, but the base idea
is the very same.

Hope this helps you start your own HTTP related test suites.

