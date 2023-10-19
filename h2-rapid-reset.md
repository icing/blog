# HTTP/2 Rapid Reset and Apache

The [HTTP/2 Rapid Reset Vulnerability, CVE-2023-44487](https://www.cisa.gov/news-events/alerts/2023/10/10/http2-rapid-reset-vulnerability-cve-2023-44487) has made the news last week. For a good overview of what Cloudflare experienced and how they responded, I recommend [HTTP/2 Rapid Reset: deconstructing the record-breaking attack](https://blog.cloudflare.com/technical-breakdown-http2-rapid-reset-ddos-attack/) by Lucas Pardue and Julien Desgats.

## Apache httpd Impact

I am assuming you are by now somewhat familiar with what the attack is (if not, read the Cloudflare blog). 

This is a Denial-of-Service attack. No private data is exposed, nothing is infected or compromised. Vulnerable infrastructure is prevented from doing its job. That may be annoying or very critical, depending on the deployment. Read: my personal website vs. the control center of a nuclear power plant.

However, this is beyond the control of the Apache httpd project. Any bug in software may affect your power plant if you do not have sufficient redundancies and fail safes. Treating all bugs in software as super critical would not help.

With a "typical" Apache httpd deployment in mind (like as reverse proxy in front of a business application), there was no security impact regarding the exploits known at the time CVE-2023-44487 was published. 

The attack causes CPU consumption to digest the incoming data, as with all data sent to the server. This stayed limited to what a single thread in your server can do. Meaning, even if a client would blast GB/s to the server, Apache would consume one HTTP/2 frame at a time, in a single thread on the connection. If the client could send faster than this thread can consume, the TCP connection would block for the client.

In coordination with the excellent [nghttp2 project](https://nghttp2.org), they added the feature to terminate connections that send "too many" Reset frames. This was released as v1.57.0 of nghttp2 and it is highly recommended by us to update to that version. With that, CPU consumption in Apache on these attacks drops significantly, making the server even more resilient.

Some days later, we were contacted by a security researcher with a variation of the Rapid Reset attack.

### CVE-2023-45802

The attack variation stumbled upon an ever-present bug in our HTTP/2 implementation: on a rare timing when a request was cancelled by a client (Reset), the memory allocated for that request was not released right away. It was released when the connection closed (or timed out). The rarity of this happening made this bug unnoticeable and survive this long.

The new attack was able to hit just the right timing to trigger this often. **Without** nghttp2 v1.57.0 installed, the attack could exhaust the memory available, causing a server process to die in a controlled way (httpd handles out-of-memory conditions by terminating the process). The httpd main process would then fork another child. 

**With** nghttp2 v1.57.0 installed, the attack will not have this impact. The abusive connection would be dropped after ~1000 Resets, the memory would be freed, other traffic is not disrupted.

In Apache httpd v2.4.58, we fixed this bug. We are not aware of any variation of Rapid Reset that affects us.

![Whale Shark](https://upload.wikimedia.org/wikipedia/commons/f/f1/Whale_shark_Georgia_aquarium.jpg)

### Apache HTTP/2 Protection (since 2016)

While working on the early implementation of HTTP/2 in Apache, we thought it unwise to give clients all available server resources just because they asked us to. Browsers at the time were mandating to support 100 parallel requests in each connection to make them faster. But on small installations, a process might have only 25-50 worker threads. Occupying all these for one connection would mean other connections suffer. This did not appear smart.

Instead, we implemented the following: Apache will start working on the first 6 requests from a client. All other requests arriving on the connection would be parked. We chose 6 because that is the limit for parallel requests in HTTP/1.1 and applications migrating from HTTP/1.1 to HTTP/2 should possibly live well with this.

This limit is raised rapidly when the client behaves well, e.g. reads all responses to its requests. And, important now, the limit is lowered for clients that Reset requests that the server is working on. When we wasted resources on the client, we become hesitant to commit further.

This raising and lowering is an ongoing evaluation on each individual connection. A connection will not get stuck, just because at one time a browser tab was closed, causing all ongoing requests to abort - as they should.

In the case of Rapid Reset attacks, this mechanism swiftly restricts the connection to 2 active requests. We count a request as active as long as we have a worker committed to it, irregardless if the client cancelled it or not. Let's say a request goes to a backend, starting an expensive database query. As long as that query has not answered, it blocks an active slot.

With this protection in place, it does not really matter if a misbehaving client sends us 2 or 2000 requests on a connection.

## Conclusion

We are protected against the attack as published. nghttp2 v1.57.0 makes this attack and variations we know of a non-issue. We had a bug exposed by this. We fixed.

Our HTTP/2 protection scheme works. We feel justified in the choices made in 2016, balancing browser demands vs. security. There is a reason [sharks in the infrastructure](https://www.simplethread.com/relational-databases-arent-dinosaurs-theyre-sharks/) do exist.

### Final Observation

We feel that issuing the blanket CVE-2023-44487  against the HTTP/2 protocol was not a good idea. I believe everyone involved had the best intentions. However, the CVE has the following flaws:

* A blanket CVE means products are guilty until proven innocent. But CVE infrastructure works the other way around. The CVE databases in the world are not prepared to keep lists of unaffected products, AFAIK. There is not defined path to publish this so that it keeps all users informed.
* The problems around HTTP/2 parallelism were discussed during standardization in 2015. Nothing in this is a surprise. It is something the implementors were aware they'd need to manage or fall victim to.
* Industry wide problems around HTTP/2 management were reported in 2019 as [CVE-2019-9511](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-9511), [CVE-2019-9513](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-9513) and [CVE-2019-9516](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-9516) in an effort led by Netflix. The Rapid Reset is not *that* different. This should have been a wake-up call for everyone.
* In 2019 we described Apache httpd's mitigation to such attacks to all participants in the Netflix effort. All parties now claiming surprise were present. It's fine if you decide not to follow our lead and chose another path. But crying "foul" 4 years later makes you appear weak.

Own up to your mistakes, we all make them. Do not throw mud in everyone's direction.

- @icing@chaos.social





