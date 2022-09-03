---
title: 6. Expressions
---

TODO: These are currently in no particular order, think about a helpful ordering:

6.1 String Conversions and `asString`

The `sclang` interpreter provides built-in functions for converting many SuperCollider types to strings. While not
strictly a part of the language specification, it is useful to document the details of some string conversions. Without
following the same conventions, other implementations could produce different strings describing identical results.

For instance, a call to `asString` on a `Float`, such as the one implied in this code:

```
pi.postln;
```

Results in a call to a C `printf` family function, with the `"%.14g"` as the format string. This [format
string](https://en.wikipedia.org/wiki/Printf_format_string) is a command to the C library to use the shorter version of
either exponential (e.g. 1.77e4) or regular notation, and to include no more than 14 digits after the decimal place,
rounding off the rest.

The string conversion algorithm then checks the numeric string for the presence of any non-digit character or minus
sign. If the string consists only of these characters, the algorithm appends a ".0" to the end of the string, to
demarcate it as a floating-point number.

6.2 Primitives

