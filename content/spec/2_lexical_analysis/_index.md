---
title: "2. Lexical Analysis"
---

The first stage in the compilation of a SuperCollider program is converting an input program string into a structured
sequence of symbols meaningful to the compiler. This stage starts with *lexing* the input into a stream of *tokens*,
then *parsing* those tokens into data structures more useful for program analysis and compilation. Parsing follows a
structured series of rules called *grammar*. The SuperCollider grammar is in [Appendix A](/spec/a_full_grammar).

This section of the specification discusses the details of lexing and parsing, defining rules and conventions for
interpretation that the SuperCollider user can expect the interpreter to follow when analyzing their input code.

## 2.1 Input Structure

SuperCollider programs are text strings structured as *tokens* separated by *whitespace*. Whitespace consists of spaces,
tabs, and newlines. Except for terminating `//` line comments, the interpreter treats newlines like all other
whitespace characters.

### 2.1.1 Comments

SuperCollider supports C-style line comments, meaning it ignores all input text after the double forward slash `//`
token until the end of the input line. SuperCollider also supports block comments, ignoring all input text between the
block opening token `/*` and the block closing token `*/`.

SuperCollider supports nested block comments, which is different than other C-style languages using the same comment
syntax [[scdoc](https://doc.sccode.org/Reference/Comments.html)]. The input string `/* /* */ */` is well-formed in
SuperCollider input, but in C, C++, or Java would cause a compilation error.

## 2.2 Operating Modes

The SuperCollider grammar is ambiguous without specifying if the input string is a class definition or interpreted code.
There are two file formats, `.sc` files for class definitions or extensions; and `.scd` for interpreted code that may
not contain class definitions or extensions.

At the *block* or *method body* level, the parser modes are identical. Where the parser modes differ is what they
expect for the highest-level input. While in interpreter mode, the parser expects an input of a sequence of zero or more
imperative expressions terminated by a semicolon. The user expects the interpreter to execute these statements promptly
after input. While in class definition input mode, the interpreter adds the declared classes to the class library.

### 2.2.1 Class Definition Input Mode

A SuperCollider class definition input must consist only of class definitions or extensions. In `sclang` the entire
class library is compiled at once on program startup or when the user requests recompilation, and this input mode is
ahead-of-time (AoT) compilation.

Here is an excerpt of the grammar relevant to class definitions and extensions:

```
classdef    : CLASSNAME superclass '{' classvardecls methods '}'
            | CLASSNAME '[' optname ']' superclass '{' classvardecls methods '}'
            ;

classextension  : '+' CLASSNAME '{' methods '}'
                ;

optname : %empty
        | IDENTIFIER
        ;

superclass  : %empty
            | ':' CLASSNAME
            ;

classvardecls   : %empty
                | classvardecls classvardecl
                ;

classvardecl    : 'classvar' rwslotdeflist ';'
                | 'var' rwslotdeflist ';'
                | 'const' constdeflist ';'
                ;

methods : %empty
        | methods methoddef
        ;

methoddef   : name '{' argdecls funcvardecls primitive methbody '}'
            | '*' name '{' argdecls funcvardecls primitive methbody '}'
            | binop '{' argdecls funcvardecls primitive methbody '}'
            | '*' binop '{' argdecls funcvardecls primitive methbody '}'
            ;
```

These rules define the overall structure of valid class definition and extension input. Some noteworthy observations:

#### 2.2.1.1 Superclass Specification

Class definitions can define an optional superclass; if unspecified, they derive from `Object`
[[scdoc](https://doc.sccode.org/Guides/WritingClasses.html#Inheriting)].

#### 2.2.1.2 Optional Storage Type Specifier

The `optname` within the square brackets is used in library code to document the stored element type for some Collection
objects. The brackets enable a collection-style initialization syntax such as `var a = Array[1, 2, 3];`
[[scdoc](https://doc.sccode.org/Guides/WritingClasses.html#Slotted%20classes)].

Internally `sclang` translates the square bracket initialization into a call to `Classname.new` followed by a series of
`instance = instance.add` calls for each element. `sclang` does not enforce the square bracket class
declaration for collection classes at compile time. It follows that a program can initialize any class instance with
the square brackets syntax, and if the object doesn't have an `add` method causes a runtime `ERROR: Message 'add' not
understood.` The converse is also true, and the compiler will silently accept a class declaration using the storage type
brackets but not defining an `add` method [source: code study].

#### 2.2.1.3 Class Extensions Contain Methods Only

Class extensions can only add methods and cannot add variables or constants. An input sequence must define a class
before extending it, or the compiler will issue an error [source: code study].

{{< hint >}}
**Note:** The `sclang` interpreter grammar does not accept a mixture of class definitions and class extensions. Input
files should consist of only one or the other. Hadron does not have this requirement, but all Hadron SuperCollider code
respects this convention to maintain compatibility with `sclang`.
{{< /hint >}}

#### 2.2.1.4 Valid method names

Method names can be any identifier (including control flow keywords like `if` and `while`) or binop name.

#### 2.2.1.5 Ambiguous Parse With Binop Class Method Names

Accepting binops as method names leads to an ambiguous parse for any binop name starting with an asterisk `'*'`. To
resolve, we require a space between the asterisk signifying a class method and the binop name. For example:

```
AmbiguousBinop {
    * { }     // defines an instance method named '*'
    ** { }    // defines an instance method named '**'
    * * { }   // defines a class method named '*'
}
```

#### 2.2.1.6 Primitives Specification

A method body may contain a primitive name, and the interpreter calls any C++ code bound to that primitive, ignoring the
SuperCollider code in the method body.

**TODO:** there are comments in the class library that may imply that if the interpreter doesn't find a bound primitive
function, it will execute any sclang code following the primitive specifier. Verify this and update the documentation.

### 2.2.2 Interpreter Input Mode

While in interpreter mode, the parser is expecting expressions as input:

```
cmdlinecode : '(' funcvardecls1 funcbody ')'
            | funcvardecls1 funcbody
            | funcbody
            ;

```

Note that `cmdlinecode` only matches against code blocks surrounded by parenthesis that include at least one variable
declaration. The reason for this is that the grammar is ambiguous with `Event` declarations. Top-level blocks without
variables map against the `funcbody` rule, which matches against a sequence of expressions `exprseq`. One of the grammar
rules for a single expression `expr1` matches parentheses surrounding another expression sequence.

```
funcbody    : funretval
            | exprseq funretval
            ;

expr1   :  '(' exprseq ')'
```

## 2.3 Other Tokens

## 2.4 Identifiers and Keywords

## 2.5 Literals

## 2.6 Operators

## 2.7 Delimiters

*Class Names* are identifiers that must start with an upper-case alphabet character, and classes must have a unique name
within the entire class library compilation. If `sclang` encounters a duplicate class name while compiling the class
library, it issues `ERROR: duplicate Class found: '<classname>'` along with providing the filenames where it found the
duplicate class definitions.

