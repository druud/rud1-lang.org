# What rud1 does differently

This file lists where the choices of rud1 depart from what most languages do,
from what a newcomer meets first to the specialised. Each section gives the rule, an example
that was run, the reason as the spec or the project's history gives it (none where none is
written), and the status: **built** (the prototype does it today), **decided** (designed,
waits for a build) or **proposed** (not confirmed by the author).

Examples run as `rud1 'code'` or `rud1 file.rud`, a shell function around the executable,
which is named `rud1_run`; files end in `.rud`. `\\` starts a comment, so
`[rud1] code \\ result` is a program and what it printed. Positions in messages are
`line:column`. Contrasts were run with Perl 5.38.2, Python 3.12.3 and gp 2.15.4.

## 1. The result is shown, unless the text ends in `;`

The value of the last expression of a program is shown after its output. A final `;`
silences it. Display only: the value, and the way the program ends, stay the same.

```
[rud1] 1 + 1 \\ 2
[rud1] 1 + 1; \\ (no output)
```

Reason: gp's rule, taken over "a file never shows" and "never shown outside the REPL": one
rule for a file, a one-liner and the REPL, "the most predictable too" (the author).
Status: built.

## 2. Layout: no space before `=`, leading commas

A style, not a rule of the language: white space carries no meaning. The reference style
writes an assignment operator with no space before it and one after (`a= 1`, `x+= 1`,
`v[i]= e`); other operators keep their spaces. A long `my(…)` or call has one item per
line, the comma in front. Example code puts the constant first in `==` and `!=`, so that a
slip of `=` for `==` is a compile error.

```
[rud1]
my
( width= 3
, height= 4
);
width * height \\ 12
```

```
[rud1] my(n= 2); if(1 = n, "one", "other") \\ 1:16 error invalid assignment target
```

Reason: the missing space sets assignment apart, "as `:=` does elsewhere". Status: the
style is decided; a formatter that applies it is on the backlog.

## 3. Immutable by default

A name declared with `my(…)` can't be assigned again; `mut my(…)` makes it changeable.
Values are immutable unless their type says otherwise: if `w` is a (dynamic or fixed length)
array, then by default `w[0]= 9` builds a new list and rebinds `w`; in-place change comes
with the `{mut}` type attribute (decided, with the type pass). A function
captures the values of the names it uses, not the names.

```
[rud1] my(a= 1); a= 2 \\ 1:11 error assignment to immutable 'a'
[rud1] mut my(a= 1); a= 2; a \\ 2
[rud1] my(v= [1, 2, 3]); mut my(w= v); w[0]= 9; [v, w] \\ [[1,2,3],[9,2,3]]
[rud1] mut my(k= 1); my(f(x)= x + k); k= 10; f(1) \\ 2
```

Status: built.

## 4. Static checks before the program runs

A program is checked as a whole before its first step. An undeclared name, a name declared
twice in one `my(…)`, and a call of a known function with a wrong count or a wrong argument
name are compile errors: nothing has run.

```
[rud1] say("start"); say(total) \\ 1:19 error undeclared 'total'
[rud1] my(a= 1, a= 2); a \\ 1:1 error duplicate declaration 'a'
[rud1] my(g(a, b)= a - b); g(1, c: 3) \\ 1:21 error call of 'g': no parameter 'c'
```

`python3.12 -c 'print("start"); print(total)'` prints `start`, then
`NameError: name 'total' is not defined`.

A callee is known when it is an immutable name bound to a function. Through a `mut` name or
a parameter the same rule is checked when the call runs, and a break of it is an `error`.

Reason (spec, principles): "whatever can feasibly be solved (or proved) at compile time is,
even at the cost of extra compile rounds". Status: these checks are built. Checks on
declared types are decided, not built: `my(uint a= -1)` is accepted today.

## 5. Whole numbers and fractions are exact, of any size

`uint` (0 and up) and `sint` have arbitrary precision, and `/` on exact numbers gives an
exact rational. Nothing overflows in silence or becomes a float by itself.

```
[rud1] 2^64 + 1 \\ 18446744073709551617
[rud1] 1/3 + 1/6 \\ 1/2
[rud1] [7 / 2, 2^-3, 1/2 + 1, 1.5 * 2] \\ [7/2,1/8,3/2,3.00000000000000]
```

`perl -e 'print 2**64 + 1'` gives `1.84467440737096e+19`; `python3.12 -c 'print(1/3 + 1/6)'`
gives `0.5`; `echo '1/3 + 1/6' | gp -q` gives `1/2`.

A float has a dot with a digit after it (`1.5`), and an operation with a float operand
gives a float: today an IEEE binary64 value, shown in 15 digits.

Reason: gp's numbers; the author's first principle (generation 29): "Types are by default
of infinite precision." Status: `uint`, `sint` and rationals are built. Decided, not built: a
default float of infinite precision, native floats under names like `float8`, a native
`i64` for whoever asks for it.

## 6. A `uint` does not go below zero silently; prefix `+` makes a `sint`

A literal without a sign is a `uint`, and the difference of two `uint`s is a `uint`: it does
not silently become negative. A prefix `+` gives a `sint`, and the subtraction then happens
in `sint`. The rule protects an index: a position that went below 0 is not read as "from the
end", while a literal negative position is.

```
[rud1] +2 - 3 \\ -1
[rud1] my(a= 3, b= 5); abs(+a - b) \\ 2
[rud1] my(v= [10, 20, 30], i= 0); [v[-1], guard(v[i - 1], "none")] \\ [30,"none"]
```

`python3.12 -c 'print([10, 20, 30][0 - 1])'` prints `30`.

Reason (history, generation 11): the author reasons from full types, `nat 3 - nat 5` "will
of course error in nat-context but not in int-context"; a `uint - uint` of type `sint` with a
checked narrowing was rejected as "too lax in several places". Status: the `error` and the
prefix `+` are built. Decided, not built: a `sint` context from a declared type
(`my(sint d= a - b)`, a `sint` parameter or return type) widens the operands first.

## 7. Division and remainder are Euclidean

`//` is the whole quotient, `%` the remainder; the remainder is never negative, and
`a == b*(a // b) + a % b` holds. `/` is the exact division.

```
[rud1] [7 // -2, 7 % -2, -7 // 2, -7 % 2] \\ [-3,1,-4,1]
```

For the first two: `python3.12 -c 'print(7 // -2, 7 % -2)'` gives `-4 -1`;
`perl -e 'print int(7 / -2), " ", 7 % -2'` gives `-3 -1`; `echo '[7 \ -2, 7 % -2]' | gp -q`
gives `[-3, 1]`.

Reason: as gp's `\` and `%`, with the invariant above. Status: built.

## 8. Number literals: `0177` is an error

Decided in generation 24, built in generation 40: a `0` followed by a digit is an error,
`0_177` is octal (127), and one `_` between two digits of a number is ignored
(`1_000`; `1__2` and `1_` are errors). The same holds for `uint(s)` and
`sint(s)` and for a template's `$uint` and `$sint`. There are no hex literals.

```
[rud1] 0177 \\ 1:1 error malformed number '0177': a 0 before a digit (0_ starts an octal number)
[rud1] [0_177, 1_000_000, uint("1_000")] \\ [127,1000000,1000]
[rud1] 0x1F \\ 1:1 error malformed number '0x1F'
```

Reason: the readings elsewhere
disagree. `perl -e 'print 0177, " ", "0177" + 0'` gives `127 177`;
`python3.12 -c 'print(0177)'` is a `SyntaxError` ("leading zeros in decimal integer literals
are not permitted"); `echo '0177' | gp -q` gives `177`. Status: built; no hex (the author
on `0_177`: "though brittle, so maybe needs to be changed later").

## 9. A strict `bool`

`true` and `false` are values of a type of their own, shown as `1` and `0`. A `bool` casts
to a number in arithmetic, so a sum of comparisons counts. It is not compared with a number
and it is no index (`v[true]` is an `error`). Only a numeric zero is false: `[]` and `""`
are true.

```
[rud1] [true + true, sum([x > 3 | x <- [1, 5, 7]])] \\ [2,2]
[rud1] true == 1 \\ we exited <error("true == 1: a bool and a uint are not compared ('===' tells whether two values are the same)")> at 1:1
[rud1] [10, 20][uint(true)] \\ 20
```

`python3.12 -c 'print(True == 1, [10, 20][True])'` prints `True 20`.

Reason (the author, generation 29): "Built-ins should be strict where feasible"; a user's
type may allow more. Status: built.

## 10. `==` compares values, `===` asks for the same kind too

`==` promotes numbers as arithmetic does and ignores the names of a list's fields. `===`
is identity: exact numbers by value, a float only with a float, names included. Values of
different families are an `error` under `==` and plainly not identical under `===`.

```
[rud1] [1 == 1.0, 1 === 1.0, 0.1 == 1/10, 4/2 === 2] \\ [1,0,1,1]
[rud1] [[x: 1] == [1], [x: 1] === [1]] \\ [1,0]
[rud1] "1" == 1 \\ we exited <error("\"1\" == 1: a str and a uint are not compared ('===' tells whether two values are the same)")> at 1:1
[rud1] "1" === 1 \\ 0
```

Reason: `===` is gp's identity; `==` promotes because otherwise `0.1 - 1/10 == 0` and
`0.1 == 1/10` would disagree. On the `error` (generation 29): "let's not allow `==` between
differently typed values, unless they are really of the exact same family". Status: built;
the spec still marks the rule of `===` as proposed.

## 11. Comparisons chain, in one direction

`a < b <= c` means `a < b` and `b <= c`, with `b` evaluated once. A chain mixes `<`, `<=`,
`==` or `>`, `>=`, `==`, never both directions; `!=`, `!==` and `<=>` do not chain.

```
[rud1] [1 < 2 < 3, 3 > 2 > 1, 1 <= 1 < 2] \\ [1,1,1]
[rud1] 1 < 3 > 2 \\ 1:7 error comparison chain mixes '<' and '>'
[rud1] 1 < 2 != 3 \\ 1:7 error '!=' doesn't chain
```

`echo '3 > 2 > 1' | gp -q` gives `0` (the result of `3 > 2` against 1);
`python3.12 -c 'print(1 < 3 > 2)'` prints `True`.

Status: built.

## 12. Errors are states of a value, not exceptions

A result is a value, or a state with an origin. `undef` is soft: it is bound, passed and
combined, and the code around it goes on. `error` is hard: it ends each sequence, loop and
call around it, up to the nearest `guard(x, h)`, which gives `h` in its place. Each call
that an `error` ends is added to its stack. There is no `throw`, no `try`, no `return`. A
state is no value: it has no literal and no `==`; `is_defined(x)` asks.

```
[rud1]
my
( half(n)= if(0 == n % 2, n // 2, error("odd"))
, twice_half(n)= 2 * half(n)
);
say(guard(twice_half(7), -1));
my(r= max([]));
say(is_defined(r));
twice_half(7);
say("not reached")
```

Output:

```
-1
0
we exited <error("odd")> at 2:35
  in half at 3:22
  in twice_half at 8:1
```

A program whose last value is not defined ends with such a message and exit status 1.

Reason (history, generation 11): the author, "evaluation should stop at `1/0`", with a
full disk as the case (`write(f, d); say("saved")` must not say "saved"). "It is exception
behaviour done with values: no new mechanism, `guard` is the catch, a defer the cleanup."
Status: built; the stopping is proved in Lean per evaluation step. Not built: `state(x)`,
`origin(x)`, a handler that is a function receiving the failure. A run-time error of an
operator says why since generation 31 (`2 - 3` in section 6).

## 13. Positions count from 0, a literal negative one from the end

Lists are indexed from 0; a literal `-N` means `#__ - N`, where `__` is the list being
indexed, so `v[-1]` is the last element; a computed position below 0 is an `error`, not the
end of the list as in Python; several indices gather. A position
out of range is an `error`, reading as well as storing.

```
[rud1] my(v= [10, 20, 30, 40]); [v[0], v[-1], v[0, -1]] \\ [10,40,[10,40]]
[rud1] [10, 20, 30][3] \\ we exited <error("[10, 20, 30][3]: index 3 is outside 0 .. 2")> at 1:1
```

It departs from gp: `echo '[10, 20, 30][1]' | gp -q` gives `10`. Status: built. A 1-based
list type for translated gp code, `vector`, is proposed.

## 14. Ranges hold both ends, take a step after `;`, and splice

`[a..b]` runs from `a` to `b`, both included; backwards it is empty. A step follows a `;`
and its sign is the direction: `[a..b;s]`. A range among the other elements of a list is
spliced in, so `[1, 3..9]` is 1, then 3 to 9, and not the odd numbers. The same forms work
as an index; a stepped range stands alone in its brackets.

```
[rud1] [[1..4], [4..1], [1, 3..5, 9]] \\ [[1,2,3,4],[],[1,3,4,5,9]]
[rud1] [[1..9;2], [10..1;-3]] \\ [[1,3,5,7,9],[10,7,4,1]]
[rud1] my(v= [10, 20, 30, 40, 50]); [v[1..3], v[4..0;-2]] \\ [[20,30,40],[50,30,10]]
```

`python3.12 -c 'print(list(range(1, 4)))'` prints `[1, 2, 3]`.

Reason: the step mimics gp's `forstep`; the older spelling with a second element
(`[7, 5..0]`) went because it "is just ugly, is much harder to explain" (the author,
generation 26). The splice is the rule an index had already (`v[1, 3..5]`). Status: built
(generation 27). Open: a right-open range, omitted bounds.

## 15. Lists and comprehensions are lazy

A range and a comprehension `[e | x <- L, filter, …]` give their elements when asked, so a
list may be endless (`oo` as a bound).

```
[rud1] [x^2 | x <- [1..oo], 0 == x % 3][0..3] \\ [9,36,81,144]
```

gp's comprehensions are strict. Status: built by name: an element is computed again each
time it is asked for. The spec's rule (computed once, then kept) is not built.

## 16. An operator is a function with a name

A binary operator has the name `_op_`, one token without spaces, and the name is its
function as a value. The names are sealed: a script can't rebind `_+_`.

```
[rud1] [fold(_+_, [1..10]), _*_(6, 7), scan(_+_, [3, 1, 2])] \\ [55,42,[3,4,6]]
[rud1] my(_+_= 5); 1 \\ 1:4 error '_+_': names starting with '_' belong to the language
```

Reason: operators are calls of internal names, as in gp, where `_+_` is the operator's
internal name in the help (`?_+_`), in error messages (`_+_: forbidden addition`) and for
`alias("add", "_+_")`, though gp can't call `_+_(1,2)` itself (gp 2.19.0, run).
The spec's example of what sealing prevents: `_+_` rebound to a concatenation. Status: the
binary names as values are built; the prefix names (`-_`, `#_`) and operators declared by
a module are not.

## 17. Placeholders make a function of an argument

`_1`, `_2`, … inside an argument turn that argument into a function of as many parameters
as the highest number. The function is the whole argument the placeholder stands in, and a
call around a placeholder takes it as its own argument: `h(_1) + 1` is `h(x -> x) + 1`.
The second line shows both readings; here they happen to agree, and the compiler still warns
on `h(_1) + 1`, since a bare placeholder argument of a call that is itself an operand or an
argument is seldom meant so.

```
[rud1] [sum([1..10], _1^2), fold(_1 + 2 * _2, [1, 2, 3])] \\ [385,11]
[rud1] my(h(f)= f(1)); [h(_1 + 1), h(_1) + 1] \\ [2,2]
1:29 warning: '_1' alone makes the argument of 'h' the function x -> x: for a function of the whole, write 'x -> …' around the call
```

Where it matters, the arrow form names the variable: `d -> h(d) + 1`. The handler of
`guard` is the exception: `_1` there is the failed result itself (`guard(v[i], str(_1))`).

Reason (history, generation 12): rud1 is to be as short as gp, whose iterator forms
(`sum(i=1, 10, i^2)`) win otherwise; on the extent: "no rule without a delimiter is right
in every nesting, this one is short". Status: built. A policy switch on placeholders is
proposed.

## 18. Rest parameters, spreads, named arguments

One parameter, anywhere in the list, may collect the arguments the others leave: `v[...]`.
An argument `L[...]` spreads a list into a call. Every parameter can be given by name
(`z: 9`); the names bind first, then the unnamed arguments fill what is left.

```
[rud1]
my(f(a, v[...], z= 0)= [a, v, z]);
[f(1), f(1, 2, 3), f(1, 2, z: 9), f([4, 5][...])] \\ [[1,[],0],[1,[2,3],0],[1,[2],9],[4,[5],0]]
```

The binding is resolved when the program is compiled, wherever the callee is known (section
4). A rest takes unnamed arguments only, so a mistyped name is reported, never absorbed;
there is no options bag. Status: built. Decided, not built: binding by type (`sort(cmp, L)`
accepted with a warning that names the order).

## 19. `defer`

`defer(s)` runs `s` when the enclosing block ends, however it ends, also by an `error`. A
block is a `{…}` or a loop; a `defer` in a loop body runs at the end of each round. Several
run newest first.

```
[rud1]
{ defer(say("closed")); say("work"); 5 }
```

Output:

```
work
closed
5
```

A deferred function is called with the block's result, however the block ended; at an
`exit` it gets `undef`.

```
[rud1] my(log(r)= say("result ", r)); { defer(log); 5 } \\ result 5, then 5
[rud1] { defer((r) -> say("got ", is_defined(r))); exit(3) } \\ got 0
```

Reason (spec): a block is entered one way, at its top; "`defer` exists because there are
many ways out". Status: built. `amend(f)`, a defer that replaces the block's result, is
proposed.

## 20. Long names are the names; short ones come from a module

The names of the language are full words, taken from the standards where they have them
(the Unicode Standard's toUppercase is `str.to_uppercase`). A function that makes a text
has `str.` in front; `to_` marks a new value, `is_` a question.

```
[rud1] [str.to_uppercase("abc"), is_defined(max([]))] \\ ["ABC",0]
[rud1] str.uc("abc") \\ 1:1 error no function 'str.uc'
[rud1] my(isdef= is_defined); isdef(max([])) \\ 0
```

Reason (the author, generation 18): "The long names are the main names, the ones that the
compiler will put in a normalized transformation of a script"; "`str.to_uppercase` does
send the message more, that it doesn't change in-place"; "shortnames leave the `core`, so
an `alias.perl` module, or such, can bring perlish shortnames." Status: the names are
built. Modules are not: today a script binds its own short names, as in the last line, and
a dotted one (`str.uc`) has to wait.

## 21. A `str` is code points; `bytes` is another type; no `s[i]`

A `str` is always well-formed UTF-8 and its length counts code points. `bytes` is raw, with
its own literal `b"…"`. A `str` has no index: one takes a list first, and says of what.
Text is joined with `~`; `+` is never defined for it, and no text turns into a number by
itself (`uint("42")`, `str(42)`). A `str` is never normalized unasked.

```
[rud1] [#"hé", #b"h\x{c3}\x{a9}"] \\ [2,3]
[rud1] [str.to_codepoints("hé"), str.to_characters("hé")] \\ [[104,233],["h","é"]]
[rud1] "abc"[0] \\ we exited <error("\"abc\"[0]: a str has no positions (str.to_characters(s) lists its characters)")> at 1:1
[rud1] "abc" + "def" \\ we exited <error("\"abc\" + \"def\": no '+' for a str and a str")> at 1:1
```

`perl -e 'print length("hé")'` and `echo '#"hé"' | gp -q` give `3`;
`python3.12 -c 'print(len("hé"))'` gives `2`.

Reason: for `s[i]` the spec gives the rule only; for `+` it names the trap of one operator
that adds and joins. Status: built: `str` and its `str.` family; of `bytes` the literal,
`#`, the index, `~`, the comparisons and `str(b)`, the checked way to a `str`. Not built:
`str.to_graphemes`, `bytes(s)`, the compile error for a `str` joined with `bytes` (an
`error` when run, today).

## 22. Case mapping is the Unicode Standard's full one

`str.to_uppercase`, `str.to_lowercase` and `str.to_casefold` use the full mappings of the
Standard (data of Unicode 17.0.0): a result can be longer than its input, and lowercasing
follows Final_Sigma. No language or locale setting changes a result.

```
[rud1] [str.to_uppercase("straße"), str.to_lowercase("ΟΔΟΣ")] \\ ["STRASSE","οδος"]
```

`perl -CS -Mutf8 -e 'print lc("ΟΔΟΣ")'` gives `οδοσ`;
`python3.12 -c 'print("ΟΔΟΣ".lower())'` gives `οδος`.

Reason (history, generation 18): a simple uppercase leaves `straße` as `STRAßE`, "so a
lookup of an uppercased key misses"; without Final_Sigma "every Greek word ending in sigma
is misspelled". The price: lowercasing no longer works piece by piece. On locale: "stay
away from `locale` for now, just don't make it impossible". Status: built (generation 19).
Not built: `str.to_titlecase` of a whole text, `str.is_casefolded` (each an `error` that
says so), the rules of a language.

## 23. A byte escape has braces

A byte is `\x{89}` in hex, `\o{211}` in octal or `\b{10001001}` in binary, and values are
shown the same way. `\x89` without braces is a compile error. `\0` to `\7` are one digit
each, and a digit right after one is a compile error too (`"\012"`).

```
[rud1] b"\x{89}PNG" ~ b"\o{15}\b{00001010}" \\ b"\x{89}PNG\x{0d}\x{0a}"
[rud1] b"\x89PNG" \\ 1:1 error \x needs braces: \x{...}
```

`perl -e 'print "\012" eq "\n" ? "LF" : "other"'` prints `LF`.

Reason: the author, generation 23: "Let's make braces required for now"; they balance,
which a `t{…}` literal needs (section 25). `"\012"` would be three bytes here and is one
byte elsewhere, so it is refused. Status: built.

## 24. Reading changes nothing

A read gives what came: no line end taken off, no CR dropped, no signature removed, nothing
substituted. `read_line()` gives the text up to and including the next LF, `undef` at the
end of input, and an `error` for a piece that is not well-formed UTF-8.

```
[rud1]
my(a= read_line(), b= read_line(), c= read_line()); [a, b, is_defined(c)]
```

Run as `printf 'quit\r\nlast' | rud1 file.rud`. Output:

```
["quit\x{0d}\x{0a}","last",0]
```

With the bytes `quit` CR LF in a file `f.txt`,
`python3.12 -c 'print(repr(open("f.txt").readline()))'` prints `'quit\n'`.

Reason (the author, generation 20): "Never change any data, that is for the user to
decide." It departs, knowingly, from the Unicode Standard's recommendation to treat CR, LF,
CRLF and NEL alike on input. Status: the principle is decided; `read_line()` on standard
input is built; the reader in sends cut by time, the other verbs and files are not.

## 25. A template is a checked literal

`unpack` reads bytes by a template: Perl's `pack` codes, extended with sets, repeats,
alternatives and named patterns. A template is a literal `t{…}`, a value of its own type
and no `str`. It is parsed when the program is compiled, so a wrong template is a compile
error. The check goes beyond syntax: two variants of a `|` that can read the same text are
refused, with the shortest such text in the message; `||` ("or else") lets the first win.

```
[rud1] unpack(t{magic:a4 w:n}, b"\x{89}PNG\x{00}\x{10}") \\ [magic:b"\x{89}PNG",w:16]
[rud1] unpack(t{n a[=k]}, b"A") \\ 1:8 error template 1:5: the count =k is not named before its use
```

```
[rud1]
unpack(t{%[0-9A-Fa-f]+ | %[0-9]+}, b"15")
```

Output:

```
1:8 error template 1:1: variants 1 and 2 can read the same text, "1": make one of them narrower, or write '||' where the first is to win
```

Reason (history): a template in a cooked `"…"` was unreadable (`a["\n"]` became
`"a[\"\\n\"]"`), hence the raw literal; the check on variants has to hold for every input,
as a search over short texts "would leave a silent hole". Status: built: the literal, the
parser and its checks (generations 23 to 28), `unpack(t, b)` on a `bytes` value (generation
29). Not built: `unpack(t)` on a stream, `pack`. How strict the check on variants is, is
proposed.

## 26. `unpack` never tries again

The reader goes through the data once. A `|` takes the variant that reads furthest, not the
first listed. A repeat takes what it can and gives nothing back. The template has to cover
the data: bytes left over are an `error`, unless it ends in `x*`, which takes them.

```
[rud1] unpack(t{"foo" | "foot"}, b"foot") \\ [b"foot"]
[rud1] unpack(t{C x*}, b"AB") \\ [65,b"B"]
```

```
[rud1]
unpack(t{%[a-z]* "z"}, b"abz")
```

Output:

```
we exited <error("unpack(t{%[a-z]* \"z\"}, b\"abz\"): template 1:9: the data ends inside the literal \"z\": 1 wanted at byte 3, 0 left")> at 1:1
```

In Perl: `perl -e '"foot" =~ /foo|foot/; print $&'` gives `foo`;
`perl -e 'print unpack("C", "AB")'` gives `65`, the `B` ignored;
`perl -e 'print "abz" =~ /^[a-z]*z$/ ? "match" : "no match"'` gives `match`.

Reason (history, generation 25): a model of a reader that tries variants showed "the cost
doubling per nesting level"; a reader that never tries again is a function of the input,
which makes the check of section 25 possible. The price: a grammar written for a parser
that tries again may need rewording. Status: built (generation 29); parts of the rules are
proposed.

## 27. No silent limits

Decided in generation 29, as a first principle. The author: "It is one of the reason why
Lean 4 is the primary host-language: to catch such things. Will lead to interesting
discussions: no silent limits." Its first rule: a text is a finished value, so a cast or a
format whose text can be endless (a rational in decimal: `1/9`) is refused when compiled
unless the call gives a limit; by the type, not by the value.

Status: decided, not built (the format functions are not). The prototype still has silent
cases; an audit of generation 30 lists twelve. Two of them, a value dropped without a word,
are an `error` since generation 40 (decided in generation 34):

```
[rud1] mut my(a, b); [a, b]= [1, 2, 3]; [a, b] \\ we exited <error("destructuring [a, b]: more than 2 values for 2 names")> at 1:15
[rud1] mut my(v= [0, 0, 0]); v[0, 1]= [5, 6, 7]; v \\ we exited <error("scatter assignment: more than 2 values for 2 positions")> at 1:23
```

Others on that list: a type word that is not used (section 4), the binary float of section
5 (`0.1 + 0.2 == 0.3` is 0), and an old `[1, 3..9]` that changed its meaning with section
14. What counts as a silent limit (a value dropped, an argument ignored, a meaning changed,
a digit cut) is not written down yet.
