---
title: Modular Class Library Proposal
linkTitle: Modular Class Library Proposal
date: "2021-12-21"
author: Luke Nihlen
type: "posts"
---

# tl;dr

Here I will propose a reorganization of the SuperCollider class library, the large body of code (roughly 74k lines) that
is distributed with every build of SuperCollider and managed in the primary SuperCollider repository under the
`SCClassLibrary` directory. I propose to break the existing SuperCollider class library into several interdependent
modules, each organized into one of three layers.

![Diagram of the proposed three layers. In the center is the Core, surrounded by the middle layer, with the exterior
layer at the outside](/images/class-library-layers.png)

The *Core* consists only of classes that define fundamental types, like `Integer` or `Symbol`, and services essential to
the interpreter like `Interpreter`, `Class`, or `Method`. SuperCollider developers will maintain the Core in its current
location in the SuperCollider repository. Developers can hold the test and documentation coverage to a higher standard
with a smaller Core.

We'll use the RFC process to identify *owners* for each module outside the Core. Owners will take responsibility for the
maintenance and development of their modules, meaning they can approve changes to the module without the involvement of
the SuperCollider development team. We will move all modules to Quarks, and owned modules will still be a part of the
SuperCollider distribution. We will remove modules without owners from the SuperCollider distribution, although they
will still be available for download as Quarks.

Each module owner will develop an automated *validation process* for that module. The process can include `UnitTest` and
integration test scripts maintained in the individual modules. Any developer making changes to the language or the Core
library will need to *validate* the proposed change against every middle layer module using the automated validation
process.

Maintaining small libraries is much easier than larger ones. Developers can make *faster* decisions across the project,
with responsibility for each piece resting clearly with a designated owner. Developers can make *better* decisions
because the validation signals are a credible source of confidence for developers (and reviewers) to gauge the impact
and potential harm of any change they would like to make.

# Problem Space

## Unclear Authority

A few years ago, James McCartney, the original author of SuperCollider, resigned from the project, followed later
by the resignation of the most senior remaining developer. It's now less clear who the deciding authority is on any
nontrivial technical decision. Furthermore, as the Class Library has grown in size, fewer people have the requisite
knowledge to make informed decisions about cross-cutting changes or complex areas of the code base like `Server` or the
contents of `JITLib`.

There is an [RFC process](https://github.com/supercollider/rfcs#what-does-an-rfc-do) for formal proposals for changes to
SuperCollider. Democratic decision-making processes trade speed for fairness and inclusiveness, and sometimes reaching a
consensus can take a long time. For high-impact changes, the RFC process is worth the time and trouble. However, there's
a middle ground right now for issues that are larger than developers feel comfortable deciding on in a PR and smaller
than might feel worth the time and energy investment required of an RFC.

## Library Validation

Our current test coverage is low, making it difficult to understand the impact of a language change on the library. This
ambiguity increases the burden on the code maintainers and tends to make reviewers conservative in reviewing changes
because of concerns about unintended consequences. This uncertainty adds drag to the project. It also has a chilling
effect on new volunteers because contributors get very little automated feedback about the correctness of their work,
meaning they are slow to build confidence.

## Primitive Bloat

SuperCollider has no runtime [Foreign Functions Interface](https://en.wikipedia.org/wiki/Foreign_function_interface). We
support foreign functions at compile time via *primitives* in the interpreter C++ code. The policy is to add new FFI
libraries behind CMake option flags, usually keeping the flag off in the distribution until the library code has reached
a certain level of maturity. The compile-time options space for the interpreter is large, and there aren't conventions
about how to establish which configurations are available to the interpreter at runtime.

We've distributed the library code supporting all primitives along with the class library. Consider a minimal
SuperCollider build configuration with all UI features disabled. In this example, the view library code is still defined
but attempting to use view code in that SuperCollider build results in unpredictable behavior. The expansion of the
interpreter into a matrix of possible build configurations dilutes our limited developer support and testing coverage
even further.

We also distribute library code specific to individual operating systems on all platforms (see [Linux MIDI
code](https://doc.sccode.org/Classes/MIDIOut.html)), but changing this will very likely cause breakage in client code.

# Technical Plan

A detailed design document is out of scope for this proposal, but I would write as a separate RFC before beginning the
work. Overall I envision the Core classes remaining in the SuperCollider repository. We would move the middle layer
modules into Quarks but commit them as git submodules. This change should minimally impact the build and deploy process,
as the files will all be in place at compile time.

## Organizational Principles

I include here some principles that can help drive decisions about the organization of individual files and classes.

### Core Supports Validation

The Core should be minimally small, including only enough code to support `UnitTest` and script validation. Organizing
the Core around support for testing forces the testing code to have minimal dependencies and provides a good rubric
for determining if a class needs to be in Core or not. This organizing principle also creates opportunities down the
road to better integrate testing into the SuperCollider language. Lastly, it means the Core can *validate itself* with
no external dependencies.

### Primitives Guide Organization

We should organize build configurations such as `SC_ABLETON_LINK` or `SC_QT` into individual modules. We could also
enable a change in the install step for SuperCollider where it will not copy disabled modules over to the installed
class library, leaving the classes out of the final build. If SuperCollider ever gets a runtime FFI, this would be the
first step towards supporting a modular binary class library.

### Potential Owners Should Have a Say

Developers should accommodate volunteers willing to take ownership of a module for their ideas about what parts belong
in that module and what don't. It is best to divide control and responsibility in equal measure.

### SuperCollider Developer Module Ownership

Given the size of the code base, it's likely that there will be a lot of modules at first that are essential but
unowned. The *de facto* owners for these modules are the SuperCollider developers. Modules in this state are
no worse off than today, other than the work required for reorganization. However, there are now numerous opportunities
for new developers to volunteer, and they are starting with a clear charter and well-defined scope for their work.

### Don't Worry About Dependencies

As the class library is a monolith, there are a lot of complex dependencies between classes. SuperCollider is a dynamic
programming language, so there aren't good ways to enumerate dependencies between potential modules. However, since we
intend to distribute the entire class library monolithically, everything a class *needs* should be there at runtime.

The exception is for unowned classes intended for removal from the SuperCollider distribution. With improvements to
library validation developers can gain the confidence needed to remove unowned modules from distribution.

### Rely on Class Extensions

`Object` is a clear candidate for the Core but has around 270 methods. It is common in SmallTalk-like languages to
see method bloat in the root object. Anything in `Object` not needed for validation should be moved to modules and
added as class extensions. For example, things like `isUGen` and `numChannels` could likely go to an *audio* or *synth*
module, whereas `awake`, `beats`, and `clock` could all move to a *timing* module.

# Future Work

Improvements to the class library organization benefit both SuperCollider and Hadron development. Here are some
ideas for future projects in Hadron that carry this organization work further.

 * *Runtime Foreign Functions Interface* - This could radically expand the capabilities of a SuperCollider programmer to
   interact with all the C and C++-based libraries forming the bulk of desktop application development tools. FFI also
   adds significant complexity to the code, previously hidden in the interpreter.
 * *Module-Specific Recompilation* - Hadron is likely to be a slower compiler than SCLang, so why not only compile those
   files that have changed?
 * *Conditional Compilation, Optional Dependencies* - The interpreter should know what modules are available and which
   are disabled.
 * *Namespaces* - A controversial topic in the developer community. Currently, all class names are global and
   SuperCollider provides no way to detect or resolve conflicts between Quarks or library code.
