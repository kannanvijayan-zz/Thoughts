
# Binary AST - Motivations and Design - Part 1

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

As we work towards that, we'd like to share the reasoning and
motivations behind the project requirements and the design choices we've made
while hammering out the preliminary specification and building the first prototype. 


## The Problem

David' opening newsletter post covers the problem in detail:
[https://yoric.github.io/post/binary-ast-newsletter-1].

... but in short: the web uses a lot of Javascript, and every part of downloading,
parsing, compiling, optimizing and executing that Javascript is time-consuming, 
and substantial improvements to any of them will result in a much better Web
experience for everybody. 

With that in mind, our goal was to design a source format that allows for fast
parsing.

First, we should explain d why JS parsing is currently slow. The "What if we
just made the parser faster?" section in David's article details a number
of reasons, but there's a fundamental, unavoidable issue underneath all of them:

*Parsing is slow because the parser has to look at and process every byte
of the source it loads, regardless of whether that code is about to run
or not*.

What's interesting is that JS engines already realize this to some degree,
and try their best to avoid fully parsing as much code as they can get
away with.  This technique is called lazy parsing, and all major JS engines
employ it:  When code is first loaded, most functions are not fully parsed.
Instead, a faster "syntax-error-only" parse is run to check for errors.
Later on, if the function is called, it will be re-parsed with a "full"
parser to generate bytecode for execution.

This technique delays heavyweight parsing until after page-load,
on the hopes that the function is only called after the page done loading
(or better yet, never).  In total, this increases the time spent parsing the
function if it is ever called, but can be a worthwhile trade-off for
start-up time.

Unfortunately, we've basically reached the end of the performance curve for
optimizing the parsing of existing JS syntax.  In fact, recent syntax
additions to the language (e.g. destructuring syntax) has added more ambiguity
and parsing complexity, resulting in slower parsing times overall.

Several languages and source formats have tried to address this problem
in different ways, making different work-per-byte and load-time performance
tradeoffs when they're loading and running code:

### Java

The Java spec requires heavy verification of a loaded .class file.  The
classes, class structure, functions, function signatures, and foreign
interface use must be verified across the class file.  Methods must have
their bytecode verified for stack and type consistency, dependent
class files may also be loaded and recursively verified.

The per-byte work of loading a Java class file is very large due to
Java's verification requirements. As a direct result, of this, Java is
historically well known for having very slow application start-up
times.

### Javascript

With no type checking, JS requires much lighter verification of loaded
code than Java. Only syntax checking is needed, so even though JS ships
a much higher-level source format that's harder to parse than Java's class
files, per-byte work of loading source is much lower and conseqently
JS application load times are a fraction of Java application load times.

Even with this lighter burden of verification, the fact that every byte
of source has to be scanned is a high burden, and JS engines mitigate
it using the syntax-parsing technique I described earlier.

### Native Apps

Native apps are the obvious extreme when it comes to start-up performance.
Native binaries load and start executing almost instantly.

Not coincidentally, native executable formats specify almost no verification
of the executable data.  Load-time work includes parsing an executable
file's object tables and doing initial linking, but native program loaders
don't scan every byte of an executable doing some sort of byte-level
verification of the program.  If the program is malformed, that malformation
will (typically) present itself as a runtime crash if and when it is hit.

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

### Dart

Google's Dart language deserves a special mention here: Google's engineers
noted the hit to load-times incurred by heavy load-time verification, and
decided to make all syntax errors lazy.

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
parsing completely. 

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


(1) - https://lists.freebsd.org/pipermail/freebsd-current/2010-August/019310.html] 




