
# BinaryJS - Motivations and Design - Part 1

The BinaryJS proposal recently cleared Stage 1 of the TC39 standards committee.
BinaryJS is a proposal for specifying a binary-encoded syntax for
JS with the intent of allowing browsers and other JS-executing environments
to parse and load code much faster.

While the final byte-level format has yet to be fully nailed down, we have
arrived at a prototype design, and a set of design requirements for a final
format that we are confident will deliver the performance benefits promised by
the prototype.  The prototype shows potential performance gains of 80% in
parsing code as compared to minified JS.

For the next stage of the standards process, we are working towards a
production implementation along with a supporting website (Facebook home page)
that demonstrates the claimed performance gains.

This is one of a series of articles detailing the motivation behind
the technical design choices in the BinaryJS proposal.  This article should
be understandable to moderately experienced web developers with passing
familiarity of some compiler internal concepts (parse trees, etc.).

## The Problem

The BinaryJS project was initiated between Mozilla and Facebook in response
to parse times becoming a bottleneck for page load.  Facebook in particular
had been feeling the pinch of parsing 7MB of JS to load their front page,
leading to parse times of hundreds of millisends, even on moderately powerful
machines.

For web pages that ship a lot of JS, or for large applications deployed
on the web, parse times are becoming a direct UX bottleneck, on top of
the power resources used (especially on mobile devices).

Note that this is a problem of-its-time.  We are running into this problem
now because the development of the JS ecosystem has reached a point where
codebase sizes are making parse times a more noticeable and painful
bottleneck.  A pragmatic need for faster parsing has arised now that
top-site JS payloads are starting to approach 10MB.

The goal of BinaryJS is to improve page load times for web pages which
ship large amounts of javascript.  In doing so, we hope to improve load
times for all JS-based application platforms, such as Electron and friends.

JS engines already realize that parse time is a pageload bottleneck, and
employ many tricks to reduce it.  Every major JS engine does "lazy" parsing:
for most code, a stripped-down syntax-error-only parse is done.  Bytecode
is not generated, and most parse information is thrown away.  This is
done because a syntax-error-only parse is much faster than a "full" parse
for execution.  If a function is called only late in the program, then
doing a syntax-error-only parse early can save a lot of page-load time.
When the function is eventually called, it needs to be re-parsed using
the full parser.  This exemplifies the sensitivity of load-time parsing:
browsers will even choose to parse a function twice, just in the hopes
that the load-time can be reduced.

Unfortunately, we've basically reached the end of the performance curve for
optimizing the parsing of existing JS syntax.  In fact, recent syntax
additions to the language (e.g. destructuring syntax) has added more ambiguity
and parsing complexity.

The only way to improve the parse-time of JS is to change some aspect of
the syntax to allow the parser to load code faster.  Before we can think
of how to change the syntax, however, we need to clearly understand why
parsing is slow, so our new syntax actually allows for the performance
gains we seek.

## Why is parsing slow?

I'll start with the conclusion we arrived at and then work on justifying it.

*Parsing is slow because parsers have to look at and process every byte
of the code they load, regardless of whether they are about to execute
it or not*

As application sizes grow, simply having to look at code before executing
it becomes a problem.  Consider a 10MB program.  If every character of that
10MB program needs to be processed in some way, then every nanosecond spent
on a character of source translates to 10ms of load-time delay.

This property is independent of whether the source format is binary or
text based.  To put it somewhat obnoxiously: *the fastest way to parse
something is not to parse it at all*.

Let's take a look at some examples in practice.

### Java

The Java spec requires heavy verification of a loaded .class file.  The
classes, class structure, functions, function signatures, and foreign
interface use must be verified across the class file.  Methods must have
their bytecode verified for stack and type consistency.  Dependent
class files may also be loaded and recursively verified as part of the
verification of an "origin" class file.

This leads to Java programs being extremely slow to load, as initial code
execution is impeded by constant verification of newly loaded code.  However,
once a Java application has reached its steady state, it runs very fast.

Java uses a binary bytecode format, but the heavyweight load-time requirements
make it extremely slow to start.  Historically, even small Java programs have
been known to take seconds to start up.

### Javascript

Javascript requires much lighter verification of a loaded code than Java.
Any syntactic errors in a file must be raised at load time, but all other
errors are raised at runtime.

No dependencies need to be pulled in, and no bytecode needs to be abstractly
interpreted.  Types, interfaces, method calls, and field accesses do not need
to be checked for consistency.

This means that even though Javascript ships a much higher-level source
format that's harder to parse than Java's class files, Javascript's application
load times are a fraction of Java application load times.

Even with this lighter burden of verification, Javascript still has to
check every byte of source code it loads.  To mitigage the impact of that,
JS engines split parsing into two modes: a lightweight syntax-error-only
parse, and a heavyweight parse-for-execution.

### Dart

Dart deserves a bit of special mention here.  It observed the hit to
load-times incurred by heavy load-time verification requirements, and decided
to make all syntax errors lazy.  However, Dart stuck with a text format for
its source.

The lazification of syntax errors allowed Dart to greatly reduce the cost
of syntax-error-only parsing.  Code only needs to be scanned for a handful
of tokens such as brackets (for checking for consistent nesting) and strings
and comments (because the parser needs to ignore brackets within strings and
comments).  These checks are simple enough to be accelerated by SIMD
data-parallel instructions.

All that said, Dart still requires that all source code be scanned by a
simple tokenizer.

### Native Apps

Native apps are the pracital extreme when it comes to start-up performance.
Native binaries load and start executing almost instantly.

Not coincidentally, native executable formats specify almost no verification
of the executable data.  Load-time work includes parsing an executable
file's object tables and doing initial linking, but native program loaders
don't scan every byte of an executable doing some sort of byte-level
verification of the program.  If the program is malformed, then the
malformation will typically exhibit itself as a runtime crash if and when
it is hit.

This lets native programs load instantly, even though the native code format
is typically much less efficient than a source representation (for example,
a call `f(1,2)` in source code takes 6 bytes to represent, but is larger
in machine code.  Of course, the fact that native code is directly decoded
by the CPU hardware doesn't hurt.

It's important to note that native code formats (e.g. the ELF object file
format) treat their input as random access.  Aside from some key tables
describing the structure of the source, a native-code loader generally
never looks at a part of the source until it needs to execute it.

## Analysis

There are multiple pieces of evidence pointing at the conclusion that if
we want to speed up load-time parsing, we need to choose some format and
semantics that allows the parser to ignore large chunks of loaded code
until they are executed.

The proposed performance theory "makes sense" in a trivial way (more code +
having to look at every byte of code + more work per byte = longer load-times).

The usage of syntax-error-only parsing by all major JS engines points to
an existing awareness of the load-time parsing problem.

The load-time performance properties of existing source formats clearly reveal
a relationship between load-time verification requirements and poor performance
outcomes.

The design question then becomes: how do we build a JS source format that
lets the parser achieve the performance improvements it wants, but is still
compatible with the existing JS ecosystem, convenient for developers to use,
and fits the values of the web and JS in general.

Our answer to that question is the set of features in the Binary JS proposal.
It proposes a new syntax that is translate-eable to and from plaintext JS,
and a set of core properties of the syntax needed to enable fast parsing.

In subsequent articles, we'll go into each of the proposed Binary JS
implementation features and discuss how it relates to the performance
goals we seek.

Keep reading.
