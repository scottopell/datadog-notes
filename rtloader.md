# Enabling python in agent build

## The Problem
On a linux vm where I'm trying to build the agent and run it locally, `python`
components are disabled for me.
I can see this log that indicates a failure to load the `rtloader` subsystem
properly.
```
2022-06-10 11:19:52 EDT | CORE | ERROR | (pkg/collector/embed_python.go:19 in pySetup) | Could not initialize Python: could not load runtime python for version 3: Unable to open three library: libdatadog-agent-three.so: cannot open shared object file: No such file or directory
```

## Background
`inv agent.build` builds the `rtloader` cmake project first which results in
shared libraries for each platform for the configured python version. In my
case, this yields:
- `dev/lib/libdatadog-agent-rtloader.so`
- `dev/lib/libdatadog-agent-rtloader.so.1`
- `dev/lib/libdatadog-agent-rtloader.so.1.0`
- `dev/lib/libdatadog-agent-three.so`

Starting at the top, we can see the agent binary directly depends on `libdatadog-agent-rtloader.so.1`
```
readelf -d bin/agent/agent | grep -i needed
 0x0000000000000001 (NEEDED)             Shared library: [libdatadog-agent-rtloader.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libgcc_s.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [ld-linux-aarch64.so.1]
```

When running `bin/agent/agent`, the linker is able to find
`libdatadog-agent-rtloader.so.1` due to a correctly set `runpath`.

```
 0x000000000000001d (RUNPATH)            Library runpath: [/home/ubuntu/dev/datadog-agent/dev/lib]
```

So far so good, but why can't it find `libdatadog-agent-three.so`?
To recap, the original error was
```
2022-06-10 11:19:52 EDT | CORE | ERROR | (pkg/collector/embed_python.go:19 in pySetup) | Could not initialize Python: could not load runtime python for version 3: Unable to open three library: libdatadog-agent-three.so: cannot open shared object file: No such file or directory
```

Hmm, okay, so what is trying to load `libdatadog-agent-three.so`?

Running with `LD_DEBUG=all bin/agent/agent run` shows me:
`225739:	file=libdatadog-agent-three.so [0];  dynamically loaded by /home/ubuntu/dev/datadog-agent/dev/lib/libdatadog-agent-rtloader.so.1 [0]`

So if we check `libdatadog-agent-rtloader.so.1`'s library dependencies, we
should find `libdatadog-agent-three.so`, right?


```
readelf -d dev/lib/libdatadog-agent-rtloader.so | grep -i needed
 0x0000000000000001 (NEEDED)             Shared library: [libstdc++.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [libgcc_s.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [ld-linux-aarch64.so.1]
```

Wrong.

What's going on here is that `libdatadog-agent-rtloader.so.1` is loading
`libdatadog-agent-three.so` via the `dlopen` syscall as a runtime load (not a
link-time load)

This is a bit hard to confirm, but I'm fairly confident this is what's going on.

We can review the code (for linux) in
[`api.cpp`](https://github.com/DataDog/datadog-agent/blob/d7ddde09f076acf95248455906c5dc0e2299f84b/rtloader/rtloader/api.cpp#L194)
which pretty clearly shows the `dlopen` to load `libdatadog-agent-three.so`.

And to confirm, we can review the `LD_DEBUG=all` logs:

```
    225739:	binding file bin/agent/agent [0] to /home/ubuntu/dev/datadog-agent/dev/lib/libdatadog-agent-rtloader.so.1 [0]: normal symbol `make3'
    225739:	symbol=dlopen;  lookup in file=bin/agent/agent [0]
    225739:	symbol=dlopen;  lookup in file=/home/ubuntu/dev/datadog-agent/dev/lib/libdatadog-agent-rtloader.so.1 [0]
    225739:	symbol=dlopen;  lookup in file=/lib/aarch64-linux-gnu/libgcc_s.so.1 [0]
    225739:	symbol=dlopen;  lookup in file=/lib/aarch64-linux-gnu/libc.so.6 [0]
    225739:	binding file /home/ubuntu/dev/datadog-agent/dev/lib/libdatadog-agent-rtloader.so.1 [0] to /lib/aarch64-linux-gnu/libc.so.6 [0]: normal symbol `dlopen' [GLIBC_2.34]
    225739:
    225739:	file=libdatadog-agent-three.so [0];  dynamically loaded by /home/ubuntu/dev/datadog-agent/dev/lib/libdatadog-agent-rtloader.so.1 [0]
    225739:	find library=libdatadog-agent-three.so [0]; searching
```

We can see the linker resolving the `make3` symbol, which is the function that
contains the `dlopen` to `libdatadog-agent-three.so`.
We then see the binding of `dlopen` which is almost certainly what triggers the
next bit where `libdatadog-agent-three.so` is dynamically loaded by
`libdatadog-agent-rtloader.so.1`

~~My only unanswered question here is: Why don't I see the `dlopen` syscall in my
`strace` output?~~
Answer: simple, `dlopen` is not a syscall, its a library call. It is visible
with `ltrace`:

```
$ ltrace -e dlopen -f -o ltrace_out.txt bin/agent/agent run
226823 libdatadog-agent-rtloader.so.1->dlopen("libdatadog-agent-three.so", 257) = 0
```

One question you could ask here is: But we're setting the `RUNPATH` in the agent
binary correctly, so why isn't it being used to search for
`libdatadog-agent-three.so`?

The answer is that `RUNPATH` is _not_ used to search for dependencies loaded by
a library dependency (ie, transitive dependencies).

From `man dlopen`
>        If filename is NULL, then the returned handle is for the main program.  If filename contains a slash ("/"), then it is interpreted as a (relative or absolute) pathname.  Otherwise, the dynamic linker searches for the object as follows (see ld.so(8) for further details):
>      o   (ELF only) If the calling object (i.e., the shared library or executable from which dlopen() is called) contains a DT_RPATH tag, and does not contain a DT_RUNPATH tag, then the directories listed in the DT_RPATH tag are searched.
>      o   If, at the time that the program was started, the environment variable LD_LIBRARY_PATH was defined to contain a colon-separated list of directories, then these are searched.  (As a security measure, this variable is ignored for set-user-ID and set-group-ID programs.)
>      o   (ELF only) If the calling object contains a DT_RUNPATH tag, then the directories listed in that tag are searched.

So for this to work, `libdatadog-agent-rtloader` would need to set a `RPATH` or
`RUNPATH` tag, and we can see it does not:
```
$ readelf -a dev/lib/libdatadog-agent-rtloader.so | grep -iE "rpath|runpath" && echo "has it" || echo "no go"
no go
```

Where does that leave us?
We have a _runtime_ (not link-time) library dependency that isn't being
satisfied, at least in dev builds. I imagine this is not a problem in
"production" because we install libs to a location that is in the default
library search path.


## The solution
The best solution I can see here is for `libdatadog-agent-rtloader` to set the `RUNPATH` tag
to a location that includes the libraries it depends on.

I attempted to add this library dependency into cmake, however cmake complains
(rightly) that this is a cyclic dependency!
So its not allowed to have this dependency from `libdatadog-agent-rtloader` to
`libdatadog-agent-three` because there is already a dependency in the opposite
direction from `libdatadog-agent-three` to `libdatadog-agent-rtloader`.

I imagine this is why `libdatadog-agent-rtloader` loads `libdatadog-agent-three`
via `dlopen` in the first place.


This doesn't leave any obvious solutions that I can see unfortunately.

## The workaround
Use `LD_LIBRARY_PATH` manually to specify where `libdatadog-agent-three` can be
found.
`LD_LIBRARY_PATH=$PWD/dev/lib/ bin/agent/agent run`


Not great, but not too painful either.


