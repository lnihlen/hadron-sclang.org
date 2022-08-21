---
title: "Stack Frame"
---

{{< toc >}}

## The SuperCollider Frame Object

The [Frame](https://doc.sccode.org/Classes/Frame.html) object has no public member variables from the SuperCollider
language side. Inside the interpreter, Legacy SuperCollider (henceforth LSC) defines the `Frame` in
`lang/LangSource/PyrKernel.h` as:

{{< highlight cpp "lineNos=table,lineNoStart=78">}}
struct PyrFrame : public PyrObjectHdr {
    PyrSlot method;
    PyrSlot caller;
    PyrSlot context;
    PyrSlot homeContext;
    PyrSlot ip;
    PyrSlot vars[1];
};
{{< /highlight >}}

I haven't found any documentation about the intended uses of the members of `PyrFrame`, my reading of the code,
particularly around `executeMethodWithKeys` inside of `lang/LangSource/PyrMessage.cpp`, leads me to suppose the
following:

 * `method`: Contains a `Method` instance associated with the executable currently running. On frame creation, LSC sets
   `method` to the method about to be called and then sets a global variable method field to the same value. The
   interpreter resolves `thisMethod` to the same global variable method field, so the `method` field usually has the
   same semantics as `thisMethod`.
 * `caller`: A `Frame` from the calling code, restored as the active frame when returning.
 * `context`: A `Frame` defining the *next* outermost context for any nested functions. LSC makes this self-referential
   for top-level method code with no outer context.
 * `homeContext`: A `Frame` defining the *top* outermost context for any nested functions. For a top-level method,
   LSC makes this self-referential.
 * `ip`: An instruction pointer for continuing in this frame, should the code call another method or yield.
 * `vars`: Storage for every local variable in the frame. A size one array as the last element in a structure is a
   common idiom in LSC code that implies that `PyrFrame` instances will size this array to accommodate all the local
   variables stored in the frame.

SuperCollider supports [lexical closure](https://doc.sccode.org/Reference/Scope.html), meaning that in some instances,
`Frame` objects may outlive the code invocation they support. A contained
[Function](https://doc.sccode.org/Classes/Function.html) object keeps a reference to the outer context `Frame` in its
`context` member. This reference prevents the premature garbage collection of the `Frame` until the inner `Function` is
garbage collected.

Most methods are [closed](https://doc.sccode.org/Classes/Function.html#-isClosed), meaning they don't use lexical
closure and don't need the frame to outlive their invocation. Future optimization work could skip allocations of
separate frame objects for closed method chains.

Hadron reserves a CPU register for a *frame pointer* that points at the current `Frame` instance. Hadron maintains
function arguments and local variables here, saving any modifications to their location relative to the frame pointer.
On method invocations, Hadron writes the instruction pointer in the `Frame` instance for returning to the calling code
on method return. At runtime, Hadron uses the following frame pointer structure:

| frame pointer        | contents                         |
|----------------------|----------------------------------|
| `fp`                 | `Frame` schema header            |
| `fp` + 1             | `method`                         |
| `fp` + 2             | `caller`                         |
| `fp` + 3             | `context`                        |
| `fp` + 4             | `homeContext`                    |
| `fp` + 5             | `ip`                             |
| `fp` + 6             | Argument 0 (this) / Return Value |
|  ...                 |  ...                             |
| `fp` + 6 + n - 1     | Argument n - 1                   |
| `fp` + 6 + n         | Local Variable 0                 |
|  ...                 |  ...                             |
| `fp` + 6 + n + m - 1 | Local Variable m - 1             |
| `fp` + 6 + n + m     | < register spill area >          |

## The Stack Pointer

Hadron also reserves a CPU register for a *stack pointer*, which points at an incomplete `Frame` instance used for
constructing new messages. The calling code copies the in-order and keyword-based arguments into the new `Frame`.

SuperCollider is a dynamic programming language, so we often don't know the message's intended recipient at compile
time. That means we don't know how many arguments the callee code expects or the names of those arguments. SuperCollider
copies default values into the frame for any missing in-order arguments and then overwrites any named arguments with the
provided key/value pairs. See [the spec]({{< ref "/spec/4_execution_model" >}}) for details about parameter assignments
to arguments.

## Frame Size Computation

Frames contain the interpreter state variables, arguments, local variables including any inlined frames, and register
spill space. Hadron maintains a per-selector maximum size tracking the upper bound of frame size across all
methods with the same name. The upper bound is computed as the maximum number of arguments for that selector, plus the
maximum number of local variables for that message, plus the fixed constant maximum number of spill registers.

Computing the maximum number of arguments ensures that when we set up the new frame for the message we won't overwrite
argument defaults with keyword/value argument pairs. We use the total maximum size to determine if there is sufficient
room to support calling a message without allocating additional memory.

We also compute the *exact* frame size needed for each `Method`, allowing for more accurate size computation when the
method calls are unambiguous. This figure is also useful when allocating separate frames for open functions.

TODO: variadic functions?

## Stacklets

Large-size allocations, perhaps with some hysteresis on garbage collection when calling across boundaries. Normally the
frame pointer points ahead of the stack in the same large allocation.

## Dynamic Dispatch

Dynamic dispatch refers to all method calls where the target object is unknown at compile time. We include special cases
for variadic arguments and non-closed functions.

To avoid recopying arguments we lay out a `Frame` object instance with the arguments in their expected position, but
re-use some of the other fields before the arguments start to contain all the information known about the message at
compile time. At message send time the Frame object is laid out as follows:

| stack pointer     | contents                            | Notes                                                    |
|-------------------|-------------------------------------|----------------------------------------------------------|
| `sp`              | `HadronPrototypeFrame`  header      | Preserved to support object identification               |
| `sp` + 1          | Selector symbol                     | Dynamic dispatch will replace with the `Method` instance |
| `sp` + 2          | `caller`                            | Caller saves a copy of their frame pointer here          |
| `sp` + 3          | Number of In-Order Arguments (j)    |                                                          |
| `sp` + 4          | Number of Keyword / Value Pairs (k) |                                                          |
| `sp` + 5          | `ip`                                | Caller saves their return instruction address here       |
| `sp` + 6          | Argument 0 (callee this)            | In-Order arguments start here with `this`                |
|  ...              |  ...                                |                                                          |
| `sp` + 6 + j - 1  | Argument j - 1                      |                                                          |
| ...               | ...                                 | `nil`                                                    |
| `sp` + 6 + ArgMax | Keyword / Value Pair 0              |                                                          |
|  ...              |  ...                                |                                                          |


### The Dynamic Dispatch Algorithm

Caller Side:

On entry, `sp` is pointing at the first free spot on stack after the current `Frame` at `fp`.

Check for available stack space in `sp` against the maximum frame size for the given selector.
If there's not enough room:
  Interrupt for allocate stack frame, will redirect `sp` to new frame

Build `HadronPrototypeFrame` object according to above
  Save selector symbol into `sp->selector` - redundant because we're branching to that code?
  Save current `fp` into `sp->caller`
  Save instruction pointer for return address label into `sp->ip`
  Save in-order and key/value arguments

Branch to Selector-specific dispatch code.

Selector-specific dispatch code:

In its simplest form, this is a series of `if` statements comparing against the class name hash in the message target.
If it finds a match on the class name hash, it executes a preamble that completes frame initialization and branches to the compiled method code.

Lastly, sets `fp` <- `sp`.

### VarArgs

Variable argument messages expect the last argument to be an `Array` containing any additional arguments specified.
The dispatch code creates a new `Array` object from those arguments during stack setup. We then overwrite the final
argument in the stack with a pointer to that new `Array`.
