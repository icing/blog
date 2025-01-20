# curl logging/tracing

I [recently tooted my happiness with curl's tracing capabilities](https://chaos.social/@icing/113798024878201625) on mastodon (the place to be for awesome curl news) and people showed interest in hearing more details about it. So, here we go...

### The case for logging/tracing

First, if you have added `printf(...)` statements before, only to remove them again after a bug was fixed, you should consider tracing. The investment is worth it.

One challenge when putting your code into the hands of people on the internet is that they often run it in environments that you have never seen before and *you do not have access to*. That maybe because it runs in their internal network or on a public system with credentials you do not have.

While "it works on my machine" is a common meme, the reverse is often true: "***It will never fail on your machine**". Because you are unable to reproduce the conditions a user has. You can add CI jobs as much as you want (curl has now close to 200!), but the world is a large place. So, what do do?

You enable your users to do the debugging themselves by giving them easy to understand and effective to control **logging capabilities** so they can get an understanding **wtf is going on**. And when they cannot figure it out, they can easily **share the logs with you**.

![Unhappy People with a Trace](./images/unhappy-without-trace.png)

### Log Files Anatomy

Stating the obvious here:

* They should be text. You do not want binary formats, they get in the way.
* They should be line based. Every event ideally *should* be on one line. Easy to grep/filter.
* Every line should carry a time stamp with sub-second resolution, human readable.
* If your code handles multiple things at the some time, your lines should carry the identifier of "the things".
* If your code is modular, add the information which module produced the log line. Something is wrong in authentication? Grep for "AUTH".
* Allow enable/disable modules individually. Keeps log files smaller.

Log levels *may* be helpful or may get in the way (levels beyond 'debug'). It depends. If your code has a good modular structure, go with modular logging first. Only if that is not sufficient, think about adding levels.

### Logging/Tracing in Curl

Curl has had "info" and "warning" output since forever. Those messages are not logging per se. They are more part of the user interface. With `--verbose/-v` you will see those. In addition, there have been options for getting more details with `--trace-ascii` and `--trace-time` for some time. 

Two years ago, we revamped curl's tracing and added transfer/connection identifiers, modular switches and better usability. Those changes started appearing in curl 8.2.0.

#### curl trace 'modules'

The following areas in curl have their own 'logging id':

 * **transfer protocols**: like FTP, HTTP, WS. Commonly named after the url prefix (hence WS for WebSockets).
 * **network protocols**: like TCP, SSL, Proxy Stuff, DoH
 * **application sides**: modules 'read' and 'write' are about reading stuff from the libcurl application (for upload) or writing response out to the application (download).
 * **others**: there is always things that do not quite fit everywhere. We recently added `ssls` for tracing of SSL session caching.

The full list of trace modules is on the [man page for curl_global_trace](https://curl.se/libcurl/c/curl_global_trace.html). Via this function, a libcurl application can specify exactly which modules it wants to receive trace output for. For the curl command line, you can use option `--trace-config`.

#### curl verbosity

Since curl 8.10.0 you can increase the verbosity of the curl command line by specifying:

*  `-v`: which gives you the "info" messages. Like curl has done for ages (backward compatibility is important to us).
*  `-vv`: which gives you timestamps and the ID of the transfer/connection involved. The first transfer has ID `0`, as has the first connection. They appear right after the timestamp as `[n-m]` for transfer `n` using connection `m`. Additionally, tracing for all protocol modules is enabled.
*  `-vvv`: enables tracing of `SSL`, `read` and `write`.
*  `-vvvv`: adds all network modules trace output. Like TCP operations on the socket. This is pretty deep for when we need to see if something is buggy on the network itself or in proxy connections.

All modules add their identifier to the trace output. A line from the `HTTP/2` module in `curl -vv` would appear as:

```
12:25:22.474682 [0-0] * [HTTP/2] [1] <- FRAME[DATA, len=11364, eos=1, padlen=0]
```
which says the transfer 0 on connection 0 received a HTTP/2 DATA frame on stream 1. The frame carried 11364 bytes of data and was marked as the End-of-Stream, e.g. the stream is closed now.

This is all great for analysing a log file. No matter how large it is, we can easily get the entries for a failed transfer and have a quick look at the HTTP/2 frames that were sent/received. etc.

#### libcurl application verbosity

If you write a libcurl application, you configure module tracing via `curl_global_trace()` *at the start*. There is no dynamic switching after that. What you *can* control dynamically is the verbosity of each transfer. The module trace output will only happen for verbose transfers (e.g. `CURLOPT_VERBOSE` is set).

### The Implementation

When implementing tracing capabilities, it is important to make them very cheap. Especially when tracing is not enabled. Transfers in "normal" operations should not be slowed down by computing trace statements that will not produce any output.

The first thing is that you can build libcurl without any verbose handling at all. If you configure `--disable-verbose` all code for traces disappears. The library will have a minimal footprint. Unless you are very tight on memory, you would not want to do that, however.

How does a trace statement look in HTTP/2 code? Here is the line that outputs the HTTP response status:

```
    CURL_TRC_CF(data, cf, "[%d] status: HTTP/2 %03d", stream->id, stream->status_code);
```

here `data` is the current transfer (for historic reasons they carry this name - naming is hard). `cf` is the "connection filter" involved. The `cf` is of type "HTTP/2". You can think of it as the "class" of a connection filter and `cf` is an instance of that class. The rest is like a `printf()` statement.

As you may have guessed from the uppercase writing, `CURL_TRC_CF()` is a C macro.

```
#define CURL_TRC_CF(data, cf, ...) \
  do { if(Curl_trc_cf_is_verbose(cf, data)) \
         Curl_trc_cf_infof(data, cf, __VA_ARGS__); } while(0)
```
which checks first, if the tuple `(cf, data)` has tracing enabled *before* the trace parameters are evaluated and the trace output function is called. This check involves other macros:

```
#define Curl_trc_is_verbose(data) \
            ((data) && (data)->set.verbose)
#define Curl_trc_cf_is_verbose(cf, data) \
            (Curl_trc_is_verbose(data) && \
             (cf) && (cf)->cft->log_level >= CURL_LOG_LVL_INFO)
```
which foremost checks `(data)->set.verbose` and if `CURLOPT_VERBOSE` has not been set, the rest of the checks is never evaluated! This means unless you have a verbose transfer, all trace statements are just a simple boolean check on the same memory location over and over and over. On modern cpus, that will cost basically nothing.

For a verbose transfer, the macro will then check the log level of `(cf)->cft`. `cft` is the type of the filter, in this case HTTP/2, which is a global static instance. This means, although you might have thousands of connections and several thousand connection filters, there is only a small, fixed amount of *filter types* that are checked. Again, cheap for modern cpus to do. And since libcurl does not allow you to change the filter verbosity on the fly, we do not need any semaphores or `volatile` or `atomics` here.

This means we can write elaborate tracing statements in our code at basically run with no cost (other than memory footprint) unless enabled.

### Conclusion

Curl's tracing capabilities have proven invaluable when tracking down bugs, server errors or other stranger things. They are easy for users to produce and even large log files are efficient to analyse due to their structure.

For `curl` users, they allow more insight into what is going on. To see, if something looks like a bug, a  mistake on their part or the network/server having a bad day.

We hope we chose the trace options in a way usable to you. We are always open for feedback. Or a PR with additions!



