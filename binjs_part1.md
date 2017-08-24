
# Binary AST - Motivations and Design Decisions - Part 1

"The key to making programs fast is to make them do practically nothing."
                      - Mike Haertel, creator of GNU Grep. (1)

Binary AST - "Binary Abstract Syntax Tree" - is Mozilla's proposal for specifying 
a binary-encoded syntax for JS with the intent of allowing browsers and other
JS-executing environments to parse and load code as much as of 80% faster than
standard minified JS.

It has recently cleared Stage 1 of the TC39 standards process, and while the
final byte-level format has yet to be fully nailed down, we are confident
that the final product will deliver on the impressive performance improvements
promised by the prototype. 

As we work towards that, we'd like to share our reasoning and motivations 
behind the project requirements and the design choices we've made while
hammering out the preliminary specification and building the first prototype. 


## The Problem

David' opening newsletter post covers the problem in detail:
[https://yoric.github.io/post/binary-ast-newsletter-1].

... but in short: the web uses a lot of Javascript, and every part of downloading,
parsing, compiling, optimizing and executing that Javascript is time-consuming. 
A substantial improvement to any of those steps will result in a much better Web
experience for everybody. While Mozilla has made remarkable progress in the last
five years on Javascript compilation, optimization and execution, we haven't seen
the same progress on the parsing front.

With that in mind, our initial goal was to design a source format that allows for 
extremely fast parsing.

First, we should explain why Javascript parsing is slow. The "What if we just made
the parser faster?" section in David's article details a number of reasons, but
there's a fundamental, inescapable issue that underpins all of them:

*Parsing is slow because the parser has to examine, understand, and make decisions
about every byte of the source it loads, regardless of whether that code is about
to run or not.*

All modern Javascript engines already understand this, of course, and for the sake
of speed will try their best to avoid fully parsing - or even fully understanding -
as much code as they can get away with.  This technique is called "lazy parsing";
when code is first loaded, rather than fully parsing the text of the code a much
faster "syntax-error-only" parser is run to check for errors. Then later, if the
function is called, it will be re-examined with the "full" parser that generates 
the bytecode to be executed.

Lazy parsing delays the most expensive, heavyweight parts of the parsing process
until the code's execution path confirms they're necessary, implicitly making a bet
that the function is only called after the page is done loading (or, for the best
results possible, never called at all). Perhaps obviously, two-pass techniques like
this will increase the total time spent parsing the function if it is called, but
this can still be a worthwhile trade-off for startup time and user-perceived
performance.

Unfortunately, lazy parsing and minification have basically brought us the end
of the performance curve for optimizing the parsing of existing JS syntax. And,
worse, recent additions to the language (e.g. destructuring syntax) have added
more ambiguity and hence parsing complexity to the process, resulting in slower
parsing times overall.

This is not a new problem, and several languages and source formats have tried to
address it in different ways, making different work-per-byte, load-time
and run-time performance tradeoffs whenever code is executed. Some examples of
those tradeoffs include:

### Java

The original Java spec required heavy verification of a loaded .class file.
The classes, class structure, functions, function signatures, and foreign
interface use must be verified across the class file.  Methods must have
their bytecode verified for stack and type consistency, dependent
class files may also be loaded and recursively verified.

Java's issues are mitigated by its binary format that allows parsers to process
the source much faster than plaintext source, but even so early Java's verification 
requirements were still too heavy (in particular the stack verification for methods),
and later versions of Java updated those requirements to allow for a much lighterweight
verification regime. 

This is an extremely cautious approach; the benefit is a very high degree
of confidence in the consistency and integrity of your execution environment,
but it comes at the cost of front-loading a lot of heavy work before you can run
a single instruction. 

### Javascript

With no type checking, Javascript parsers have a much lighter verification burden
than Java. Only syntax checking is needed, so even though Javascript ships as much
higher-level and harder-to-parse source files, compared to Java's class files, the per-byte
workload is much lower.

The tradeoff is that Javascript's text syntax forces parsers to tokenize the source
(split it up into atomic "words" before further decisions can take place) and then
re-construct the logical structure of the code. This is much more time consuming
than parsing binary structures.

The result is that even with the lighter burden of verification, parsing is enough
of a bottleneck for Javascript load times that all engines use the syntax-error-only
parsing technique to optimize it on first pass, and incurring additional costs later
in the execution process.

### Native Apps

Native apps are the obvious extreme when it comes to start-up performance.
Native binaries load and start executing almost instantly, and not coincidentally,
native executable formats specify almost no verification of the executable data.

In this context load-time work involves parsing an executable file's object
tables and doing initial linking, but (outside of cryptographic verification)
native program loaders will typically avoid the high cost of doing byte-level
verification of executables simply by not doing any verification at all. 
Consequently, if a program is malformed the best-case outcome is that it
will only crash rather than doing something insane or malicious.

It's important to note that native code formats (e.g. the ELF object file
format) treat their input as random access.  Aside from some key tables
describing the structure of the source, a native-code loader generally
never looks at a part of the source until it needs to execute it. 

The benefits here are obvious: this approach lets native programs load
and begin execution instantly and per-byte work for loading native code
is almost zero, because a native-code loader can ignore all loaded code
completely until it is run. Even though the native code format is typically
much less efficient than a source representation; for example, a call `f(1,2)` 
in source code takes 6 bytes to represent, but is much larger in machine code.

Of course, the fact that native code is interpreted directly by the CPU doesn't hurt. 

### Dart

Google's Dart language deserves a special mention here: Google's engineers
noted the hit to load-times incurred by heavy load-time verification, and
decided to make all syntax errors lazy as well.

The lazification of syntax errors allowed Dart to greatly reduce the cost
of load-time parsing.  Code only needs to be scanned for a handful
of tokens such as brackets (for checking for consistent nesting) and strings
and comments (because the parser needs to ignore brackets within strings and
comments).  These checks are simple enough to be accelerated by SIMD
data-parallel instructions.

Dart's approach was to design their language syntax and semantics to
allow the lazy "syntax-error-only" parser to run as fast as possible, and
the upshot of this is that per-byte work for loading Dart is much lower than for JS.

## Analysis

Reflecting on above examples drove us to reconsider our problem statement: the
best way for us to "fix" the parsing issue is to eliminate the need for
parsing code entirely, unless that code is about to run. 

Our goal with Binary AST, then, is aggressive: design a Web-compatible source
format that allows a parser to achieve load-time performance comparable to
native programs by enabling the parser to skip looking at code
it's not about to execute.

The challenge is to do this while respecting and adhereing to the
general ethos of the Javascript language, being easily accessible to
web developers, and having a good compatibility with the existing
Javascript ecosystem.

This set of constraints has led to the design decisions in Binary AST.
With the goal of eliminating unnecessary parsing, we identified and
addressed the aspects of plaintext Javascript that prevent that goal.

The design we arrived at is not so much a "new syntax" as it is
a pre-processed re-packaging of the information in a Javascript source file.
The Binary AST format can be converted to and from plaintext Javascript,
including no information that plaintext Javascript doesn't, and we intend
to ship bidirectional tooling alongside the final product to make
taking advantage of this format as easy as possible.

In subsequent articles, we'll address individual features in
the Binary AST proposal, and how those features contribute to
the goals we have set out above.


(1) - https://lists.freebsd.org/pipermail/freebsd-current/2010-August/019310.html] 
