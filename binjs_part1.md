
# Binary AST - Motivations and Design - Part 1

The Binary AST proposal recently cleared Stage 1 of the TC39 standards committee.
Binary AST is a proposal for specifying a binary-encoded syntax for
JS with the intent of allowing browsers and other JS-executing environments
to parse and load code much faster.

David Teller posted a Binary AST newsletter introducing the broad proposal.

While the final byte-level format has yet to be fully nailed down, we have
arrived at a prototype design, and a set of design requirements for a final
format that we are confident will deliver the performance benefits promised by
the prototype.  The prototype shows potential performance gains of 80% in
parsing code as compared to minified JS.

For the next stage of the standards process, we are working towards a
fleshed out spec as well as a production implementation and supporting
website (Facebook home page) that demonstrates the claimed performance gains.

This is one of a series of articles detailing the motivation behind
the technical design choices in the Binary AST proposal.  This article should
be understandable to moderately experienced web developers with passing
familiarity of some compiler concepts (e.g. parse trees).

## The Problem

David' opening newsletter post covers the problem in detail:
[https://yoric.github.io/post/binary-ast-newsletter-1].

Our goal is design a source format that allows for fast parsing.  To do so
effectively, we must first understand why JS parsing is currently slow.
Specifically, the "What if we just made the parser faster?" section in
David's article details a number of reasons why parsing JS is time consuming.

While there are a lot of reasons listed there, there's actually a much
simpler and more fundamental way to restate the issue behind all of them:

*Parsing is slow because the parser has to look at and process every byte
of the source it loads, regardless of whether that code is about to run
or not*.

Turning that around, we can restate it as: *The fastest way to parse
something is not to parse it at all*.

What's interesting is that JS engines already realize this to some degree,
and try their best to avoid fully parsing as much code as they can get
away with.  This technique is called lazy parsing, and all major JS engines
employ it:  When code is first loaded, most functions are not fully parsed.
Instead, a faster "syntax-error-only" parse is run to check for errors.
Later on, if the function is called, it will be re-parsed with a "full"
parser to generate bytecode for execution.

This technique is used to delay heavyweight parsing until after page-load,
on the hopes that the function is only called after the page is loaded
(or never at all).  In total, this increases the time spent parsing the
function if it is ever called, but can be a worthwhile trade-off for
start-up time.

Unfortunately, we've basically reached the end of the performance curve for
optimizing the parsing of existing JS syntax.  In fact, recent syntax
additions to the language (e.g. destructuring syntax) has added more ambiguity
and parsing complexity.

Let's take a look at some existing source formats and their load-time
performance as compared to the work-per-byte they do when loading code.

### Java

Java is historically well known for having very slow application start-up
times.

The Java spec requires heavy verification of a loaded .class file.  The
classes, class structure, functions, function signatures, and foreign
interface use must be verified across the class file.  Methods must have
their bytecode verified for stack and type consistency.  Dependent
class files may also be loaded and recursively verified.

The per-byte work of loading a Java class file is very large due to
Java's verification requirements.

### Javascript

JS requires much lighter verification of a loaded code than Java.  JS has
no type checking, so only syntax checking is needed.

This means that even though JS ships a much higher-level source
format that's harder to parse than Java's class files, JS's application
load times are a fraction of Java application load times.

The per-byte work of loading source is much lower for JS than for Java.

Even with this lighter burden of verification, the fact that every byte
of source has to be scanned is a high burden, and JS engines mitigate\
it using the syntax-parsing technique I described earlier.

### Dart

Google's Dart language deserves a bit of special mention here.  It observed
the hit to load-times incurred by heavy load-time verification requirements,
and decided to make all syntax errors lazy.

The lazification of syntax errors allowed Dart to greatly reduce the cost
of load-time parsing.  Code only needs to be scanned for a handful
of tokens such as brackets (for checking for consistent nesting) and strings
and comments (because the parser needs to ignore brackets within strings and
comments).  These checks are simple enough to be accelerated by SIMD
data-parallel instructions.

Dart's approach was to design their language syntax and semantics to
allow the lazy "syntax-error-only" parser to run as fast as possible.

The per-byte work for loading Dart is much lower than for JS.

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

The per-byte work for loading native code is almost zero.

## Analysis

The above examples justify the reduced problem statement from earlier.
If we want to fix the parsing issue, we need to:

1. Reduce the amount of per-byte work done for loading code that's not about to run.
2. Preferably reduce it to zero.

Our goal with Binary AST, then, is aggressive: design a source format
that allows a parser to achieve load-time performance comparable to
native programs, by enabling the parser to skip looking at code
it's not about to execute.

The challenge is to do this while respecting and adhereing to the
general ethos of the JS language, being easily accessible to
web developers, and having a good compatibility with the existing
JS ecosystem.

In subsequent articles, we'll address individual features in
the Binary AST proposal, and how those features contribute to
these goals.
