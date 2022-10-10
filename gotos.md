# Gotos

Something light and seemingly trivial: on the use of `goto` statements. Inspired by [this tweet](https://twitter.com/typeswitch/status/1578881799438356481). People discover this independently with growing experience and it seems beneficial to make this an "official" coding pattern (so these things exist).

This post is about `goto` use in the `C` programming language. It is not advocating the use of `C` or make any comparison with other languages. This is for people who write/maintain `C` code. A suggestion on good use of the language.

## What are we talking about?

We are simply taking about the following pattern for a function:

```
status_t some_function(type foo)
{
    status_t rv = ERROR;
    
    if (looks_bad(foo)) goto cleanup;

    rv = do_good(foo);
cleanup:
    return rv;
}
```
 * vars start initialized. here `rv`.
 * success is optional, start with `rv = ERROR`.
 * check conditions for success, `goto` single exit place
 * success code stays on the lowest indentation level
 * there is **one** place where the function returns

When we read such code, we can easily grasp that

 * `some_function()` is, apart from error cases, doing `do_good()`.
 * it is only successful, if `do_good()` is successful.

(side note: if you name the label `cleanup` or `do_exit` or `unwind` is rather unimportant. We *do*
 want to use the same term everywhere in a project, though. So, it needs some neutral name.)
 
A function often does more than one thing. How would that look?

```
    ...
    rv = do_good1(foo);
    if (SUCCESS != rv) goto cleanup;
    
    rv = do_good2(foo);
cleanup:    
```

The same `goto` pattern applies to the steps in successful execution. Again, we can read the successful execution path of the function at the lowest indentation level. If there are alternate paths, one can still keep the pattern:

```
    ...
    if (test(foo)) {
        rv = do_good1(foo);
        if (SUCCESS != rv) goto cleanup;
    }
    else {
        rv = do_good2(foo);
        if (SUCCESS != rv) goto cleanup;

        rv = do_good3(foo);
        if (SUCCESS != rv) goto cleanup;
    }
    rv = do_good4(foo);
cleanup:    
```

The branching in successful execution becomes visible. By sticking to this pattern, reading "foreign" code helps us in understanding what it does. If the function goes beyond one page, we know a statement at lowest indentation will *always* be run on success, no matter what is done in the lines we do not see displayed.

Also, the handling of "fail"s loses complexity. It is the same *for all fails*. 

## Resource Allocations

Often, a function needs to allocate resources during its execution (memory, files, locks). How does that fit in ?

```
status_t some_function(type foo)
{
    status_t rv = ERROR;
    char *buffer = NULL;
    
    if (looks_bad(foo)) goto cleanup;

    buffer = malloc(10);
    if (!buffer) goto cleanup;
    
    rv = do_good(foo, buffer);
cleanup:
    if (buffer) free(buffer);
    return rv;
}
```

With **every** execution path ending in `cleanup`, we have one place to deallocate resources. In `C` we have to take care about that. It is a burden, so we want this to be as manageable as possible. This becomes very important when you **maintain** a function and need to add code and allocations. Lots of leakages result from early returns or nested if/else branching where one case was overlooked.

### Returning Allocations

A variant of the above is when a function is supposed to return an allocated resource. Such as:

```
status_t make_out(otype **pout, type foo)
{
    status_t rv = ERROR;
    otype *out = NULL;
    
    if (looks_bad(foo)) goto cleanup;

    out = create_out();
    if (!out) goto cleanup;
    
    rv = do_setup(out, foo);
cleanup:
    if (SUCCESS == rv) {
        *pout = out
    }
    else {
        *pout = NULL;
        if (out) free_out(out);
    }
    return rv;
}
```
We see that `pout` is only assigned a value on success and `NULL` otherwise. *No matter* what else the function may need to do in future revisions. Calling a `do_setup2()`, `do_setup3()`, allocate other resources etc. will not change that.

# Concluding

This is a proposal, a recommendation from experiences working in decades old code bases. Most everything will change over the years, require handling of additional edge cases, changing APIs and use in varied contexts. We want changes to "fit in easily". We want to be sure to not overlook some deeply nested if/else edged early return that leaks or leaves something in a half-initialized state.

This pattern works for this.
