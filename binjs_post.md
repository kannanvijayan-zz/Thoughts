
# BinaryJS - Motivations and Design

The BinaryJS proposal has cleared Stage 1 of the TC39 standards committee.
The next steps of the project are to produce an implementation and demo
that demonstrates the claimed value.

Given this progress, it's a good time for clearly and publically identifying
the goals of the feature.  My intent in writing this post is to clarify the
motivations of the project and avoid any misunderstandings.

If you're going to be happy or angry about BinaryJS, I'd prefer if you
were happy or angry about the thing we were actually trying to accomplish,
rather than some other thing cooked up by the internet :)

## Introduction

The BinaryJS project was initiated between Mozilla and Facebook in response
to parse times becoming a bottleneck for page load.  Facebook in particular
had been feeling the pinch of parsing 7MB of JS to load their front page,
leading to parse times of close to a second even on moderately powerful
machines.

The goal of BinaryJS is to improve page load times for web pages which
ship large amounts of javascript.  This goal motivates the design of entire
proposal.

Javascript engines already realize that parse time is a pageload bottleneck,
and employ many tricks to reduce it.  Every major JS engine does lazy
parsing: for most code, a stripped-down syntax-error-only parse is done.
When a lazily-parsed function is executed, it is re-parsed with a full parser
to generate bytecode.  This exemplifies the sensitivity of load-time parsing:
browsers will even choose to parse a function twice, just in the hopes that
the load-time parsing can be reduced.

So we know there's a problem.  And the reality is that we've basically reached
the end of the performance curve for optimizing the parsing of existing
JS syntax.  The reasons why will be detailed below.

## Why is parsing slow?

If we want to speed up parsing, we had better understand why it's slow.
The surface level reasons are easy to identify:

1. Early syntax errors
    - Javscript spec requires that certain syntax errors be raised at load
      time.  This demands the parser parse every function in a loaded file,
      even those it's not about to execute, to check for syntax errors.
    - In fact, the Dart language recognized these as a source of performance
      bottlenecks, and made almost all syntax errors lazy as a result.

2. Scoping information
    - JS engines often need to know how a variable is used by an inner
      function to decide where to allocate the variable.  This requires
      the parser to scan inner functions before it can generate an outer
      function's bytecode.

These seem like two separate problems, but really they are the same
performance problem: they force the parser to scan code it's not about to
execute.  At the very least, a tokenizer will step through every character
in the source file about to be loaded.  Every nanosecond spent by the
parser on a character, is a millisecond per megabyte of source loaded.

This leads to the following realization: *to speed up the parse stage,
we need to figure out how we can get the parser to skip looking at
entire chunks of code*.

In addition to the above goal, we adopted the following constraints
as being appropriate for the js/web ecosystem:

1. Preserve the basic syntax-oriented nature of Javascript.
2. Keep semantics as close to identical as possible with plaintext JS.
3. Keep new syntax as small (when compressed) as existing minified JS.
4. Minimize the cost of parsing when parsing must occur.
5. Define a semantics-preserving map between text js and binary js.

## Syntax proposal

We propose a binary syntax designed around an encoding of an abstract
syntax tree (AST) for Javascript.

The following sections discusses each major design proposal and how
it relates to the goals described above.

## Design Feature 1: Binary Encoded AST

Javascript is fundamentally a source-based language, with its semantics
defined in terms of a grammar production for a source file.  Introducing
a new source format presents the problem of defining a semantics that maps
to it, as well as a way for developers to map their familiar plaintext
JS code to and from BinaryJS code.

BinaryJS solves that problem by encoding an AST that can be transformed
back to Javascript source code.  We can design an AST for Javascript that
is guaranteed to map to and from plaintext JS.  This allows us to simply
specify that the semantics of a well-formed BinaryJS file is simply the
regular semantics of the corresponding plaintext JS.

This reduces the spec burden considerably, as the BinaryJS spec can restrict
itself to identifying how its files are structured and how those files map
to plaintext JS.

We additionally encode the AST with a pre-order traversal, and prefix the
encoding of each subtree with an integer denoting the encoded-byte-length of
the subtree.  This allows a parser to skip past an entire AST subtree by
simply adjusting the read pointer ahead by the encoded-byte-length.

Key takeaways for a binary-encoded AST:
1. Easy to transform to and from plaintext JS.
2. Specify operational semantics in terms of existing spec.
3. Length-prefixing subtrees lets the parser skip past them at zero cost.

## Design Feature 2: Lazify all syntax errors

One of the reasons that plaintext JS parsers are forced to scan every byte of
loaded code is the JS spec requirement that mandates the raising of early
errors for malformed syntax.

There is no way to avoid scanning all the bytes of a loaded source file
without relaxing this requirement, and so we relax it.  In BinaryJS, a
malformed syntax error will be raised when the function containing it
is called, not when the code is loaded.  In effect, a syntax error
within a function causes that function to always throw the error when
it is called.

This choice introduces a small semantic divergence with plaintext JS. A
plaintext JS file with a syntax error will fail to load, while a malformed
BinaryJS file may load and execute, only throwing an error when the function
containing the syntax error is called.

This is considered an acceptable semantic divergence, because it only
relates to malformed files: all syntactically corrext plaintext Javascript
source translates cleanly into BinaryJS source with the same execution
semantics.

There is no translation from syntactically incorrect plaintext JS to BinaryJS.
This 

Key takeaways for lazy syntax errors:
1. All syntax errors in BinaryJS are treated as lazy.
2. The function directly containing the error will always throw when called.
3. Source without syntax errors behaves like corresponding plaintext JS.

## Design Feature 3: Predeclare scoping-related information

There's yet another reason that Javascript parsers are forced to scan inner
functions, and it has to do with variable scoping.  When variables are
declared in functions, the choice of where to store them often depends on
how they are used by inner functions.  If a variable is "captured by"
(referenced in) an inner function, it generally has to have a heap location
allocated for it.  Otherwise, it can be stored entirely on stack.

Figuring out which variables are captured by inner functions requires
scanning those inner functions.  To avoid this, BinaryJS "lifts" scoping
information from inner functions, out to the outer functions where variables
are declared.

For now, this scoping information is just a set of boolean flags on a
function node, identifying which variables are captured, as well as other
miscellaneous information like the presence of 'eval' within the function.

## Design Feature 4: Constant table for identifiers

BinaryJS lifts all identifiers in the source file into a constant table,
allowing the source to refer to identifiers using an index into the constant
table.  We impose the constraint that the names in the constant table are
unique.  This has several benefits.

First, it reduces the size of the source file significantly (even when
compressed).

Second, it speeds up handling of identifiers during parsing.  All parsers
need to map identifiers to internal VM strings.  In plaintext JS, the parser
has no idea how many unique names there exist in a source file, or where
they occur, so it must use a hashtable lookup using a string key to map
parse-time identifiers to VM strings.

The BinaryJS constant table turns all names in the source code into
integers.  This means lookup of a name is always a single array read -
no string hashing and no hashtable probing.  Furthermore, comparison
between names during parse becomes a blind integer comparison, with
no interning cost incurred.

## Design Feature 5: Single-pass bytecode generation

This is not so much an explicit feature as it is a consequence of the previous
features.

Current plaintext JS parsers are multi-pass.  They build an in-memory AST,
then walk that AST to generate bytecode.  Generating an in-memory AST is
required because the parser needs to collect book-keeping information (e.g.
the captured variable info mentioned above) from further down the source
code before it can compile the code it is currently parsing.

BinaryJS's format moves all of this book-keeping information to the locations
where the parser needs it at the time of code-generation.  This offers
implementors the opportunity to build a parser that generates bytecode
in a single pass, as it scans the source.

In BinaryJS, we have basically pre-generated the relevant AST with book-keeping
information and serialized it.  A BinaryJS parser needs only walk the
pre-built AST and generate bytecode for it.

## Summary

In all, the features described above are designed to work together to
reduce parse times as much as possible, while minimizing fundamental
changes to JS semantics.

The proposal enables skipping of uninteresting code subtrees, fast
traversal of the AST structure, fast code generation, and single-pass
codegeneration.

There is a key point I'd like to draw attention to here: *both plaintext
and binary JS source carries the same information, but formatted differently*.

Basically all we're doing is taking the information in an existing plaintext
JS file, restructuring it in a way that makes it very fast for a parser to
go through it, and then dumping that as a new file format.

From a developer perspective, this will be nothing more than a drop-in to
their toolchain which generates a `.binjs` file from their `.js` files.

From an implementor perspective, we have to write a new (relatively simple)
parser frontend.  Parsing a binary-encoded pre-order tree traversal is quite
a bit easier than parsing general-purpose text, so I don't see this as a
major burden.

From a user perspective, pages will magically load a whole lot faster.

## Prototype

DESCRIBE PROTOTYPE IMPLEMENTATION HERE.

## Some final thoughts

Javascript as a language needs the ability to scale from small pages to large
apps.  Top tier webpages today are shipping 7MB or more of uncompressed
Javascript, and this trend is not going to reverse.  There will be more JS
shipped to users tomorrow than there is today.

*The JS community cannot avoid the requirement to speed up parse times*.
Pages _will_ get larger, bigger apps _will_ ship on the web, and the
load-time issues we are seeing today will increase.  Bemoaning the state
of large amounts of shipping JS will not change that situation.  And the
situation as it exists is leading to poor user experiences, wasted time,
and wasted energy as phones and computers spend inordinate amounts of time
parsing plaintext JS on growing pages.

Lastly, I will quickly address the `WebAssembly` issue, because it's bound
to come up.  While WebAssembly is a fantastic feature for the web, and
will have a tremendous force and impact on what the web becomes, it is
not going to replace regular Javascript anytime in the forseeable future.

Regular JS use will continue to grow, and while that's happening, it's still
our job in the community to keep ensuring that our users aren't jerked around
by poor experiences, and that their experiences are improving.


