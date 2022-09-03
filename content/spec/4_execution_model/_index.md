---
title: "4. Execution Model"
---

## 4.1 Structure of a program

## 4.2 Naming and Binding

### 4.2.3 Assignment of Parameter Values to Message Arguments

SuperCollider matches messages on type and selector name only, it makes no consideration of the number, name, or type of
arguments provided to that message. At compile time the message recipient type may be unknown, so SuperCollider follows
the following mechanism to assign parameter values to message arguments:

 1. **Default First Argument**: The reference to the callee provided by `this` is always the first argument to every
    method.
 2. **In-Order Argument Assignment**: If the method expects more arguments than provided by the caller, the interpreter
    assigns the default values of each argument to the missing in-order arguments. If the method didn't specify a
    default argument the default is `nil`. If the caller provided more in-order arguments the interpreter ignores them.
 3. **Keyword Argument Assignment**: The interpreter then processes the keyword arguments in order, assigning each
    value to the argument name matching the keyword. If no argument name matches a provided keyword, the interpreter
    issues a keyword argument not found warning.

This order of operations means allows for the redundant provision of arguments, with the last revision of the argument
taking priority. See the following examples:

```
(
var f = { |foo = 1, bar = 2, bazz|
	"foo: %, bar: %, bazz: %".format(foo, bar, bazz).postln;
};

f.value();                                 // foo: 1, bar: 2, bazz: nil
f.value(3, 4, 5);                          // foo: 3, bar: 4, bazz: 5
f.value(3, 4, 5, 6, 7, 8, 9, 10);          // foo: 3, bar: 4, bazz: 5
f.value(bazz: 5, bar: 4, foo: 3);          // foo: 3, bar: 4, bazz: 5
f.value(1, 2, 3, foo: 4, bar: 5, bazz: 6); // foo: 4, bar: 5, bazz: 6
f.value(bazz: 1, bazz: 2, bazz: 3);        // foo: 1, bar: 2, bazz: 3
// WARNING: keyword arg 'bork' not found in call to function.
f.value(bork: 99);                         // foo: 1, bar: 2, bazz: nil
)
```

## 4.2 Exceptions
