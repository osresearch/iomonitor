I/O monitoring tool
====

`iomonitor` uses `/usr/bin/strace` to track file descriptor activity of a process
and all of its children, and creates a time-series of the I/O operations. This
is useful for figuring out how much you're writing/reading, to/from where, and
to/from which remote servers.  These read/write calls can be used for tracing
database calls, network activity, finding excess logging and so on.
Additionally, the timeseries `iomonitor` produces can be processed to generate
interesting stats on number of calls, bandwidth used, etc.

Usage
---
    Usage:
       ./iomonitor [options] command [command-options...]
       ./iomonitor [options] -p pid
       ./iomonitor [options] -i strace.log
    
    Options:
        -o | --output output-file     Write the csv to an output file
        -i | --input input-file       Read strace log file
        -p | --pid pid                Attach to existing process
        -v | --verbose                Print all system calls
    
    strace log files should be produced with -f -T -tt -v to ensure
    that all the important values are recorded.

Examples
---
Here is an example run, showing how it tracks subprocesses invoked by a subshell --
you can see the new PID for each process:

    % ./iomonitor bash -c 'wc /etc/fstab ; /bin/echo foo > /dev/null'
    time,dt,pid,fd,dir,bytes,file,port
    17:51:39.778259,0.000007,3853,3,R,832,"/lib/libncurses.so.5",
    17:51:39.778425,0.000006,3853,3,R,832,"/lib/libdl.so.2",
    17:51:39.778581,0.000009,3853,3,R,832,"/lib/libc.so.6",
    17:51:39.779379,0.000062,3853,3,R,1024,"/proc/meminfo",
    17:51:39.781562,0.000005,3854,3,R,832,"/lib/libc.so.6",
    17:51:39.781925,0.000008,3854,3,R,2022,"/etc/fstab",
    17:51:39.781961,0.000006,3854,3,R,0,"/etc/fstab",
    17:51:39.782031,0.000016,3854,1,W,26,stdout,
    17:51:39.783490,0.000010,3855,3,R,832,"/lib/libc.so.6",
    17:51:39.784430,0.000063,3855,1,W,4,"/dev/null",

We can see the various libraries that are read, the first `read()` of `/etc/fstab`,
the second `read()` that returns 0, indicating end of file, and then a write of
23 bytes to `stdout`. Then we can see the echo process writes 4 bytes to
`/dev/null`.

For a network process it attempts to match up the sockets with their peers. For
instance, here is netcat talking to sshd on a remote machine:

    % ./iomonitor nc kremvax.su 22 < /dev/null
    time,dt,pid,fd,dir,bytes,file,port
    17:56:14.265905,0.000005,5398,3,R,832,"/lib/libc.so.6",
    17:56:14.266287,0.000007,5398,3,R,308,"/etc/resolv.conf",
    17:56:14.266316,0.000005,5398,3,R,0,"/etc/resolv.conf",
    17:56:14.266480,0.000008,5398,3,R,308,"/etc/resolv.conf",
    17:56:14.266503,0.000005,5398,3,R,0,"/etc/resolv.conf",
    17:56:14.266739,0.000008,5398,3,R,500,"/etc/nsswitch.conf",
    17:56:14.266766,0.000005,5398,3,R,0,"/etc/nsswitch.conf",
    17:56:14.266904,0.000005,5398,3,R,832,"/lib/libnss_files.so.2",
    17:56:14.267072,0.000008,5398,3,R,92,"/etc/host.conf",
    17:56:14.267094,0.000005,5398,3,R,0,"/etc/host.conf",
    17:56:14.267204,0.000007,5398,3,R,413,"/etc/hosts",
    17:56:14.267231,0.000005,5398,3,R,0,"/etc/hosts",
    17:56:14.267371,0.000005,5398,3,R,832,"/lib/libnss_dns.so.2",
    17:56:14.267493,0.000005,5398,3,R,832,"/lib/libresolv.so.2",
    17:56:14.267762,0.000365,5398,3,W,55,127.0.53.1,53
    17:56:14.268210,0.000007,5398,3,R,71,127.0.53.1,53
    17:56:14.268412,0.000008,5398,3,R,4096,"/etc/services",
    17:56:14.268500,0.000006,5398,3,R,4096,"/etc/services",
    17:56:14.269255,0.000017,5398,0,R,0,stdin,
    17:56:14.275372,0.000035,5398,3,R,44,192.168.212.172,22
    17:56:14.275536,0.000030,5398,1,W,44,stdout,

Here we can see the host name lookup, first by consulting `/etc/resolv.conf` and then
`/etc/host.conf` and `/etc/hosts`, followed by DNS resolution on port 53 to 127.0.53.1. The
process reads the OpenSSH header and writes it to `stdout`, then closes the
socket.

Issues
===
* `iomonitor -p $PID` can't report file names for files or sockets that were opened before the tracer was attached. This is due to a patch for [a subtle security vulnerabilty](https://lwn.net/Articles/359219/) in handling `/proc/$PID/fd/$FD`
* It does not track `mmap()` and can not track page flushes, so there is a bit of a blind spot with regards to memory mapped I/O.
* `splice()` and variants are not tracked. This should be easy to add if there are any examples that use it.
* Various `fcntl()` options, such as close-on-exec (`FD_CLOEXEC`) are not tracked.
* There are likely bugs in the way `clone()`'ed file descriptor tables are updated. Keeping track of which fd's are open in which processes is more complicated than it should be.
* There need to be test cases written that do lots of `fork()` and `dup()` calls.
