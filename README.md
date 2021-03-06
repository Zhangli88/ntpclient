ntpclient
=========

Table of Contents
-----------------

* [Introduction](#introduction)
* [Building](#building)
* [Troubleshooting](#troubleshooting)
* [Usage](#usage)
* [Bugs](#bugs)
* [Compliance](#compliance)
* [Origin & References](#origin--references)


Introduction
------------

ntpclient is an NTP client for UNIX-like systems, [RFC 1305] and
[RFC 4330].  Its functionality is a small subset of [ntpd], [chrony],
[OpenNTPd], and [xntpd].  Since it is much smaller, it is also more
relevant for embedded systems in need for only a client.

The goal of ntpclient is not only to set your computer's clock right
once, but keep it there.


Building
--------

To build on Linux, type <kbd>make</kbd>.  Solaris and other UNIX users
will probably need to adjust the `Makefile` slightly.  It's not complex.
For changing the system clock frequency, only the Linux `adjtimex(2)`
interface is implemented at this time.  Non-Linux systems can only use
ntpclient to measure time differences and set the system clock, by way
of the POSIX 1003.1-2001 standard routines `clock_gettime()` and
`clock_settime()`.  Also, see section [Bugs](#bugs), below.

There are a few compile-time configurations possible, which require
editing the Makefile.  Either do or don't define:

    ENABLE_DEBUG
    ENABLE_REPLAY
    USE_OBSOLETE_GETTIMEOFDAY
    PRECISION_SIOCGSTAMP

Try it first without changing the default: that will give you a full-
featured ntpclient, that uses modern POSIX time functions, and works
reasonably with any Linux kernel.  There are comments in `ntpclient.c`
that you should read before experimenting with `PRECISION_SIOCGSTAMP`.


Troubleshooting
---------------

Some really old Linux systems (e.g., Red Hat EL-3.0 and Ubuntu 4.10)
have a totally broken POSIX `clock_settime()` implementation.  If you
get "clock_settime: Invalid argument" with <kbd>ntpclient -s</kbd>,
rebuild with `-DUSE_OBSOLETE_GETTIMEOFDAY`.  Linux systems that are even
older won't even compile without that switch set.


Usage
-----

    Usage: ntpclient [options]
     -c count     Stop after count time measurements. Default: 0 (forever)
     -d           Debug, or diagnostics mode  Possible to enable more at compile
     -g goodness  Stop after getting a result more accurate than goodness msec,
                  microseconds. Default: 0 (forever)
     -h hostname  NTP server, mandatory(!), against which to sync system time
     -i interval  Check time every interval seconds.  Default: 600
     -l           Attempt to lock local clock to server using adjtimex(2)
     -L           Use syslog instead of stdout for log messages, enabled
                  by default when started as root
     -n           Don't fork.  Prevents ntpclient from daemonizing by default
                  Only when running as root, does nothing for regular users
     -p port      NTP client UDP port.  Default: 0 ("any available")
     -q min_delay Minimum packet delay for transaction (default 800 microseconds)
     -s           Simple clock set, implies -c 1 unliess -l is also set
     -t           Trust network and server, no RFC-4330 recommended validation
     -v           Be verbose.  This option will cause time sync events, hostname
                  lookup errors and program version to be displayed
     -V           Display version and copyright information

Mortal users can use this program for monitoring, but not clock setting
(with the `-s` or `-l` switches).  The `-l` switch is designed to be
robust in any network environment, but has seen the most extensive
testing in a low latency (less than 2 ms) Ethernet environment.  Users
in other environments should study ntpclient's behavior, and be prepared
to adjust internal tuning parameters.  A long description of how and why
to use ntpclient is in the [HOWTO] file.  ntpclient always sends packets
to the server's UDP port 123.

One commonly needed tuning parameter for lock mode is `min_delay`, the
shortest possible round-trip transaction time.  This can be set with the
command line `-q` switch.  The historical default of 800 microseconds
was good for local Ethernet hardware a few years ago.  If it is set too
high, you will get a lot of "inconsistent" lines in the log file when
time locking (`-l` switch).  The only true future-proof value is 0, but
that will cause the local time to wander more than it should.  Setting
it to 200 is recommended on an end client.

The `test.dat` file that is part of the source distribution has 200
lines of sample output.  Its first few lines, with the output column
headers that are shown when the `-d` option is chosen, are:

      day    second   elapsed    stall      skew  dispersion  freq
    36765 00180.386    1398.0     40.3  953773.9       793.5  -1240000
    36765 00780.382    1358.0     41.3  954329.0       915.5  -1240000
    36765 01380.381    1439.0     56.0  954871.3       915.5  -1240000

* day, second: time of measurement, UTC, relative to NTP epoch (Jan 1, 1900)
* elapsed:     total time from query to response (microseconds)
* stall:       time the server reports that it sat on the request (microseconds)
* skew:        difference between local time and server time (microseconds)
* dispersion:  reported by server, see [RFC 1305] (microseconds)
* freq:        local clock frequency adjustment (Linux only, ppm*65536)

ntclient performs a series of sanity checks on UDP packets received, as
recommended by [RFC 4330].  If it fails one of these tests, the line
described above is replaced by `36765 01380.381 rejected packet` or, if
`ENABLE_DEBUG` was selected at compile time, one of:

    36765 01380.381  rejected packet: LI==3
    36765 01380.381  rejected packet: VN<3
    36765 01380.381  rejected packet: MODE!=3
    36765 01380.381  rejected packet: ORG!=sent
    36765 01380.381  rejected packet: XMT==0
    36765 01380.381  rejected packet: abs(DELAY)>65536
    36765 01380.381  rejected packet: abs(DISP)>65536
    36765 01380.381  rejected packet: STRATUM==0

To see the actual values of the rejected packet, start ntpclient with
the `-d` option; this will give a human-readable printout of every
packet received, including the rejected ones.  To skip these checks, use
the `-t` switch.

The file `test.dat` is suitable for piping into <kbd>ntpclient -r</kbd>.
There are more than 200000 samples (lines) archived for study.  They are
generally spaced 10 minutes apart, representing over three years of data
logging (from a variety of machines, and not continuous, unfortunately).
If you are interested, [contact Larry].

Also included is a version of the `adjtimex(1)` tool.  See its man page
and the [HOWTO] file for more information.

Another tool is `envelope`, which is a perl script that was used for the
lock studies.  It's kind of a hack and not worth documenting here.


Bugs
----

* Doesn't understand the LI (Leap second Indicator) field of an NTP packet
* Doesn't interact with `adjtimex(2)` status value
* Can't query multiple servers
* IPv4 only
* Requires Linux `select()` semantics, where timeout value is modified
* Always returns success (0)


Compliance
----------

Adherence to [RFC 4330] chapter 10, Best practices:

1. Enforced, unless someone tinkers with the source code
2. No backoff, but no retry either; this isn't TCP
3. Not in scope for the upstream source
4. Not in scope for the upstream source
5. Not in scope for the upstream source
6. Supported
7. Not supported
8. Not supported (scary opportunity to DOS the _client_)


Origin & References
-------------------

ntpclient was originally created by [Larry Doolittle] and is freely
available under the terms of the [GNU General Public License][GPL],
version 2.  For questions on the original, [contact Larry], he remains
the official upstream for ntpclient.

This is a fork maintained by [Joachim Nilsson], with the intent to add
common features like syslog support, more accessible documentation, and
other small things.

[GPL]: http://www.gnu.org/licenses/old-licenses/gpl-2.0.html
[ntpd]: http://www.ntp.org
[xntpd]: http://www.eecis.udel.edu/~mills/ntp/
[chrony]: http://chrony.tuxfamily.org/
[OpenNTPd]: http://www.openntpd.org
[RFC 1305]: http://tools.ietf.org/html/rfc1305
[RFC 4330]: http://tools.ietf.org/html/rfc4330
[Larry Doolittle]: http://doolittle.icarus.com/ntpclient/
[contact Larry]: larry@doolittle.boa.org
[HOWTO]: https://github.com/troglobit/ntpclient/HOWTO.md
[Joachim Nilsson]: http://troglobit.com
[TroglOS]: https://github.com/troglobit/troglos/
