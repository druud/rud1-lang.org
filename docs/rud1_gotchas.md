# rud1 gotchas

Places where rud1 does something a coder may not expect, each
with the smallest case. Started in generation 25 (the user: "it should be listed in a
`rud1-gotchas` doc"). Most entries are about the template language of `unpack` and `pack`
(spec §5d). **Designed** means decided and not built; **built** means the prototype does it
today. "Run" means the counterpart was run in Perl 5.38.2, gp 2.15.4 or the prototype.

The entries were last checked in generation 30: the audit of that generation ran the
examples in the prototype (its executable is `rud1_run` since generation 31, files end in `.rud`), and
a status mark says what the prototype did then. The counterparts in Perl, Python and C were
not run again. Entries 54 on come from that audit. `pack` is not built: what an entry says of
it is designed.

## From Perl's regex to a template

A backslash letter that stands outside a quote in Perl is `&` and the same letter here. A
backslash form exists only inside a quote or a set. The
parser reads since generation 26: the two rows on sets (`%[^\(\)]`, `%[a-f 0-9]`),
`x* x+ x?`, `x[2..5]`, `a | b` and `a || b`; since generation 27 also the letters that are
sets (`&d` to `&N`, `&R`, `&alpha`) and `??[…]`; since generation 28 also `&b &B &K`, the
groups `(?i: … )`, `(?u: … )`, `(?= … )`, `(?! … )`, `(*commit)`, `(*fail: "msg")`,
`"\g{name}"` and `\p{…}`. Since generation 29 `unpack` reads them from data: the table is
built (run in generation 30), but for the rows that say "not built" or "none".

| Perl | rud1 | Mind |
|---|---|---|
| `\d \D \s \S \w \W` | `&d &D &s &S &w &W` | ASCII only, always (Perl: 680 digits without `/a`, run) |
| `\h \H \v \V` | `&h &H &v &V` | ASCII only (entry 8) |
| `\N` | `&N` | any byte but LF |
| `\N{name}` | `\N{name}` | inside a quote or a set; a character by its name; not built (it waits for the table of names) |
| `\R` | `&R` | CR LF as two bytes, or one of `&v` |
| `\b \B` | `&b &B` | the edges of the data count as non-word, and so do the edges of a sized group |
| `\b{wb}` and kin | none yet | will not be `&b{…}`; `\b{…}` in a quote is a byte in binary (entry 22) |
| `\K` | `&K` | the result shows a count, not the bytes (entry 17) |
| `\X` | `&X` | spelling fixed, not built |
| `\p{…} \P{…}` | the same | inside a set, within `(?u: … )` only; General_Category only, the name exact (entries 41, 42) |
| `[[:alpha:]]` | `&alpha` | 14 POSIX names; inside a set too: `%[&alpha _]`; not `%[[:alpha:]]` (entry 59) |
| `[^()]` | `%[^\(\)]` | punctuation is escaped, or in an inner bracket (entry 6) |
| `[a-f 0-9]` under `/xx` | `%[a-f 0-9]` | blanks ignored (entry 4) |
| `x* x+ x?` | the same | never give back: Perl's `x*+ x++ x?+` (entry 9) |
| `x{2,5}` | `x[2..5]` | `[..5]`, `[2..]`, `[3]` |
| `a\|b` | `a \| b` | the longest wins; a tie is a compile error, let through it goes to the first listed (entries 1, 2) |
| `(?>a\|b)` | `a \|\| b` | "or else": `b` only when `a` can't be read |
| `(?i: … )` `(?= … )` `(?! … )` | the same | `(?i: … )` takes no repeat and no name (entry 39); a look-ahead keeps its names (entry 40) |
| `(*COMMIT)` `(*FAIL:msg)` | `(*commit)` `(*fail: "msg")` | |
| `\g{name}` | `"\g{name}"` | in a quote; the bytes of an earlier field |
| `(?(cond) … )` | `??[…]` at the head of a variant | a test on counts read earlier |
| `(?{ … })` | hooks | not designed |
| `\A \z \Z \G`, look-behind, `x*?` | none | |
| `/x` | always | white space and `#` comments are free outside quotes |

## Templates: reading

1.  **Two variants of a `|` can read the same bytes.** With `&hex ::= %[0-9A-Fa-f]+ ;` and
    `&dec ::= %[0-9]+ ;`, the bytes `15` are read by both variants of `&hex | &dec`. Three
    guards: while ranking is strict (the default) such a `|` is a compile
    error that shows the shortest text both read (`1`; built in generation 28: "variants 1
    and 2 can read the same text, "1""); let through, a tie goes to the first listed (the
    reader's rule \[BR\] of generation 29, confirmed in generation 34);
    and `pack` reads back what it wrote (designed). With ranking lax the first listed wins silently:
    the decimal 15 is written as `15` and read as hex. Write `||` where one variant is meant
    to win (`&hex || &dec` reads `15` as `[[b"15"]]`, run).
    Only `policy.ranking` 2 lets a tie through the compiler (`rud1_template -p.ranking=2`
    takes the `|`, run). A template can't ask for it ("policy.ranking comes from outside a
    template") and `rud1_run` has no option for it, so in a program no tie reaches the
    reader today.
2.  **`|` takes the longest variant; Perl's takes the first listed.** `"foo" | "foot"` on
    `foot` reads `foot`; Perl's `foo|foot` on `barefoot` gives `foo` (run). Built (generation
    29: `unpack(t{"foo" | "foot"}, b"foot")` is `[b"foot"]`); the compile-time checks go by it
    (generation 28).
3.  **`X??[e]` is `X`, then a test.** In Perl `X??` is the non-greedy optional (run). An
    optional `X` before a test is `X???[e]`. `unpack(t{"a"??[1 == 1]}, b"a")` is `[b"a"]`,
    `unpack(t{"a"???[1 == 1]}, b"")` is `[[]]`. Built (the parser in generation 27, the
    reader in generation 29).
4.  **Blanks inside a set are ignored.** `%[ \t]` holds the tab only; a space is `\x{20}` or
    `\ `. Perl's `/xx` does the same (run). Built (generation 26).
5.  **A `^` in the middle of a set is "but not".** `%[&w ^ _ &d]` is the letters. In Perl's
    `(?[ … ])` a binary `^` is the symmetric difference (run). The `^` is built (generation
    26: `%[a-f ^ c-e]` is `abf`), and so are the built-in names in a set (generation 29: the
    example with `*` reads `ab` and stops at `_` and at `1`).
6.  **Punctuation in a set is not plain.** At the outer level of `%[…]` a punctuation byte
    that is no syntax there is a compile error while the setting is strict; it is escaped
    (`%[^\(\)]`) or stands in an inner bracket (`%[^[()]]`), and the normalized form writes
    the backslash. Generation 24 wrote `%[^,]`: now `%[^\,]`.
    Built (generation 26). The setting is `policy.punctuation.bare`: a template asks for
    lax with the header `(*policy.punctuation.bare: 2)` at its top, `rud1_template -p.punctuation.bare=N` gives the value from outside, and a `t{…}` in a program is strict unless its header
    asks.
7.  **`(?i: … )` does not reach into a pattern used inside it.** `(?i: &kw)` with
    `&kw ::= "select" ;` refuses `SELECT`; the definition carries its own group. Perl's
    modifiers behave so too (run). Built (generation 29: `&kw ::= (?i: "select") ;` reads
    `SELECT`); the rule is proposed.
8.  **`&h` and `&v` are ASCII only.** Perl's `\h` also holds `\x{a0}` and its `\v` `\x{85}`,
    even under `/a` (run). Built (generation 29: `&h` refuses `\x{a0}`, `&v` refuses `\x{85}`).
9.  **A repeat does not look at what follows it.** `(&h16 ":")[..5] &h16 "::"` refuses
    `1:2::3`: the second round takes `2:`. Reworded as `&h16 (":" &h16)[..5] "::"` it reads
    (a model run on 729951 addresses). Perl's possessive `a++a` fails the same way (run).
    Built (generation 29: with `&h16 ::= %[0-9a-f]+ ;` before each and `x*` after each for the
    `3`, the first is an `error` at byte 4, the second is `[[b"1"],[[b":",[b"2"]]],b"::",b"3"]`).
10. **A star leaves room for a tail only on a code.** `data:a* crc:V isize:V` works; a set's
    run (`%[…]*`) and a group's repeat stay local. Built (generation 29: the first is
    `[data:b"hello",crc:1,isize:2]`; `%[a-z]* "o"` on `hello` and `("a")* "a"` on `aaa` are
    an `error`); which codes: proposed (entry 52).
11. **`$uint` is not `$dec`.** `$uint` follows the number rule of the language and refuses
    `0177`; `$dec+` reads it as 177, `$oct+` as 127. `$dec` and `$oct` alone read one digit:
    of `0177` the `0`, and the three bytes left over are an `error` (entry 50). How many
    digits is a count or a repeat (`$dec[4]`, `$dec+`). Built (generation 29).
12. **A part that is there 0 or 1 times is a list of rounds.** `name:(a["\x{0}"])[=fname]`
    gives `name:[[b"name.txt\x{00}"]]` or `name:[]`: the value is `r.name[0][0]`. Built
    (generation 29; run with `=fname:C` before it).
13. **A required round may be empty.** `(%[0-9]*)+` on `x` is one empty round, no `error`.
    The empty round that ends an open repeat shows in the uncondensed tree only. Built
    (generation 29: with `x*` after it for the `x`, `[[[b""]],b"x"]`); the prototype has no
    way to show the uncondensed tree, so that sentence is not run.
14. **Without `(*commit)` a broken variant falls through to `||`.** `"select" &s+ $dec+ || name:&w+ &s+ arg:&w+` reads `select x` by the second part, as a statement named
    `select`. Perl's `(*COMMIT)` is the counterpart (run). Built (generation 29:
    `[name:b"select",b" ",arg:b"x"]`; with `(*commit)` after `"select"` an `error`).
15. **A reference that turned out empty still matches.** `"\g{tag}"` with an empty `tag`
    reads nothing and is an item of the result. Built (generation 29: `tag:a0 "\g{tag}"` on
    no data is `[tag:b"",b""]`, `tag:%[a-z]* "," "\g{tag}"` on `,` is `[tag:b"",b",",b""]`).
    Not with a star on a code: `tag:a* "\g{tag}"` is the `error` "=tag is not read" on any
    data, since the `*` asks for the width of its tail, which is the field itself (found in
    generation 30, not decided).
16. **A template can be unambiguous and still disagree with another program's reader.** A
    zip comment can hold a complete fake end record; Python's `zipfile` and UnZip 6.00 both
    take the last one (run in generation 24). No check inside a template sees that.
17. **`&K` drops bytes from the condensed result.** What its bracket pair read before it
    shows as one number, the count of bytes. It is the one place where the condensed result
    and the template do not give the data back; the uncondensed tree does. An `x` skip
    keeps its bytes. Built (generation 29: `a2 &K C` on `abc` is `[2,99]`, `(x2 C)` is
    `[[b"ab",99]]`); the prototype has no way to show the uncondensed tree.
18. **Inside `(?u: … )` a count counts code points; a size counts bytes, always.** A zip's
    name length is in bytes: `([=nlen] (?u: name:a*))`, not `(?u: name:a[=nlen])`. Built
    (generation 29: with `=nlen:C` before each, on the byte 3 and `h\x{c3}\x{a9}` the first is
    `[nlen:3,[name:"hé"]]`, the second an `error`); the size in bytes: proposed.
19. **`U` is not Perl's `U`.** It reads one code point of UTF-8 and gives its number, in
    every mode. Perl's `unpack("U*")` on the bytes `c3 a9` gives 195 169; 233 comes only as
    `C0U*` (run). Built (generation 29: `unpack(t{U*}, b"\x{c3}\x{a9}")` is `[[233]]`).
20. **Three `N`s.** `&N` is any byte but LF, `\N{name}` a character by its name, `\p{N}` the
    numbers (1831 code points at Unicode 15.0.0, run). `&N` and `\p{N}` are built (parsed
    in generations 27 and 28, read in generation 29: `&N` reads `5` and refuses a line feed,
    `(?u: %[\p{N}])` reads `½`); `\N{name}` is not built (it waits for the table of names).
21. **A setting has three values, and 0 is not "off".** 0 = strict, refuse any overrides;
    1 = strict, allow overrides to 2 or 0; 2 = lax, allow overrides to 1 or 0. Asking for 1
    or 2 where something outside says 0 is a compile error. Ranking is on at 2, not at 1
    (generation 24: on at 1). Built (run in generation 30 through `rud1_template -p.NAME=N`:
    `punctuation.bare` 0 from outside refuses a header that asks for 2, 1 allows it, and 2
    with a header that asks for 1 is strict; `ranking` 2 lets a tie of two variants through,
    1 does not). `rud1_run` has no option for a setting.

## Templates: codes (against Perl's `pack`)

22. **`\b{…}` in a quote is a byte written in binary** (`\b{10001001}` is `\x{89}`), `\o{…}`
    one in octal; `\x89` without braces is an error. Perl's `\b` is a boundary, or a
    backspace in a class. Built (generation 23).
23. **`b` and `B` give a number of that many bits**, not Perl's text of `0`s and `1`s; `b`
    takes the low bits of a byte first. Bit fields in a row share bytes. Built (generation
    29: on the byte `0d`, `b3 b5` is `[5,1]` and `B3 B5` is `[0,13]`).
24. **`s S l L q Q` are little-endian without a mark**, never the machine's order; `i I j J`, `p P` and `!` are gone. `unpack(t{s S}, b"\x{01}\x{00}\x{01}\x{00}")` is `[1,1]`.
    Built (the parser and the reader).
25. **`W` is the varint with the low group first** (300 is `AC 02`); Perl's `w` has the
    high group first and stays; Perl's own `W` is gone. `unpack(t{W}, b"\x{ac}\x{02}")` and
    `unpack(t{w}, b"\x{82}\x{2c}")` are both `[300]`. A varint written longer than
    needed (`AC 82 00`) is an `error`, where Perl's `w` reads it; the template's header
    `(*policy.varint.overlong: 2)` reads it too:
    `unpack(t{(*policy.varint.overlong: 2) W}, b"\x{ac}\x{82}\x{00}")` is `[300]`.
    Built (the parser and the reader).
26. **`.` is the start mark, `@` counts from it**; a group that starts with a jump comes
    back when it closes (a visit); `=here:@` reads nothing and holds the offset; a bare `@`
    is a compile error. Built (the parser: the position in generation 27; `unpack` in
    generation 29: `(@2 C) C x*` on the bytes 1, 2, 3 is `[[2,3],1,b"\x{02}\x{03}"]`, the
    group read at offset 2 and the `C` after it at 0; `C =here:@ C` is `[1,here:1,2]`).
27. **`/` is short for a count read before its field.** `t:C/a` reads a `C`, then that many
    bytes as `a`: `unpack(t{t:C/a}, b"\x{02}hi")` is `[2,t:b"hi"]`. The name goes to the
    field; the count stands before it without a name, and is an item of the result too. With
    a name for the count the long form is `=t:C u:a[=t]` (`[t:2,u:b"hi"]`). `=t:C t:a[=t]`,
    which this entry gave and the spec gives as the long form, is a compile error ("the name
    t is used twice in one bracket pair"): what the long form of `t:C/a` is, is open. Nothing
    read stays out of the result (but entry 17). Built (generation 29).
28. **A name can start with a digit.** `1:C` is a field named `1` (run on the parser).

## The language

29. **A list that holds an undefined element is not a plain value.** Showing it whole ends
    the program (`we exited <missing(…)>`, run); `is_defined(r.name)` on it works. Built.
30. **gp's `v[^i]` has no counterpart.** In gp `[10, 20, 30][^2]` is `[10, 30]` (run); here
    `^` before a name is the outer-name prefix, and `v[^1]` is the compile error "'^' needs
    a name" (run). Today: `[v[i] | i <- [0..#v-1], 1 != i]`.
31. **`0177` is an error, `0_177` is octal.** Perl reads `0177` as 127 in source and as 177
    from a string, gp as 177 (run in generation 24). Built (generation 40): `0177` is
    `1:1 error malformed number '0177': a 0 before a digit (0_ starts an octal number)`,
    `0_177` is 127, also in `uint("0_177")` and a template's `$uint`; one `_` between two
    digits is ignored (`1_000`), `1__2` and `1_` are errors.

## Templates: later entries

32. **A number after a pattern's name is part of the name.** `&row3` is the pattern `row3`;
    `&row` three times is `&row[3]`. After a code or a group a bare number is a count (`a3`,
    `(n C)3`), as in Perl's `pack`. Built (generation 26; the brackets: proposed).

33. **`%[0-9]` is one byte.** C's `scanf` reads `%[0-9]` as a run that is not empty (run in
    generation 24); here the run is `%[0-9]*`, and it may be empty. Generation 23's
    `a['0-9']` was a run: it is now a compile error that names `%[0-9]*`. Built (generation
    26).

34. **`q{a\}` does not close.** In a `name{…}` literal a backslash keeps the next byte out
    of the brace count, and stays in the text: `q{a\}b}` is the four bytes `a\}b`, and
    `t{%[\}]}` is a set of one brace. A raw text cannot end in one backslash. Perl's
    `q{a\}b}` is `a}b`: there the backslash goes. Built (generation 26; before, the lexer
    counted every brace).

35. **`\0` to `\7` are bytes, and one digit is all of it.** `"\0"` is a zero byte, `"\7"` the
    byte 7; `\8` and `\9` are errors. `"\012"` is a compile error: in Perl 5.38.2, Python
    3.13, gcc 13.3 and bash's `printf` it is one byte, a line feed (run); write `\o{12}` or
    `\n`. `\1` is never a group again (Perl's regex; a field again is `\g{name}`). `\9` is
    a TAB in none of the four: the digit `9` with a warning in Perl and C, an invalid
    escape in Python, a backslash and a `9` in `printf`. A TAB is `\t`, `\x{9}` or `\o{11}`.
    Built (generation 26).

36. **`v[-1..0;-1]` is empty, not the reverse.** A range in an index is a range of
    numbers first, and its numbers become positions after that: from −1 down to 0 there
    is nothing, since −1 is below 0. The reverse of `v` is `v[#v-1..0;-1]`; the last three
    are `v[-3..-1]` (run). Built (generation 27). The entry that stood here, `v[5, 3..1]`
    against the list `[5, 3..1]`, went with the old spelling of the stepped range.

37. **`[1, 3 .. 9]` is not the odd numbers.** A range among the elements of a list is
    spliced in, as in an index: `[1, 3 .. 9]` is `[1,3,4,5,6,7,8,9]`. The step of a range
    follows a `;`: `[1..9;2]`, `[10..1;-3]`, as gp's `forstep(i=10, 1, -3, …)`, and a stepped
    range stands alone in its brackets (`v[0, 5..1;-2]` is an error). The form with the
    second element was Haskell's, and this language's until generation 26: an old
    `[10, 7 .. 1]` is `[10]` now, with a compile-time warning since `7 .. 1` is empty; an
    old `[1, 3 .. 99]` gets none. Built (generation 27), decided.

38. **A list of steps goes one way.** `[1..20;[2,4]]` is gp's `forstep(i=1, 20, [2, 4], …)`.
    gp takes the direction from the sum of the vector, so `[3, -1]` from 1 to 10 walks
    1, 4, 3, 6, 5, 8, 7, 10, 9 (run); here the first step is the direction, and a 0 or a
    step against it is an `error` at the element where it is met. Built (generation 27);
    the rule is proposed.

39. **`(?i: … )` and `(?u: … )` are a mode, no group of the result.** The items in one are
    items of the bracket pair around it: `(?i: =n:C) a[=n]` sees `=n`, and `(?i: n:C) n:C` is
    a name twice. A repeat stands on a group inside, `(?i: ("ab")*)`; `(?i: "ab")*` and
    `kw:(?i: "select")` are compile errors. Perl's `(?i: … )` clusters without capturing
    too, and takes a quantifier. The parser is built (generation 28); the rule is proposed.

40. **A look-ahead keeps its names.** `(?= =n:C) a[=n]` is a compile error: nothing named in
    `(?= … )` or `(?! … )` is in sight after it. In Perl a group inside a look-ahead keeps
    what it captured (run: `"ab" =~ /(?=(a))ab/` sets `$1` to `a`). The parser is built
    (generation 28); the rule is proposed.

41. **Inside `(?u: … )` a set holds code points, no bytes.** `%[\x{e9}]` is a compile error
    there (it is `%[\u{e9}]`, or `%[é]`), and so is `%[&D]`, which holds bytes above 0x7F:
    "all but the digits" is `%[^&d]`. The normalized form writes `\u{e9}`. Outside the group
    `\u{…}` and `\p{…}` in a set are compile errors. As an item, `&D` inside the group still
    reads one byte, so it can stop inside a character (open). The parser is built
    (generation 28); the rules are proposed.

42. **`\p{lu}` is not `\p{Lu}`.** A value of General_Category is written exactly as the
    UCD's short or long name (`\p{Lu}`, `\p{Uppercase_Letter}`; `\p{L}`, `\p{LC}` for the
    groups). Perl matches loosely: `\p{lu}`, `\p{uppercase letter}` and `\p{Digit}` all
    match there (run). Other properties (`\p{Greek}`) are not built. The parser is built
    (generation 28); the exact name is proposed.

43. **A variant can't start like another and then take its width from the data.** The
    reader never tries a variant and gives it up, so where the look-ahead of one variant
    stops (a count from the data, a recursion, a bit field), no other variant may still be
    possible: `"foo" =n:C a[=n] | "foot"` and `&p ::= C | "(" &p ")" ;` are compile errors
    that show the shortest input (`"foot"`, `"("`). Factor out what they share, make one
    narrower (`%[a-z] | "(" &p ")"`), or write `||`. Built (generation 28); how strict it
    is, is proposed.

44. **`&e ::= &e "-" C | C ;` is a compile error.** Left recursion never reads a byte; the
    message names the repeat that says the same, `&term ("-" &term)*`. A yacc grammar is
    written so; a template is not. Built (generation 28).

45. **Inside `(?u: … )`, alternatives of letters want `||`.** The look-ahead goes by bytes
    and stops at a set that holds code points above 0x7F, so `(?u: %[à] | "x")` is "still
    alike". `(?u: %[à] || "x")` reads. Built so (generation 28); a look-ahead that reads
    UTF-8 is the text round's.

46. **A template is no `str`, and its `str` is not what you typed.** `t{C n} == "C n"`,
    `t{C n} ~ " v"`, `#t` and `str.to_uppercase(t)` are all an `error`: a `template` is
    never used as a `str` by itself. Say which is meant: `str(t) == "C n"` or
    `t == template("C  n")`, `str(t) ~ " v"` or `t ~ template(" v")`, `#str(t)`. And `str(t)`
    is the normalized form: `str(t{a[4]})` is `"a4"`, `str(t{C  n})` is `"C n"`, without
    comments and without a policy header. `say(t)` writes it; a result shows `t{a4}`. Two
    templates compare by it (`t{C  n} == t{C n}`), which is the same spelling, not the same
    meaning (`t{%[0-9]} !== t{&d}`); `t === "C n"` is false without an `error`. Built
    (generation 29), the user's rules; a type of your own may allow more.

47. **`t ~ u` of two templates is not their texts glued.** It is the template that reads
    `t`, then `u`: a join of the two trees. `t{C} ~ t{n}`
    is `t{C n}` (`"C" ~ "n"` is `"Cn"`), and `t ~* 2` is `t` joined twice. A
    `|` at the top of a piece does not take the neighbour into its variant:
    `t{"a" | "b"} ~ t{C}` is `t{("a" | "b") C}`. The patterns of both pieces are
    pooled (one defined in both, differently, is an `error`), and so are the headers. Each piece has
    to be a template by itself: a count can't be named in one piece and used in the next
    (`t{ a[=n]}` alone is a compile error). A join that is no template is an `error` at
    run time, a compile error for two literals. Built (generation 29; the tree join decided in
    generation 34, built in generation 40).

48. **A text field is `bytes`, not a `str`.** `unpack(t{name:C/a}, b"\x{02}hi")` is
    `[2, name:b"hi"]`: what was read is bytes, and nothing says they are UTF-8.
    `str(r.name)` is the checked way to a `str` (an `error` when they are not). Hex digits
    (`h`, `H`) are a `str`. Built (generation 29); proposed.

49. **`C3` is a list, and so is `C1`.** A count on a number code gives a list of that many
    numbers (`[1,2,3]`, `[1]`); without a count it is the number itself. Perl gives a flat
    list either way. The normalized form keeps `C1` apart from `C` for that reason. Built
    (generation 29); proposed.

50. **`unpack` has to cover its data.** Bytes left over are an `error`; a template that
    ends in `x*` (the rest as one skipped item) or `@*` (a jump to the end) takes them.
    Perl's `unpack` ignores what is left. Built (generation 29).

51. **An unnamed group stays unnamed, also with a repeat.** `(name:a["\x{0}"])[=fname]`
    gives `[[name:b"hi.txt\x{00}"]]` as an unnamed entry: the name belongs to the field
    inside each round. To reach the rounds by a name, name the group:
    `names:(a["\x{0}"])[=fname]`. Built (generation 29).

52. **`x* C` leaves the last byte for `C`.** A `*` on a text, a skip or a number code runs
    to the end, less what the rest of its row takes when that is known before it is read
    (`data:a* crc:V isize:V`, gzip's tail). With anything of unknown width after it, the `*`
    runs to the end. Built (generation 29); proposed.

53. **A failure inside `(*commit)` or a recursive pattern does not fall through.**
    `("a" C)* x*` on `a\x{01}a` reads one round and leaves `a` to `x*`; with
    `("a" (*commit) C)*` the second round's missing byte is an `error`.

    - `(*commit)`: built (generation 29; out of alternatives and repeats since generation
      31). `"a" (*commit) "b" || x*` and `"a" (*commit) ("b" | "c") || x*` on `ad` are an
      `error`, and so is `("a" (*commit) ("b" | "c"))* x*` on `abad`.
    - A recursive pattern: built (generation 31; decided in generation 30, label J). A
      failure is final once a variant or a round has read bytes and then enters a
      recursive pattern, and it stays final after that pattern returns. With
      `&r ::= "(" &r ")" | "a" ;`, `&r`, `&r || x*` and `(&r)* x*` on `(a` are each an
      `error` (the `)` is missing); `(&r)* x*` on `(a)zz` reads one round and leaves `zz`
      to `x*`, since nothing was read when the second round met `z`.

## From the audit of generation 30

54. **A surplus value in a destructuring or scatter assignment is an `error`.**
    `my([a, b]= [1, 2, 3]); [a, b]` exits with
    `error("destructuring [a, b]: more than 2 values for 2 names")`, and so does
    `mut my(a, b); [a, b]= [1, 2, 3]; [a, b]`; `mut my(v= [1, 2, 3]); v[0, 1]= [5, 6, 7]; v` with
    `error("scatter assignment: more than 2 values for 2 positions")`, before any element is
    written. A right side that is too short is an `error` too (`[a, b]= [1]`: an index out
    of range). Built (generation 40; decided in generation 34).
55. **A placeholder makes the nearest enclosing argument the function.** With
    `my(h(n)= n * 10)`, `h(_1) + 1` is `h(x -> x) + 1`, an `error` (a function times 10),
    and the compiler warns: "'_1' alone makes the argument of 'h' the function x -\> x: for a
    function of the whole, write 'x -\> …' around the call"; the function of `d` is
    `d -> h(d) + 1` (`my(h(n)= n * 10, f= d -> h(d) + 1); f(2)` is 21). Where `h` takes a
    function nothing tells the two apart: `my(h(f)= f(1)); h(_1) + 1` is 2, with the same
    warning. The handler of `guard` is the exception: in `guard(error("x"), say(_1.message));`
    the `_1` is the failed result itself, not a function, so `say` gets the `error`,
    which ends its call, and the program exits with `error("x")` (the reader `.message` is not
    designed). Built (generation 40; the rule of generation 13).
56. **A whole rational is a whole number.** `4/2` is the `uint` 2 (`-4/2` the `sint` -2), as gp's
    `4/2` is a `t_INT`: `4/2 === 2` is 1, and `[10, 20, 30][4/2]` is 30, as
    `[10, 20, 30][2]` is. So `(4/2) - 3` is an `underflow`, as `2 - 3` is (a
    `uint` less a `uint` below 0), where gp gives -1. The same holds for two unsigned rationals:
    `1/2 - 3/2` is an `underflow`, `-1/2 - 3/2` is -2. Built (generation 39).
57. **A `bool` and a number are not compared, also inside the startup functions.**
    `true < 1` and `1 == true` are an `error`, and so is `max(true, 0)`: the comparison is
    in `lib/startup.rud`, where the message points. Arithmetic casts: `true + 1` is 2 and
    `sum([true, true])` is 2. Built (generation 29).
58. **`defer` evaluates its call's arguments where the `defer` is met.**
    `mut my(a= 1); { defer(say("defer ", a)); a= 2; say("body ", a) };` writes `body 2`, then
    `defer 1` (the "Go rule", spec §8; Perl's `defer { … }` would write `defer 2`). A block
    is read at the exit: `defer({ say("defer ", a) })` writes `defer 2`. So `exit_status()`
    belongs in a block: `defer(say(exit_status()))` is a compile error that says so. Built
    (generation 32).
59. **`%[[:alpha:]]` is not Perl's class.** It is a set with an inner bracket that holds the
    bytes `:`, `a`, `l`, `p`, `h`: its normalized form is `%[\: alpha\:]`,
    `unpack(t{%[[:alpha:]]+}, b"alpha:")` is `[b"alpha:"]`, and `xyz` is refused, with the
    compile warning "'\[:alpha:\]' is POSIX's class, not a set of its bytes: the pattern is &alpha,
    in a set %\[&alpha\]" (generation 40, for the 14 names). The letters are
    `&alpha` (`&alpha+` reads `xyz`), inside a set `%[&alpha _]`. Built.
60. **An end mark that is missing takes the rest of the data.** `a["\x{0}"]` reads up to
    a zero byte and takes it: `unpack(t{a["\x{0}"] x*}, b"ab\x{00}c")` is
    `[b"ab\x{00}",b"c"]`. On `abc`, where no zero byte comes, `unpack(t{a["\x{0}"]}, b"abc")`
    is `[b"abc"]`, no `error`. Built (generation 29).
61. **A run-time error of an operator says what and why.** `1/0` ends the program with
    `we exited <error("1 / 0: division by zero")> at FILE:1:1`, `"a" ~ 1` with `"a" ~ 1: no '~' for a str and a uint`; a function too: `uint(-1)` ends with `uint(-1): -1 is negative`.
    The message is the operation with its values, as rud1 writes them, then why; a value
    longer than 40 characters by its kind and size (`[…; 1000 elements]`). Built
    (generation 31: why; generation 38: the operation first; before, an operator gave the
    position and no text).
62. **Positions in messages count bytes.** `"é"; zz` gives `1:7 error undeclared 'zz'`,
    `"e"; zz` gives `1:6`: the `é` is two bytes of UTF-8. Inside a template too:
    `unpack(t{"é" C}, b"\x{c3}\x{a9}")` reports its `C` at `template 1:6`, with `"e"` at
    `1:5`. Built; the spec does not name the unit.

The gp side at large (0-based lists, `my(r)` without a value, `quit`) is the `pari2rud1`
manual's.

## From generation 37

63. **A function that reads the clock or the input is marked `impure`.** Functions are
    pure unless marked, so `my(f()= len(today())); f()` is a compile error, and
    `impure my(f()= len(today()))` is the way; the top level too (`impure;` as its first part,
    or the policy `policy.main.impure`). Perl and gp have no such mark. Designed, not built:
    today it gives `10` (run).
64. **`say` alone does not make a function impure.** `say` is pure while nothing depends on
    whether its write worked, so `my(f(n)= (say(n); n^2)); f(3)` needs no mark (it writes
    `3` and shows `9`, run); `guard(say(n), alt)` does need it. The lines are written later,
    before any read. Designed.
65. **Perl's END block is no `defer` at the top level.** `defer(say("end")); 5` is
    `1:1 error 'defer' outside a block` (run); `{ defer(say("end")); 5 }` writes `end` and
    shows `5` (run). Built. Designed in generation 37: `at.main_end` (and
    `at.main_start` for checks before `main()`), one list, ends run backward.
