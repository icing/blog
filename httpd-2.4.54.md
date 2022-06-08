# Apache httpd 2.4.54

On 2022-06-08 the project released version 2.4.54 of Apache httpd. 8 CVEs have been fixed, 7 low and 1 moderate. Bugs have been fixed, the ACME certificate provisioning has some enhancements.

In detail:

## CVEs

The relatively large number of CVEs are the results of security people continuing to analyze our code and the deployment scenarios it runs in. We are of course not happy about having vulnerabilities, but we are grateful that people are finding them and working with us to fix them!

For the curious: 4 out of 8 vulnerabilities were in relation to C, all 4 with low impact. Interestingly, the more relevant vulnerabilities are in combination with other systems. Request smuggling or header suppression make attack vectors where exploits are not immediately visible to defenders.


 * CVE-2022-26377 (moderate): reported by Ricter Z @ 360 Noah Lab. This vulnerability allowed request smuggling to an proxied AJP server (like tomcat). The cause was a weak check on HTTP/1.1 'Transfer-Encoding'. Transfer-Encoding determines where a request ends (and the next one starts). Confusion about request length has always been an attractive attack vector in HTTP/1.1. Not a C vulnerability.
 
 * CVE-2022-28330 (low): reported by Ronald Crane (Zippenhop LLC). A string character was checked in the Windows mod_isapi module, but the length limit was applied from another string, making that check possibly read beyond bounds. A C vulnerability.

 * CVE-2022-28614 (low): reported by Ronald Crane (Zippenhop LLC). An API function for writing byte arrays was not handling lengths larger than INT_MAX correctly, due to signed/unsigned conversions. We are not aware of a situation this can be triggered from outside, but it is in the API and external software might stumble over this. A C vulnerability.

 * CVE-2022-28615 (low): reported by Ronald Crane (Zippenhop LLC). As in the previous CVE, signed/unsigned conversion strikes when calling `ap_strcmp_match()` with very large strings. No exploit known, but API visibility. A C vulnerability.

 * CVE-2022-29404 (low): reported by Ronald Crane (Zippenhop LLC). A denial of service attack when sending large request bodies that `mod_lua` then may try to interpret. Apache enforces limits on request sizes, but mod_lua - due to its sneaky nature - was not protected. Not a C vulnerability.
 
 * CVE-2022-30522 (low): reported by Brian Moussalli from the JFrog Security Research team. In certain configurations involving `mod_sed`, very large input may cause the module to make even large memory allocations. That then may cause a server process to abort. A denial of service attack. Not a C vulnerability.
 
 * CVE-2022-30556 (low): reported by Ronald Crane (Zippenhop LLC). Using `mod_lua` with websockets could lead to read beyond bounds of allocated storage buffers. A C vulnerability in the implementation of a lua function.

* CVE-2022-31813 (low): reported by Gaetan Ferry (Synacktiv). Proxied backends are supplied with `X-Forwarded-*` headers by Apache that they use to make decisions, for example if to allow or deny a request. When a client added these headers to the `Connection` header field, httpd no longer forwarded these headers to the backend. This is a potential security vulnerability, as backends might draw wrong conclusions on the header fields absence. Not a C vulnerability.


## ACME

There have been 2 enhancements I described in other blogs:

1. [More ACME](./more_acme.md): setup another CA as backup in case the first one has an extended outage.
2. [tailscale](./tailscale.md): how to setup real certificates for your Tailscale domains.

## Bugfixes and other things

 * SSLFIPS is now working with OpenSSL 3.0
 * `mod_proxy_http` fixed an edge case in `100-continue` handling
 * ACME fixed a bug with very large certificates (many domain names)
 * `mpm event` fixed bugs in handling child process restarting/accounting
 * `ab`, the cli client, can now use TLSv1.3
 * `http2`: when clients seemingly misbehave, ongoing requests are no longer interrupted.

