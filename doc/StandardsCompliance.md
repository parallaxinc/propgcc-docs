---
layout: doc
title: Standards Compliance
---

*This document is still a work in progress, and not all standards
violations are necessarily listed herein!*

The Propeller GCC compiler + library conforms to the C90 (ISO/IEC
9899:1990) standard, as far we we know. Any failure to conform is a bug;
please report it as such. The compiler and library mostly conform to the
C99 standard (ISO/IEC 9899:1999) except as noted in [exceptions to the
C99 standard](#exceptions).

Implementation Defined Behavior
===============================

Here are some of the implementation defined behaviors as specified in
Annex J.3 of the C99 Standard. See also the GCC documentation for the
general GCC implementation behaviors; here we document details that are
specific to the Propeller implementation of GCC and to the default
library.

Environment
-----------

The `argv` argument to `main` is derived from the global array
`const char *__argv[]`, which is a NULL terminated list of strings. By
default this contains only a single empty string followed by NULL.

Characters
----------

There are 8 bits in a byte (3.6). `char` is equivalent to
`unsigned char` by default (6.2.5). We use UTF-8 encoding for multibyte
characters at run time; however, note that the "C" locale does not
support any operations on these characters. Use the "C.UTF-8" locale to
operate with multibyte characters and wide characters.

Exceptions to the C99 Standard
==============================

Propeller GCC does not implement all of the features of C99. The
exceptions are as follows.

Compiler
--------

GCC does not implement some of the `#pragma` directives for floating
point control mandated by the standard.

Library
-------

The library is missing some features of the C99 standard. The missing
features are outlined in this section.

### Missing features in stdio functions

The `printf` family of functions does not accept the `%a` and `%A`
specifiers for output of hexadecimal floating point. Similarly, `scanf`
does not accept those specifiers for input of floating point numbers in
hexadecimal notation.

To get floating point output from `printf` and related functions it is
necessary to link with `-lm`. (This is not actually a standard
violation, but it may be surprising.)

`printf` and `scanf` do not correctly round floating point numbers in
many cases.

### Missing features in stdlib functions

`strtod` and `strtof` do not always round their outputs correctly
according to the standard.

### Wide character I/O is not implemented

The wide character input and output functions from \<wchar.h\>, such as
`fgetwc`, `fputwc`, `wprintf`, and `wscanf`, are not implemented.
