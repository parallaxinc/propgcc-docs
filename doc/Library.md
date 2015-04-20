---
layout: doc
title: Propeller GCC Library
---

The Propeller GCC Library is an implementation of the standard C
library. It attempts to conform to the C89 version of the standard, with
some C99 extensions. This documentation assumes that the user is
familiar with the C language and C libraries; we will focus on the
Propeller specific details of this library rather than general usage.

Environment
===========

The `main` function has prototype:

      int main(int argc, char *argv[], char *environ[]); 

`argv` are the arguments to the program; this is a NULL terminated list
of pointers to strings. Conventionally the first argument (`argv[0]`) is
the name of the program. Since the Propeller has no operating system, it
is up to the application to supply arguments. This is done with the
global variable `char *_argv[]`, which is passed directly to main as
`argv`.

`argc` is a count of the arguments to the program, and is initialized
from `_argv` automatically by the startup code.

`environ` is a list of environment variables, also NULL terminated. As
with `argv`, this is initialized from a global variable, in this case
`char *_environ[]`. The `getenv` library function also uses `_environ`
to look for variables.

The strings in `_environ` should be of the form `VAR=value`, where "VAR"
is the name of the environment variable and "value" is its value. At
present `TZ` is the only environment variable that is actually used by
the library (for the [time zone](TimeZOnes)), but some applications may
also look at the environment and require customized settings.

Locales and Character Sets
==========================

The library starts up in a plain "C" locale which uses an 8 bit ASCII
character set. Only the 7 bit ASCII subset is recognized by the
character classification functions, but character conversion to wide
characters uses ISO 8859-1 (so a character like `'\xa0'` converts to the
wide character `L'\u00a0'` when processed by `mbtowc` or a similar
function).

The library also provides a "C.UTF-8" locale which is the same as the
"C" locale except that the encoding is UTF-8 rather than ISO 8859-1.
UTF-8 allows wide characters to be represented by multiple bytes in a
lossless way. Only the multibyte to/from wide character conversion
routines (including the ones inside functions such as printf) are
affected; character classification functions like `isalpha` still only
recognize the ASCII subset of Unicode.

To set the UTF-8 character encoding, do

      setlocale(LC_CTYPE, "C.UTF-8");

or just

      setlocale(LC_ALL, "");

in your `main` function. To switch back to ASCII, do

      setlocale(LC_CTYPE, "C");

Note that the locale setting affects all threads in a program. Also, the
locale and multibyte/wide character functions are not available in
`-mcog` mode.

Stdio
=====

The standard I/O library implementation ("stdio") is fairly complete.
Unlike in many other systems, stdio is fundamental; it is not
implemented on top of POSIX like functions such as `open` and `close`.
Rather, there is a [driver layer](#drivers) which implements the files
that `fopen` operates on.

FILE
----

The stdio FILE structure is in most respects fairly conventional. It
contains flags for how the file was opened, a pointer to a buffer, and a
buffer size. Unlike many libraries, the Propeller GCC library does not
automatically use `malloc` to allocate the FILE buffer; rather it
defaults to a small character array inside the FILE structure. Some
drivers (particularly for disks and similar devices) may override this
to provide a bigger buffer.

The FILE structure also contains an array of driver specific variables
(in `drvarray`) which drivers may use for any purpose they wish.

fopen
-----

The `fopen` function takes a string describing the device or file that
is to be opened. In the Propeller GCC library this string should start
with a prefix which specifies the [driver](#drivers), followed by a
string which is passed to the driver. The prefix should end with a
colon. For example, to open the simple serial port driver for reading
and writing at 9600 baud, with transmit pin 5 and receive pin 5, do:

      FILE *f = fopen("SSER:9600,5,5", "r+");

Here `"SSER:"` is the prefix for the simple serial device, and
`"9600,5,5"` are the parameters for the device. Similarly, opening a
file for writing on a DOS file system might be achieved by something
like:

      FILE *f = fopen("DOS:log.txt", "w");

### \_\_fopen\_driver

The `__fopen_driver` function is like `fopen`, but has additional
parameters directly specifying the handle and driver to use, rather than
relying on a prefix. So for example:

      stdin = freopen(stdin, "SSER:9600,5,5", "r");

could also be called as:

      extern _Driver _SimpleSerialDriver;
      stdin = __fopen_driver(stdin, &_SimpleSerialDriver, "9600,5,5", "r");

`__fopen_driver` is used internally by `fopen` and `freopen`, but may
sometimes be useful to application programmers who need direct control
over driver and handle usage.

Raw vs. Cooked mode
-------------------

Some drivers (notably the serial drivers) support a "cooked" mode of
input. In this mode characters are echoed when they are received, and
backspace (or delete) may be used to delete characters. Additionally,
input is gathered until either the requested number of characters is
received or a newline is seen.

Cooked mode is selected by setting the `_IOCOOKED` bit in a `FILE`
structure's `_flag` field. If the bit is clear, then "raw" mode is used,
where there is no echoing and no interpretation of special characters.

Cooked mode is the default. To read characters directly from stdin
without processing, do something like:

      stdin->_flag &= ~_IOCOOKED;

This also causes stdin to effectively be unbuffered, since the terminal
driver only processes 1 character at a time in raw mode.

Non-blocking Reads
------------------

Terminal style drivers also support the ability to attempt to read a
byte and return immediately if none is available. This is accomplished
by setting the `_IONONBLOCK` bit in the `_flag` field of the `FILE`
structure. For example, to check to see if a byte is available from
stdin, you could do:

      stdin->_flag |= _IONONBLOCK;
      c = getchar();

If a character is available then `c` will be set to that character,
otherwise it will be set to `-1`. Note that this is the same as `EOF`;
in non-blocking mode there is no way to distinguish between end of file
and no character currently available (for terminals there never is
really an end of file).

Printf and Code Size
--------------------

The `printf` function (and related functions such as `sprintf`) can
consume quite a bit of memory. To help reduce the footprint of these
functions, the library provides several options:

-   `__simple_printf` is a simple version of printf with support for the
    most common output formats (hex and decimal integers, strings, and
    characters) but without support for floating point, 64 bit integers,
    right justification, precision, alternate output formats, and other
    advanced features. `__simple_printf` is also not thread safe, so it
    should be used only in single threaded applications. You can use
    this version of printf in your code by adding
    `-Dprintf=__simple_printf` to the command line. The simple printf is
    by far the smallest version of printf, but obviously it is limited.
-   The default `printf` function supports all integer output formats
    and options, but leaves out floating point output (the `e`, `f`, and
    `g` formats). This is the one found in libc.
-   The full version of `printf`, including floating point support, is
    found in the math library libm, which is selected by adding `-lm` to
    the command line of the final build step.

### Printf in -mcog mode

The stdio functions in -mcog mode are even simpler. Only the equivalent
of `__simple_printf` is available in -mcog mode (there isn't enough room
for more). Only the standard output is available in -mcog.

Time Related Functions
======================

There is a normal set of time related functions in the library. The
defines for these are mostly in `<time.h>`.

Types
-----

### time\_t

The `time_t` type is used for keeping track of dates. As in POSIX, it
contains the value
`86400*(days since Jan. 1, 1970) + (seconds   elapsed today)`, all in
the Coordinated Universal Time (UTC) time zone. This is not the same as
seconds elapsed since Jan. 1, 1970, because leap seconds are ignored (as
in POSIX).

Note that unless there is a real time clock on the board (with an
appropriate driver!) and the time zone has been set correctly, the
values stored in `time_t` are almost certainly bogus and should not be
relied on.

### clock\_t

The `clock_t` type contains a count of processor cycles. It is a 32 bit
type, and so on a typical 80MHz board it will roll over in about 54
seconds.

Time Zones
----------

Some functions (such as `localtime` or `strftime`) make use of time zone
settings. The time zone is set automatically from the `TZ` [environment
variable](#environment). This specifies the standard time name, offset
from GMT, and (optionally) a daylight savings time name. For example,
include `"TZ=EST5EDT"` as one of the strings in the `_environ` array for
the US Eastern time zone.

Only the North American daylight savings times rules are recognized at
present, and only the current ones (as of 2011) are implemented.

Real Time Clocks
----------------

The interface to real time clocks is via two function pointers:

        #include <sys/rtc.h>
        int (*_rtc_gettime)(struct timeval *tv);
        int (*_rtc_settime)(const struct timeval *tv);
      

These point to functions to get and set the current time, respectively.
The `struct timeval` structure is described by POSIX, and has two fields
`tv_sec` (giving the current time in `time_t` format) and `tv_usec`
(giving the microseconds elapsed since `tv_sec` last changed; during a
leap second this might go up to 2000000000).

There is a default clock that simply uses the built-in cycle counter. At
a typical clock rate of 80MHz this counter overflows in 54 seconds. The
default RTC driver uses a 64 bit tick counter to compensate for this,
but a time retrieval function (such as time()) must be called more often
than the counter overflows in order for it to keep the high word of the
tick counter accurate.

There is also a mechanism to use another cog to update the cycle
counter, which will remove the requirement for frequent calling of the
clock functions. To do this, call the `_rtc_start_timekeeping_cog()`
routine, which will start a cog running whose entire purpose is to keep
the clock up-to-date.

Alternatively, if you have spare time on one of your cogs, you can kick
the built-in cycle counter yourself (without the expensive call to the
time() routine which performs several 64-bit math operations). To do
this, set the global variable `_default_ticks_updated` to a non-zero
value, and call the \_update\_default\_ticks() routine on a regular
basis.

If you have an alternate way to keep time (such as a hardware chip like
the DS1318), replace the built-in drivers by assigning the global
`_rtc_gettime` and `_rtc_settime` pointers to your own gettime() and
settime() routines.

Device Drivers
==============

Drivers for the [standard I/O library](#stdio) may be written by the
user and installed in the system. The library comes with some default
drivers, which are described below.

Installing Drivers
------------------

In the default configuration only the simple serial driver is installed.
This is easily modified by the program at compile time. The global
variable `_driverlist` is a NULL terminated array of pointers to driver
structures that should be included. So for example to include both the
simple serial driver and full duplex serial driver, you would do:

    #include <driver.h>
    extern _Driver _SimpleSerialDriver;
    extern _Driver _FullDuplexSerialDriver;

    _Driver *_driverlist[] = {
      &_SimpleSerialDriver,
      &_FullDuplexSerialDriver,
      NULL
    };

The first driver in the list is the "default" driver, and is used to
open the `stdin`, `stdout`, and `stderr` FILE descriptors. If you want
finer control over how these files are opened, you can either use the
`freopen` function on them at the start of your code, or define a
function `_InitIO` which will set up the file handles.

Drivers Provided in the Library
-------------------------------

### Simple Serial Driver (Prefix SSER:)

The simple serial port driver (`_Driver _SimpleSerialDriver`) is a half
duplex driver for generic serial ports. It defaults to doing I/O at the
rate specified in the `_baud` variable (normally 115200 on most boards),
with transmit pin `_txpin` and receive pin `_rxpin`. These defaults may
be overridden by the string passed to `fopen`. This string contains the
baud rate, optionally followed by a comma and the receive pin number and
then another comma and the transmit pin number, all in decimal notation.
So for example the default

    FILE *f=fopen("SSER:", "r");

is equivalent to

    FILE *f=fopen("SSER:115200,31,30", "r");

on most boards.

The simple serial driver does not require any additional cogs to
operate.

Note that because the simple serial driver runs completely in the local
cog, it shares the cog's DIRA and OUTA register with the calling code.
Therefore, avoid overwriting the DIRA/OUTA bits corresponding to the
simple serial driver's TX and RX pins. You can do this when setting or
clearing bits for other I/O devices by using bit-setting and bit-masking
instead of direct assignment (e.g. `DIRA |= 3` or `DIRA &= ~3` instead
of `DIRA = 3` or `DIRA = 0`). This is always a good idea, but failing to
do this in any code that uses the simple serial driver will result in
unpredictable I/O.

### Full Duplex Serial Driver (Prefix FDS:)

The full duplex serial driver (`_Driver _FullDuplexSerialDriver`) takes
over a cog in order to provide reliable full duplex communication on a
serial line. A separate cog is needed for every set of transmit and
receive pins in use. Like the [simple serial driver](#simpleserial), the
full duplex serial driver defaults to the values in the board specific
`_baud`, `_txpin`, and `_rxpin` variables, and these may be overridden
by passing a string containing *baud*,*rxpin*,*txpin*.

Writing New Drivers
-------------------

It is fairly straightforward to write a device driver. Drivers are
represented by a `_Driver` structure, as defined in `<driver.h>`. This
structure has the following fields, which must be set up by the driver
code:

*   `const char *prefix`

    The file prefix that tells `fopen` which driver to use; this should
    end in a colon `:`

*   `int (*fopen)(FILE *fp, const char *name, const char *mode)`

    A hook called by the `fopen` function. The `name` parameter contains
    the portion of the name passed to `fopen` that comes *after* the
    prefix. `fp` is the `FILE` structure that is being set up. The hook
    should put any driver specific variables that are required into the
    `drvarg` array in `fp`; if more than 4 longs are required, it should
    allocate memory and put a pointer into `drvarg`. The `fopen` hook
    should return 0 on success, and -1 on failure; in the latter case it
    should set `errno` to an appropriate value.

*   `int (*fclose)(FILE *fp)`

    A hook called when a file is being closed; this should free any
    memory allocated by the `fopen` hook.

*   `int (*read)(FILE *fp, unsigned char *buf, int size)`
    `int (*write)(FILE *fp, unsigned char *buf, int size)`

    These are the I/O functions, called to actually read and write data.
    Many drivers will have to implement their own versions of these
    hooks, but some may be able to use the default functions
    `_null_read` and `_null_write` (which return EOF and do nothing,
    respectively) or `_term_read` and `_term_write` (which do terminal
    I/O using the `getbyte` and `putbyte` hooks, see below).

*   `int (*seek)(FILE *fp, long offset, int whence)`

    A hook called to change the current read or write position in the
    file. `whence` is one of `SEEK_SET`, `SEEK_CUR`, or `SEEK_END`,
    which have the same meanings as they do for the `fseek` function. A
    driver need not implement the `seek` hook; if this hook is set to
    NULL, then any attempt to seek on the file will fail and `errno`
    will be set to `ENOSEEK`.

*   `int (*remove)(const char *name)`

    A hook called to remove a file. File removal is not yet implemented,
    so most drivers will simply set this to NULL.

*   `int (*getbyte)(FILE *fp)`

    An optional hook for single character I/O; reads and returns a
    single character. This hook is used by the `_term_read` function for
    cooked terminal I/O; if a driver does not wish to use `_term_read`
    for its reader function, it may set this hook to NULL.

*   `int (*putbyte)(FILE *fp)`

    An optional hook for single character I/O. This hook is used by the
    `_term_read` and `_term_write` functions for cooked terminal I/O; if
    a driver does not wish to use these functions, it may set this hook
    to NULL.

After the driver is written, it must also be
[installed](#InstallDrivers).

Cog mode drivers
----------------

In `-mcog` mode the stdio driver facility is not available (it is too
large to fit in the 2K of internal space in a cog). Instead, the stdio
library for `-mcog` uses just two functions, `putchar` and `getchar`.
The default `putchar` implementation writes characters to the serial
port (using code very much like the simple serial driver).

Note that `fopen` and similar functions are not supported in `-mcog`
mode. Only `stdin` and `stdout` are supported.

Threads
=======

Sometimes one wants to run some code on another cog. There are two ways
to do this: to run a copy of the LMM (or XMM) interpreter on the other
cog, or to run native cog code (compiled with `-mcog`) on the other cog.
Both of these amount to running a thread on the other cog

There are also times when one wants to run more threads of execution
than there are cogs. The **pthreads** functions provide a way to do
this. Pthreads can run on other cogs or on the same cog.

LMM on Another Cog
------------------

Running compiled code in another thread is very straightforward in
`-mlmm`, `-mcmm`, or `-mxmmc` modes. To start a new thread running
function `foo` with argument `arg` do:

        int cog = _start_cog_thread(sp, foo, arg, tls_ptr);
      

Here `sp` points to the top of the new thread's stack, and `tls_ptr`
points to a new `_thread_state_t` to hold the thread local storage for
this thread (that is, the library variables that need to be kept
per-thread rather than globally; for example `errno`). The new cog id is
returned in `cog`. If no cogs are available, -1 is returned.

Note that `_start_cog_thread` is not recommended for use in XMM mode,
because there is no data cache coherency mechanism; thus, any use of
variables in external memory by more than one COG can lead to data
corruption. In the other modes (LMM, CMM, and XMMC) the data is stored
in hub memory and hence no external RAM cache is required.

There are several synchronization methods available for code running on
different cogs. The most basic is the `__lock/__unlock` mechanism
provided in `sys/thread.h`. These locks are spin locks; a cog that is
waiting on the lock loops until it becomes available. More sophisticated
(blocking) lock mechanisms are available in the [pthreads](#Pthreads)
library.

To use a spin lock, simply declare a variable as:

        #include <sys/thread.h>
        _atomic_t mylock;
      

and then lock and unlock it like:

        __lock(&mylock);
        // .. do some things
        __unlock(&mylock);
      

It is important to hold locks only for as long as required, and to
unlock them immediately afterwards. Also, the locks are not recursive.
If a thread attempts to lock a variable which it has already locked, it
will hang.

Pthreads
--------

It is also possible to use the standard POSIX pthreads functions in
`pthreads.h` to start and stop threads. pthreads will run on other cogs
if available; if none are available then the threads will have to share
cogs. The number of pthreads that can run simultaneously is limited only
by memory.

When a new pthread is started, the library attempts to start it on a new
cog. If no new cog can be allocated, the library runs it on an existing
cog. However, multiple threads on the same cog must cooperate; that is,
a thread will keep running until it calls some blocking function (such
as `sleep`) or calls `pthread_yield`.

To compile a program to use pthreads, link with `-lpthread`. The pthread
functions themselves, and some slight modifications to standard
functions to support pthreads, are in that library.

Native Cog C
------------

To run some C code directly in a cog (not in LMM mode) we have to
compile it with the `-mcog` flag. We also have to do some linker tricks
to make sure that some definitions in the library and kernel do not
conflict with definitions in the main LMM code.

Typically we would put all the C code for a cog in one file, and compile
it with a set of commands like:

        propeller-elf-gcc -mcog -r -o mydriver.o -Os mydriver.c
        propeller-elf-objcopy --localize-text --rename .text=mydriver.cog mydriver.cog
      

The first line creates the cog program; the `-r` option tells gcc that
we want it to be relocatable (we will be further linking it with the
main LMM program). The second line causes all of the global text segment
symbols (function definitions and such) to become local to the cog
program so they will not interfere with similar symbols in the LMM
program. It also renames the .text section (where the code is) to
mydriver.cog. The linker knows that any section whose name begins or
ends with **.cog** is an overlay intended to be run inside a cog.

By convention COG programs are named `foo.cog`, but this is just a
convention. The important thing to the linker is the name of the section
containing the cog code, which must start or end with .cog. The linker
also creates symbols telling where the code starts (and ends) in the
file. For a section named `foo.cog`, it creates symbols
`__load_start_foo_cog` and `__load_stop_foo_cog` (note that since period
is not a valid character in a C identifier, it is translated to
underscore if it is in the middle of a name and stripped if it is at the
start of the name). Similarly for a section named `.cog.my.driver` it
creates symbols `__load_start_cogmydriver` and
`__load_stop_cogmydriver`. We can use these symbols to load the overlays
into a cog. For example, to start the mydriver.cog code and pass it in a
pointer to a mailbox, we would do:

        #include <propeller.h>
        cognew(__load_start_mydriver_cog, &mailbox);
      
