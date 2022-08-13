---
title: "Appendix A: Full Grammar Specification"
---

I extracted the following grammar from the Hadron parser Bison file by removing the embedded C++ and editing some of the
rule names for clarity. I based this file off the Bison parser in `sclang`, but I've made a few changes:

 * Parse `while` and `if` statements as separate constructions, rather than treating them as messages
 * Allow mixing of class definitions and class extensions in the same input file

```plain
root    : classorclassexts
        | INTERPRET cmdlinecode
        ;

classorclassexts    : %empty
                    | classorclassexts classorclassext
                    ;

classorclassext : classdef
                | classextension
                ;

classdef    : CLASSNAME superclass OPENCURLY classvardecls methods CLOSECURLY
            | CLASSNAME OPENSQUARE optname CLOSESQUARE superclass OPENCURLY classvardecls methods CLOSECURLY
            ;

classextension  : PLUS CLASSNAME OPENCURLY methods CLOSECURLY
                ;

optname : %empty
        | IDENTIFIER
        ;

superclass  : %empty
            | COLON CLASSNAME
            ;

classvardecls   : %empty
                | classvardecls classvardecl
                ;

classvardecl    : CLASSVAR rwslotdeflist SEMICOLON
                | VAR rwslotdeflist SEMICOLON
                | CONST constdeflist SEMICOLON
                ;

methods : %empty
        | methods methoddef
        ;

methoddef   : name OPENCURLY argdecls funcvardecls primitive methbody CLOSECURLY
            | ASTERISK name OPENCURLY argdecls funcvardecls primitive methbody CLOSECURLY
            | binop OPENCURLY argdecls funcvardecls primitive methbody CLOSECURLY
            | ASTERISK binop OPENCURLY argdecls funcvardecls primitive methbody CLOSECURLY
            ;

optsemi : %empty
        | SEMICOLON
        ;

optcomma    : %empty
            | COMMA
            ;

optequal    : %empty
            | ASSIGN
            ;

optblock    : %empty
            | block
            ;

funcbody    : funretval
            | exprseq funretval
            ;

cmdlinecode : OPENPAREN funcvardecls1 funcbody CLOSEPAREN
            | funcvardecls1 funcbody
            | funcbody
            ;

// TODO: why not merge with funcbody?
methbody    : retval
            | exprseq retval
            ;

primitive   : %empty
            | PRIMITIVE optsemi
            ;

retval  : %empty
        | CARET expr optsemi
        ;

funretval   : %empty
            | CARET expr optsemi
            ;

blocklist1  : blocklistitem
            | blocklist1 blocklistitem
            ;

blocklistitem   : blockliteral
                | listcomp
                ;

blocklist   : %empty
            | blocklist1
            ;

msgsend : IDENTIFIER blocklist1
        | OPENPAREN binop2 CLOSEPAREN blocklist1
        | IDENTIFIER OPENPAREN CLOSEPAREN blocklist1
        | IDENTIFIER OPENPAREN arglist1 optkeyarglist CLOSEPAREN blocklist
        | OPENPAREN binop2 CLOSEPAREN OPENPAREN CLOSEPAREN blocklist1
        | OPENPAREN binop2 CLOSEPAREN OPENPAREN arglist1 optkeyarglist CLOSEPAREN blocklist
        | IDENTIFIER OPENPAREN arglistv1 optkeyarglist CLOSEPAREN
        | OPENPAREN binop2 CLOSEPAREN OPENPAREN arglistv1 optkeyarglist CLOSEPAREN
        | CLASSNAME OPENSQUARE arrayelems CLOSESQUARE
        | CLASSNAME blocklist1
        | CLASSNAME OPENPAREN CLOSEPAREN blocklist
        | CLASSNAME OPENPAREN keyarglist1 optcomma CLOSEPAREN blocklist
        | CLASSNAME OPENPAREN arglist1 optkeyarglist CLOSEPAREN blocklist
        | CLASSNAME OPENPAREN arglistv1 optkeyarglist CLOSEPAREN
        | expr DOT OPENPAREN CLOSEPAREN blocklist
        | expr DOT OPENPAREN keyarglist1 optcomma CLOSEPAREN blocklist
        | expr DOT IDENTIFIER OPENPAREN keyarglist1 optcomma CLOSEPAREN blocklist
        | expr DOT OPENPAREN arglist1 optkeyarglist CLOSEPAREN blocklist
        | expr DOT OPENPAREN arglistv1 optkeyarglist CLOSEPAREN
        | expr DOT IDENTIFIER OPENPAREN CLOSEPAREN blocklist
        | expr DOT IDENTIFIER OPENPAREN arglist1 optkeyarglist CLOSEPAREN blocklist
        | expr DOT IDENTIFIER OPENPAREN arglistv1 optkeyarglist CLOSEPAREN
        | expr DOT IDENTIFIER blocklist
        ;

listcomp: OPENCURLY COLON exprseq COMMA qualifiers CLOSECURLY
        | OPENCURLY SEMICOLON exprseq COMMA qualifiers CLOSECURLY
        ;

qualifiers  : qual
            | qualifiers COMMA qual
            ;

qual: IDENTIFIER LEFTARROW exprseq
    | exprseq
    | VAR IDENTIFIER ASSIGN exprseq
    | COLON COLON exprseq
    | COLON WHILE exprseq
    ;

if  : IF OPENPAREN exprseq COMMA exprseq COMMA exprseq optcomma CLOSEPAREN
    | IF OPENPAREN exprseq COMMA exprseq optcomma CLOSEPAREN
    | expr DOT IF OPENPAREN exprseq COMMA exprseq optcomma CLOSEPAREN
    | expr DOT IF OPENPAREN exprseq optcomma CLOSEPAREN
    | expr DOT IF block optblock
    | IF OPENPAREN exprseq CLOSEPAREN block optblock
    ;

while   : WHILE OPENPAREN block optcomma blocklist CLOSEPAREN blocklist
        | WHILE blocklist1
        | expr DOT WHILE OPENPAREN block CLOSEPAREN
        ;

adverb  : %empty
        | DOT IDENTIFIER
        | DOT INTEGER
        | DOT OPENPAREN exprseq CLOSEPAREN
        ;

exprn   : expr
        | exprn SEMICOLON expr
        ;

exprseq : exprn optsemi
        ;

arrayelems  : %empty
            | arrayelems1 optcomma
            ;

arrayelems1 : exprseq
            | exprseq COLON exprseq
            | KEYWORD exprseq
            | arrayelems1 COMMA exprseq
            | arrayelems1 COMMA KEYWORD exprseq
            | arrayelems1 COMMA exprseq COLON exprseq
            ;

expr1   : literal
        | blockliteral
        | listcomp
        | IDENTIFIER
        | CURRYARGUMENT
        | msgsend
        | OPENPAREN exprseq CLOSEPAREN
        | TILDE IDENTIFIER
        | OPENSQUARE arrayelems CLOSESQUARE
        | OPENPAREN valrange2 CLOSEPAREN
        | OPENPAREN COLON valrange3 CLOSEPAREN
        | OPENPAREN dictslotlist CLOSEPAREN
        | expr1 OPENSQUARE arglist1 CLOSESQUARE
        | valrangex1
        | if
        | while
        ;

valrangex1  : expr1 OPENSQUARE arglist1 DOTDOT CLOSESQUARE
            | expr1 OPENSQUARE DOTDOT exprseq CLOSESQUARE
            | expr1 OPENSQUARE arglist1 DOTDOT exprseq CLOSESQUARE
            ;

valrangeassign  : expr1 OPENSQUARE arglist1 DOTDOT CLOSESQUARE ASSIGN expr
                | expr1 OPENSQUARE DOTDOT exprseq CLOSESQUARE ASSIGN expr
                | expr1 OPENSQUARE arglist1 DOTDOT exprseq CLOSESQUARE ASSIGN expr
                ;

// (start, step..size) --> SimpleNumber.series(start, step, last) -> start.series(step, last)
valrange2   : exprseq DOTDOT
            | DOTDOT exprseq
            | exprseq DOTDOT exprseq
            | exprseq COMMA exprseq DOTDOT
            | exprseq COMMA exprseq DOTDOT exprseq
            ;

valrange3   : exprseq DOTDOT
            | DOTDOT exprseq
            | exprseq DOTDOT exprseq
            | exprseq COMMA exprseq DOTDOT
            | exprseq COMMA exprseq DOTDOT exprseq
            ;

expr    : expr1
        | valrangeassign
        | CLASSNAME
        | expr binop2 adverb expr %prec BINOP
        | IDENTIFIER ASSIGN expr
        | TILDE IDENTIFIER ASSIGN expr
        | expr DOT IDENTIFIER ASSIGN expr
        | IDENTIFIER OPENPAREN arglist1 optkeyarglist CLOSEPAREN ASSIGN expr
        | HASH mavars ASSIGN expr
        | expr1 OPENSQUARE arglist1 CLOSESQUARE ASSIGN expr
        ;

block   : OPENCURLY argdecls funcvardecls methbody CLOSECURLY
        | BEGINCLOSEDFUNC argdecls funcvardecls funcbody CLOSECURLY
        ;

funcvardecls    : %empty
                | funcvardecls funcvardecl
                ;

funcvardecls1   : funcvardecl
                | funcvardecls1 funcvardecl
                ;

funcvardecl : VAR vardeflist SEMICOLON
            ;

argdecls    : %empty
            | ARG vardeflist SEMICOLON
            | ARG vardeflist0 ELLIPSES IDENTIFIER SEMICOLON
            | PIPE slotdeflist PIPE
            | PIPE slotdeflist0 ELLIPSES IDENTIFIER PIPE
            ;

constdeflist    : constdef
                | constdeflist optcomma constdef
                ;

constdef    : rspec IDENTIFIER ASSIGN literal
            ;

slotdeflist0    : %empty
                | slotdeflist
                ;

slotdeflist : %empty
            | slotdeflist optcomma slotdef
            ;

slotdef : IDENTIFIER
        | IDENTIFIER optequal literal
        | IDENTIFIER optequal OPENPAREN exprseq CLOSEPAREN
        ;

vardeflist0 : %empty
            | vardeflist
            ;

vardeflist  : vardef
            | vardeflist COMMA vardef
            ;

vardef  : IDENTIFIER
        | IDENTIFIER ASSIGN expr
        | IDENTIFIER OPENPAREN exprseq CLOSEPAREN
        ;

dictslotdef : exprseq COLON exprseq
            | KEYWORD exprseq
            ;

dictslotlist1   : dictslotdef
                | dictslotlist1 COMMA dictslotdef
                ;

dictslotlist    : %empty
                | dictslotlist1 optcomma
                ;

dictlit2: OPENPAREN litdictslotlist CLOSEPAREN
        ;

litdictslotdef  : listliteral COLON listliteral
                | KEYWORD listliteral
                ;

litdictslotlist1    : litdictslotdef
                    | litdictslotlist1 COMMON litdictslotdef
                    ;

litdictslotlist : %empty
                | litdictslotlist1 optcomma
                ;


listlit : HASH listlit2
        ;

// Same as listlit but without the hashes, for inner literal lists
listlit2: OPENSQUARE literallistc CLOSESQUARE
        | CLASSNAME OPENSQUARE literallistc CLOSESQUARE
        ;

literallistc    : %empty
                | literallist1 optcomma
                ;

literallist1    : listliteral
                | literallist1 COMMA listliteral
                ;

rwslotdeflist   : rwslotdef
                | rwslotdeflist COMMA rwslotdef
                ;

rwslotdef   : rwspec IDENTIFIER
            | rwspec IDENTIFIER ASSIGN literal
            ;

rwspec  : %empty
        | LESSTHAN
        | GREATERTHAN
        | READWRITEVAR
        ;

rspec   : %empty
        | LESSTHAN
        ;

arglist1    : exprseq
            | arglist1 COMMA exprseq
            ;

arglistv1   : ASTERISK exprseq
            | arglist1 COMMA ASTERISK exprseq
            ;

keyarglist1 : keyarg
            | keyarglist1 COMMA keyarg
            ;

keyarg  : KEYWORD exprseq
        ;

optkeyarglist   : optcomma
                | COMMA keyarglist1 optcomma
                ;

mavars  : mavarlist
        | mavarlist ELLIPSES IDENTIFIER
        ;

mavarlist   : IDENTIFIER
            | mavarlist COMMA IDENTIFIER
            ;

// WHILE and IF are not true reserved words in sclang, so we allow them to be used as identifiers for method names for
// compatibility with LSC.
name: IDENTIFIER
    | WHILE
    | IF
    ;

blockliteral:   block
            ;

listliteral : coreliteral
        | listlit2
        | dictlit2
        // NOTE: within lists symbols can be specified without \ or '' notation.
        | IDENTIFIER
        ;

literal : coreliteral
        | listlit
        ;

coreliteral : LITERAL
            | integer
            | float
            | stringlist
            | SYMBOL
            ;

integer : INTEGER
        | MINUS INTEGER %prec MINUS
        ;

floatr  : FLOAT
        | MINUS FLOAT %prec MINUS
        ;

float   : floatr
        | floatr PI
        | integer PI
        | PI
        | MINUS PI
        ;

binop   : BINOP
        | PLUS
        | MINUS
        | ASTERISK
        | LESSTHAN
        | GREATERTHAN
        | PIPE
        | READWRITEVAR
        ;

binop2  : binop
        | KEYWORD
        ;

string  : STRING
        ;

stringlist  : string
            | stringlist string
            ;
```