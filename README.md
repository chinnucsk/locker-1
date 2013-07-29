# (WORK IN PROGRESS) locker - an Erlang per-cluster lock application

[![Build Status](https://travis-ci.org/blt/locker.png)](https://travis-ci.org/blt/locker)

This is a simple resource locking OTP application for an Erlang cluster.
Resources are identified by a name (`:: term()`) and may have more than one lock
taken out for that resource of any of the provided types. The vocabulary of this
library is currently in flux. I'm calling the 'resource lock' a 'lock' and the
'locks taken out' for that resource is 'magnitude' or 'size'. As this is early
days, please to anticipate the vocabulary changing. Suggestions appreciated.

Three kinds of locks are intended:

  * process associated :: a lock is set by a process and is destroyed only when
    the process gives it up OR the associated process dies
  * time-based :: a lock is set and given a timeout value, beyond which it is
    destroyed
  * time/process hybrid :: a lock is set by a process and is destroyed when the
    process give it up OR the timeout limit is reached but NOT if the associated
    process dies

Only the first, 'process associated' is implemented in this release.

## Example

Given two nodes on a host named 'walden' the node `alpha` running the `locker` library:

```
> erl -sname alpha -setcookie monster -pa ebin
Erlang R15B03 (erts-5.9.3) [source] [64-bit] [smp:8:8] [async-threads:0] [hipe]
[kernel-poll:false]

Eshell V5.9.3  (abort with ^G)
(alpha@walden)1> net_adm:world_list([walden]).
[beta@walden,alpha@walden]
(alpha@walden)2> application:start(locker).
ok
(alpha@walden)4> application:which_applications().
[{locker,"A lock application for Erlang clusters.","0.0.1"},
 {stdlib,"ERTS  CXC 138 10","1.18.3"},
  {kernel,"ERTS  CXC 138 10","2.15.3"}]
```

and `beta` merely part of the crowd

```
> erl -sname beta -setcookie monster -pa ebin
Erlang R15B03 (erts-5.9.3) [source] [64-bit] [smp:8:8] [async-threads:0] [hipe]
[kernel-poll:false]

Eshell V5.9.3  (abort with ^G)
(beta@walden)1> application:which_applications().
[{stdlib,"ERTS  CXC 138 10","1.18.3"},
 {kernel,"ERTS  CXC 138 10","2.15.3"}]
```

we'll set the magnitude for resource lock `small_lock` down to one

```
(alpha@walden)3> lk:set_size(small_lock, 1).
ok
```

and take the lock on `beta`

```
(beta@walden)2> lk:set(small_lock).
ok
```

but find on `alpha` that the lock resource is overburdened

```
(alpha@walden)5> lk:set(small_lock).
{error,overburdened}
```

Taking pity on `alpha` we'll use `beta` to bump the lock size up

```
(beta@walden)3> lk:set_size(small_lock, 2).
ok
```

and, just as planned, `alpha` will be able to set the lock

```
(alpha@walden)6> lk:set(small_lock).
ok
```

## FAQ

### How does this differ from `global:set_lock` and friends?

While this application took much of its initial inspiration from the `global`
module, it differentiates itself by providing:

 * more than lock semantic category (aspirational)
 * 'magnitudes' per resource lock
 * fine-grained control over retry limits (aspirational)

Resource lock names are any term, unlike `global` locks which are a bit more
complicated. This simplifications requires that all lock requests are implicitly
tied to the caller's PID.

### Are the locks volatile?

Yes, absolutely. All information about locks and their holders is stored in
memory on a single node of a cluster. If this node should fail, all the `locker`
information is lost.

The library is intended to be paired with appropriate `kernel` configuration to
keep `locker` running at all times but makes no attempt to do so on its own.

### What are the performance implications of the library?

I haven't the foggiest.

## Future Work

1. Only the process-based semantic category for locks is currently implemented.
   The remaining two will be included as well.
1. OTP admits some notifications of netsplits for applications. `locker` makes
   no attempts to use this information as yet, but should and will.

## LICENSE

`locker` is provided under the MIT License. Please see `LICENSE` in the root of
this project for details.