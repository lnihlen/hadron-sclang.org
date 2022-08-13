---
title: SuperCollider Language Specification
geekdocCollapseSection: true
---

### SuperCollider Language Specification

As it is the canonical language implementation, `sclang` has long served as a complete specification of the
SuperCollider programming language. This language specification describes the expected behavior for any
valid SuperCollider language implementation. Having a precise language specification outside the interpreter
supports the development of alternative language implementations and obviates testing and validation of all implementations.

This document is a terse technical description of the language behavior in detail. Those looking for reference material
should consult the [SuperCollider Documentation](https://doc.sccode.org/). Outside of a few exceptions for built-in
types, this specification does not cover the SuperCollider class library.

This document is inspired by and follows the overall structure of the
[Python Language Reference](https://docs.python.org/3/reference/).

{{< toc-tree >}}
