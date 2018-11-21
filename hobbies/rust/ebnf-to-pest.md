# Implementing an EBNF grammar in [pest](https://crates.io/crates/pest)

A recent project has led me to have a go at [writing an XML parser](https://github.com/compenguy/xml-grimoire).

I thought I'd document my experiences using [pest](https://crates.io/crates/pest) to implement a lexer using the [EBNF-esque](https://www.w3.org/TR/REC-xml/#sec-notation) [formal grammar](https://www.w3.org/TR/REC-xml/).

## Caveats

I have only passing experience with parsing and grammars.

## Getting started

First things first - adding the necessary dependencies to my [Cargo.toml](https://github.com/compenguy/xml-grimoire/blob/da4bff2d56df365f44570a54e4da6fba644d458c/Cargo.toml):
```
[dependencies]
pest = "2.0"
pest_derive = "2.0"
```

And setting up [my code to pull in a pest file](https://github.com/compenguy/xml-grimoire/blob/da4bff2d56df365f44570a54e4da6fba644d458c/src/lib.rs):
```
extern crate pest;
#[macro_use]
extern crate pest_derive;

#[derive(Parser)]
#[grammar = "xml1_0.pest"]
pub struct XmlParser1_0;
```

Now we're ready to dig into the real fun - writing the parsing expression grammar.

## XmlParser1_0

I named my parser type that because I'm implementing the [current revision of the xml 1.0 spec](https://www.w3.org/TR/2008/REC-xml-20081126/).  There's a 1.1 spec, but it's apparently mostly uninteresting to anyone able to use the latest revision of the 1.0 spec.  The [xml 1.1 specification indicates](https://www.w3.org/TR/2006/REC-xml11-20060816/#sec-conformance) that 1.0 should be preferred unless there's a specific requirement for 1.1 features, so 1.0 seems like a great foundation to build from.

Pest allows comments in its grammar files by starting the line with `//`.  I use this extensively to document the relevant sections of the xml spec for each [production](https://en.wikipedia.org/wiki/Production_(computer_science)).

### The Basics - `document`
Let's take a look at the first one, [xml 1.0 production 1](https://www.w3.org/TR/2008/REC-xml-20081126/#NT-document):
```
// # Document
// [1](https://www.w3.org/TR/2008/REC-xml-20081126/#NT-document)
// `document   ::=   prolog element Misc*`

document = { prolog ~ element ~ Misc* }
```

The first production (the spec gives each production a unique number) says a `document` consists of a `prolog`, followed by an `element`, and then zero or more `Misc`s.

The pest syntax for a production looks like `<name> = { <rules> }`.  To say "`A` then `B`", you write that "`A ~ B`".

Consequently, `document ::= prolog element Misc*` would be written in pest as `document = { prolog ~ element ~ Misc* }`.

### Characters, Alternation, and Ranges

The next production is used in defining generally acceptable character ranges.  It introduces a few new basic syntactic concepts:

```
// # Character Range
// [2](https://www.w3.org/TR/2008/REC-xml-20081126/#NT-Char)
// `Char   ::=   #x9 | #xA | #xD | [#x20-#xD7FF] | [#xE000-#xFFFD] | [#x10000-#x10FFFF]`
// * any Unicode character, excluding the surrogate blocks, FFFE, and FFFF.

Char = _{ "\u{0009}" | "\u{000A}" | "\u{000D}" |
          '\u{0020}'..'\u{D7FF}' | '\u{E000}'..'\u{FFFD}' |
          '\u{10000}'..'\u{10FFFF}' }
```

XML's EBNF encodes character ranges in square brackets (`[]`), with the first character of the range followed by a hyphen (`-`), and then the last character of the range.

Pest represents the same range as enclosing the first character in the range in single quotes (`'`), followed by two periods (`..`), then the last character of the range in single quotes (`'`), like so: `'a'..'z'`, which can be interpreted as "match any character that exist in the range from lowercase `a` to lowercase `z`, inclusive".

A character range is a type of alternation - it basically means "check if it matches the first option, and if it matches, accept that option and proceed to processing the next item in the sequence".  The more general form of alternation uses the pipe character (`|`).  This is true of both EBNF and pest.

The final interesting thing to note here is that the xml spec specifies character literals using either a hexadecimal representation of its unicode value (`#xA` - the "line feed" control character), or just putting the prescribed character inside single quotes (`'`) for standard ascii characters.  Pest is similar, but using rust-y notation for unicode literals - `\u{<hexadecimal>}`, for example, rendering the spec's `#x9` in pest as `\u{0009}` (leading zeroes added for clarity in alignment, and not because they're required).

You may also note that the opening curly brace for my `Char` production in pest is preceded by an underscore (`_`).  That tells pest not to emit a unique token for each `Char` that it parses.  The `Char`s will still be parsed, they'll just be returned as part of other productions that are built up using `Char` in them.

#### Side note

Pest has a [really neat playground](https://pest.rs/#editor) for testing out production rules and seeing what tokens they emit.

Go ahead and copy the following into the editor:
```

Char = _{ "\u{0009}" | "\u{000A}" | "\u{000D}" |
          '\u{0020}'..'\u{D7FF}' | '\u{E000}'..'\u{FFFD}' |
          '\u{10000}'..'\u{10FFFF}' }

S = _{ "\u{0020}" | "\u{0009}" | "\u{000D}" | "\u{000A}" }

Word = { (!S ~ Char)* }

Words = { Word ~ (S ~ Word)* }
```

Feel free to play around with it.  Try removing the leading underscores (`_`) on the `Char` and `S` productions.  The `Word` production basically says a word is any sequence of characters that isn't a whitespace char, and `Words` one or more `Word`s separated by one or more spaces.
