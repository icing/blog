# Apache httpd 2.4.50 post mortem

With Apache 2.4.50 the team fixed [CVE-2021-41773](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-41773), 
a critical security flaw that allowed _under certain conditions_ an outside to access files on your server outside of the configured document roots.

This fix was correct for the issue reported, but it did not close the weakness completely, as was discovered soon thereafter by people in the security community. Indeed, the weakness was worse than originally thought. But it also affected way less installations than was communicated in the media.

This blog explains what the issue really was and why you, most likely, were not affected.

## Apache, Base Security

At the core of security inside the server are its mechanisms for _authorization_, meaning what a request from the outside is allowed to access. The most basic is the authorization for `all` which means all requests, regardless of where they come from, what user has maybe authenticated or not or which method they use.

Most linux distributions we looked at install a Apache httpd with permissions for `all` properly set. For example, in debian, your `apache2.conf` contains:

```
<Directory />
        Options FollowSymLinks
        AllowOverride None
        Require all denied
</Directory>
```

This means that all access is denied for requests to anything in `/`, your complete file system. And this may not be overridden by other auth configurations. 

How can the server then deliver any of your website files? Well, `apache2.conf` also contains further:

```
<Directory /var/www/>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>
```

This defines `/var/www` as a place where requests have access and that has preference over the `/` settings for all files underneath (File permissions in the operating system apply in addition to this, of course, the server does not serve requests as `root`.)

Above this `Directory` base security, you can also define `Location` based requirements. These apply to the URLs send to your server, regardless of where they are later mapped to in the file system. And there are additional checks involved for checking that requests stay inside the *document root*  defined for a host.

But if those checks have a bug, the `Directory` based security settings are the last line of defense.

## Affection, 2.4.49

With base security removed, the 2.4.49 version (and only that!) allowed something like:

```
curl http://host/img-sys/.%2e/%2e%2e/etc/passwd
... contents of /etc/passwd
```

*iff* that `img-sys` is mapped using `mod_alias`'s `Alias` or `ScriptAlas`, as in:
```
   Alias /img-sys/ /opt/images/
```

Assuming `ServerRoot /path/to/httpd`, what did httpd 2.4.49 do?

 - looking for `/img-sys/.%2e/%2e%2e/etc/passwd`
 - normalize url to: `/img-sys/../../etc/passwd` (***wrong!***)
   * should have resulted in `/../etc/passwd` and failed (above root)!
 - decode for file access: `/img-sys/../../etc/passwd`
 - make it a filesystem path
   * mod_alias replaces `/img-sys/` by `/opt/images` which gives `/opt/images/../../etc/passwd`
   * and normalizes to `/etc/passwd`
 - is access granted?: yes <- if base security was removed

This was reported by cPanel security team. They verified our fix and we released 2.4.50 and reported CVE-2021-41773.

Absent an `Alias` or `ScriptAlias` for the base path `/img-sys`, Apache httpd would reject
`/path/to/httpd/htdocs/../../etc/passwd` because the core URI to path translation mechanism
rejects anything above the `DocumentRoot` (`/path/to/httpd/htdocs` here), which is not the case of
mod_alias (this is being addressed for the next httpd version as a second line of defense).


## Affection, 2.4.50

But we had not fully understood the impacts of the change in 2.4.49. People reviewed our code (we like that!)
and discovered another exploit. If httpd was configured with something like:

```
<Directory "/my-cgi-dir">
    Options +ExecCGI 
    Require all granted
</Directory>
...
ScriptAlias /cgi-bin/ /my-cgi-dir/
```

You could send a request like:

```
curl http://host/cgi-bin/%%32%65%2%65/bin/sh
# httpd executes /bin/sh
```

What httpd 2.4.50 did here:

 - looking for `/cgi-bin/%%32%65%2%65/bin/sh`
 - normalize url to: `/cgi-bin/%2E%2E/bin/sh` (***sorry, still not correct!***)
 - decode for file access: `/cgi-bin/../bin/sh` (***wrong!***)
 - check file path? no (it's a cgi)
 - make it a filesystem path
   * mod_alias replaces `/cgi-bin/` by `/my-cgi-dir/` which gives `/my-cgi-dir/../bin/sh`
   * and normalizes to `/bin/sh`
 - is access granted?: yes <- if security defaults were removed
 - give to cgi handler to execute

The fist failure was to allow something like `%%32` to decode to `%2`. `%%` is not valid in a URL. A `%` needs to be followed by 2 hex digits, always.

The second failure was that we decoded certain characters like `%32` *again* when preparing the file path. Some checks ran after the 1st decode and passed, which allowed the result of the 2nd decode to be passed to cgi handlers. The second decode for file access should no longer convert characters that the first one converted.

This vulnerability is reported in [CVE-2021-42013](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-42013) and was fixed in 2.4.51.

## Fix, 2.4.51

What httpd 2.4.51 does now in the example above (replacing `%%32` by `%25%32` because the former is now `400 Bad request`):

 - looking for `/cgi-bin/%25%32%65%25%32%65/bin/sh`
 - normalize url to: `/cgi-bin/%2E%2E/bin/sh` (***correct!***)
 - decode for file access: `/cgi-bin/%2E%2E/bin/sh` (***correct!***)
 - check file path? no (it's a cgi)
 - make it a filesystem path
   * mod_alias replaces `/cgi-bin/` by `/my-cgi-dir/` which gives `/my-cgi-dir/%2E%2E/bin/sh`
   * and normalizes to `/my-cgi-dir/%2E%2E/bin/sh`
 - is access granted?: yes, it's beneath the cgi dir
 - give to cgi handler to execute, -> not found


We hope we closed the lid on that one. We added several test cases to make sure this
will not happen again. But security is improvable. We expect people will closely inspect our fix and try to 
come up with ways to wreck it that we could not think of. 

And that is as it should be.

Another possible strategy discussed was to completely revert the changes that led to the 2.4.49 vulnerability.
But that would have sort of given up on the connected improvements that we feel are good for the server. We
would end up with untouchable code in a central part of request processing. Ultimately we felt is is
better to go forward and shore up our test cases of the situation. So we can make sure those bugs will
not happen again.

If you are interested in what this is actually about "url decoding" and path checks a server has
to do, I'll give a short overview at the end of this post.

## How It Went

We are sorry to not have caught the full implication of the bug in 2.4.49. While our fix was correct, it
did not close the issue. Maybe with more time spend on it, we would have found it ourselves. But time
was pressing to bring out 2.4.50. With hindsight, we could have prevented the bug in 2.4.49 to happen, so
such a discussion is leading us nowhere.

2.4.51 contains only the new fix and no other changes, we hope that makes upgrades to that version easier.

### Security Reporting

We are very happy that people who discovered the flaws contacted us on our security list and worked
with us to fix and verify the changes. 

On the 2.4.49 issue this was Ash Daulton along with the cPanel Security Team. They also stayed very 
active on testing the 2.4.51 version. The 2.4.50 issues was reported by Juan Escobar from Dreamlab 
Technologies, Fernando Muñoz from NULL Life CTF Team, and Shungo Kumasaka.

We would like to thank these fine people for their help and professionalism.

### The Project

We had improved our release process the weeks before and were able to ship 2.4.50 a week after reporting. 
Since the issue fixed was so critical, it got a lot of attention.

We got the report of the second weakness soon afterwards (last Tuesday). We shipped the fix in 2.4.51 
two days after that with a shortened release vote, as we feared this would not stay unknown for long.

The whole httpd security team was very active to get this done in such a short time. It 
was quite intense. Many thanks to Yann Ylavic, Rüdiger Pluem, Joe Orton, Eric Covener, Rainer Jung, 
Giovanni Bechis and Mark J. Cox, the Apache security coordinator.


October 8th 20201,
Stefan


## Appendix: URL Decoding, what is it and why?

URL Decoding, is I named it here, is the *interpretation* of a URL. What resource does the request want to access?

This is different from the *parsing* of a URL, which dissects a URL into its semantic pieces like `scheme`, `host`, `path` and `query`. The Apache httpd URL parser worked find. There was no bug in that.

But to find out which resource is being access, the different parts of the URL need to be interpreted, e.g. mapped
to the correct host configuration and *handler* that is then responsible for answering the request. The default handler in Apache maps the path of an URL to the file system, the `document root` that you have configured for a host.

Let's say you have a file named `reports 2021.txt` that you want to server over http. The space in the file name is ok in the file system, but it is not allowed in an URL. Instead, it needs to appear as `%20`. The URL would be like `https://myhost/reports%202021.txt`.

Somewhere, in answering this request, the `%20` needs to be decoded to ` ` to find the file. How hard can it be?

Some characters are necessary to "encode" this way, since they have special meaning. If you want to use `&`in a URL query, e.g. like `?fruits=apples&lemons` you need to escape it or the query will be interpreted not as you want. So you  send `?fruits=apples%26lemons`.

The URL standard allows you to escape any character, so you could also send `?fruits=%61pples%26lemons`. For the server to treat the value you sent, it needs to "decode" these characters back when using the value of the `fruits` parameter, which then is the `apples&lemons`.

This is tricky, because:

 * some access restrictions or mappings to apply the to URL and have nothing to do with files.
 * there are many URLs that point to the same file, e.g. `https://myserver/%61pples%26lemons.txt` needs to work as well.
 * once decoded, you cannot go back: which of the possible URLs do you mean?
 * you cannot decode twice: `10%2526lemons` once is `10%26lemons`, twice is `10&lemons`.

In Apache, the concepts involved here are `Location` and `Directory`. `Location`s apply to URLs and `Directory`s apply to decoded URLs, e.g. file system paths. So, to check which Location is meant by a URL, the server takes the URL itself. For a Directory the server decodes the URL. But this is not enough.

A server needs to protect Locations. For example, if your want everything beneath `https://myserver/private/` to be only available to certain users, you configure the Location `/private` with your restrictions. And you want this to work also when someone uses the URL `https://myserver/pr%69vate/`. So, a server needs to do some *normalization* of the URL locations for this to work.

This normalization is implemented by decoding those parts in the URL that are not *reserved* characters, e.g. have
no special meaning. These characters are defined in the URL standard. Then locations can be checked properly and an outsiders cannot easily bypass them.

Having checked which `Location` applies to the normalized URL, it can determine if the resource to server is a file in the file system, or a CGI script, or a proxy mapping to another host, or whatever handler you have configured. If the location maps to the file system, it needs to decode the other `%` things to get to the names the file system actually has. 

And then it needs to verify which `Directory` this URL then points to. And what access has been configured for that. That's the reason for configuration mentioned at the top:

```
<Directory />
        Options FollowSymLinks
        AllowOverride None
        Require all denied
</Directory>
```

which denies access to any file in the file system. And on top of that, one defines the `Directory`s where access is `granted`. Such as in

```
<Directory /var/www/>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
</Directory>
```

While not explaining every mapping and security check in the server, this hopefully gives an insight into the complexities 
involved and why you want to keep those default configurations that all linux distributions have.

