<p align="center"><img src="docs/rud1_logo.svg" alt="rud1" width="200"></p>

# rud1

**Exact numbers, errors as states of a value, and one checked template language that reads
and writes binary and text formats.**

rud1 (`.rud`) started out inspired by pari/gp, Perl and C++, and keeps their terse one-liners.
It has grown its own character: what can be decided before a program runs is decided then, a result that
went wrong is a state that travels with it, and reading and writing data share one
description. The prototype is written in [Lean 4](https://lean-lang.org), with proofs of
parts of its semantics; ports to C++, Rust and the JVM are to follow.

rud1 is meant for more than coders. Auditors, companies and AI tools are expected to
welcome it too, because it caters for their needs wherever feasible: no silent limits and no
silent ambiguity, checks before a program runs, errors that say where, and data formats
described once for reading and writing.

Every example below was run on the prototype; `\\` starts a comment, and what follows is what
came out.

## What is special already (built)

**Exact by default.** Numbers have no size limit by default: whole numbers and fractions in
the prototype, and a default float of unlimited precision is decided. Native types (`f64`,
types derived with a fixed width) remain available where speed matters. `/` gives a fraction
when the division does not come out, and the remainder is never negative.

```
[rud1] [7 / 2, 2^100, -7 % 2, 1/10 + 2/10 == 3/10]  \\ [7/2,1267650600228229401496703205376,1,1]
```

**Errors are states, not exceptions.** A result that went wrong carries its state (`error`,
`undef`, …) and travels outward until something catches it: an `if` without "else" whose
condition is false is `undef`, and `guard` gives a fallback.

```
[rud1] my(half(n)= if(0 == n % 2, n // 2)); [half(10), guard(half(7), "odd")]  \\ [5,"odd"]
```

**One description for reading and writing.** A template `t{…}` is a checked literal: it is
compiled with the program, and `unpack` reads bytes by it, `pack` writes them.

```
[rud1] unpack(t{id:n flags:C size:V}, b"\x{01}\x{02}\x{07}\x{10}\x{00}\x{00}\x{00}")  \\ [id:258,flags:7,size:16]
[rud1] pack(t{id:n flags:C size:V}, [id:258, flags:7, size:16])  \\ b"\x{01}\x{02}\x{07}\x{10}\x{00}\x{00}\x{00}"
```

**Templates and comments are first-class.** A template is a value of its own type, as an
integer is: it is bound to a name, kept in a list, joined with `~`, compared, shown. So is a
doc comment `/*{ … }*/`: `_doc_()` gives the one of the block it stands in (a format handler
can turn its text into data).

```
[rud1] my(t= t{id:n flags:C}); [str(t), t ~ t{V}]  \\ ["id:n flags:C",t{id:n flags:C V}]
[rud1] my(help()= { /*{ prints help }*/ _doc_() }); help()  \\ " prints help "
```

**No silent ambiguity.** `unpack` never tries again: two variants that can read the same
text are refused when the template is compiled, before any data is read, unless `||` says
which one wins. And `pack` reads back what it wrote: bytes that would not read back as the
same tree are an error, not a file that some other reader takes differently.

```
[rud1] unpack(t{$dec+ | $hex[2]}, b"15")  \\ 1:8 error template 1:1: variants 1 and 2 can read the same text, "11": make one of them narrower, or write '||' where the first is to win
[rud1] unpack(t{$dec+ || $hex[2]}, b"15")  \\ [15]
```

**Unicode as the Standard has it.** A `str` is code points, `bytes` is another type, and
case mapping is the Standard's full one.

```
[rud1] str.to_uppercase("straße")  \\ "STRASSE"
```

**Terse where it counts.** Comprehensions, ranges that hold both ends, placeholders, `defer`,
named arguments, lazy lists.

```
[rud1] [x^2 | x <- [1..10], 0 == x % 3]  \\ [9,36,81]
```

## What is to come (decided or planned)

- A type pass: types inferred, the run-time checks of today made static.
- The format translator, written with templates as their first user; `printf` whose
  sub-formats are functions, so that modules add conversions; later `pandoc.rud`.
- Templates that change what they read next from the data (hooks), and the rest of text:
  normalization, word boundaries.
- A compiler, with ports of the prototype to C++, Rust and the JVM; a compiled program that
  carries enough to give back its normalized source.
- A small core, with the rest written in rud1 itself: a port to a host language replaces a
  feature by the host's own where that is faster, and is checked against the rud1
  implementation on generated data; an install may also keep to rud1's own, slower and purer.
  Where proof matters, an executable with only the core ported runs the program, or gives
  the final check of a faster one's results.
- Migration tools, such as `pari2rud1.rud` from pari/gp (its rules are collected in the
  `pari2rud1` manual).
- `gp.rud`: pari/gp's names, for coders who come from gp.
- For auditors: a mode that traces a value dynamically through a process, and through the
  systems it passes (databases, requests, streams), with near-real-time analysis that flags
  anomalies and violations, such as guarded data that crosses to where it may not go.

## Documents

| Document | Content |
|---|---|
| [Not yet](docs/rud1_not_yet.md) | what is decided or proposed and not yet built |

An introduction, examples, the differences from other languages and the gotchas are coming
soon. Each example in the documents is run again before a document is published.

## Licence

- Code (Lean, Python, scripts): MIT, see `LICENSE`.
- Text, figures and other documents (papers, `.md`, `.tex`, images): CC BY 4.0, see `LICENSE-CC-BY-4.0`.
