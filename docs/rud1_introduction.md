# Introduction to rud1

rud1 is a mostly functional programming language. This document introduces its first
release: what it will hold and offer, and how a program in it looks and runs. The author's
summary: "The rud1-language is sound all over, and strict where we must, and fast where we
can." The spec states the aim as making things "easy for the coder, without giving up quality
or provability", and as much as possible is decided when a program is compiled: "whatever can
feasibly be solved (or proved) at compile time is". The project's domain is rud1-lang.org.
Of the summary, soundness is built as far as the prototype reaches, strictness in part
(below), and speed is the aim of the host, not measured yet. The author expects more:
"Auditors will drool. Companies will drool. Developers will feel right at home. AI-tooling
deeply loves it already."

Three words are used with one meaning each, and every feature below carries one of them:

- **built**: the prototype does it.
- **decided**: designed, a rule of the spec; it waits for a build.
- **proposed**: written in the spec, not confirmed by the author.

A feature without a word is built. Every example was run on the prototype as it is written
here, and the output shown is what came out. The prototype is written in Lean, which also
holds the proofs about the language's rules; its executable is named `rud1_run`, and
`rud1` below stands for it.

## The first release

| Part | Status |
|---|---|
| one executable, `rud1`: a one-liner, a file, standard input, and the REPL | proposed; today `rud1_run`, without the REPL |
| C++ as the host language, whole numbers by GMP | proposed (the author: "most likely") |
| the REPL, after gp's: the prompt `? `, every value numbered (`%1`, `%`) | decided |
| the language of the sections below: numbers, names, functions, loops, lists, strings, bytes | built |
| results with a state (`error`, `undef`, `underflow`, …) instead of exceptions | built |
| no silent rounding: a float that would round is an error | built |
| a float of infinite precision as the default float | decided |
| templates: `unpack` and `pack` for binary data and text formats | built |
| a template refused when compiled when reading by it would take time that grows faster than the data | built, the rule proposed |
| functions pure unless the compiler finds an effect | decided |
| status words on declarations: `deprecated_( … )_deprecated` | built |
| regions with conditions and taint: `SOx_( … )_SOx` | proposed, designed in part |
| messages with an id and an explanation: `rud1 --explain` | built |
| policies on the command line: `-p.NAME=N` | built for warnings, `underflow`, `unpack` |
| `sprintf` with Perl's formats, written in rud1 with templates | built, as a library file |
| `use` and modules | proposed |
| files and handles | not designed yet |
| a binding to libpari, as a module | proposed |

## Running code

`rud1 'code'` runs a one-liner, `rud1 file.rud` a file; source files end in `.rud`.

A program is a sequence of expressions separated by `;`. Its value is the value of the last
one, and that value is shown after the program's output, unless the source ends in `;`.
`say` writes its arguments and a newline, and its value is its last argument:

```
[rud1]
say("hello")
```

Output:

```
hello
"hello"
```

With a final `;`, the usual form of a script, only the output is left.

Most examples below are sessions, as the REPL will show them (decided): an entry follows the
prompt `? `, a longer one goes on in lines indented by two, and its value follows `> `. An
entry that ends in `;` shows no value. The REPL numbers each value as well (`%1 = …`,
after gp; decided, not built); the sessions here leave the numbers out.

```
? 2 + 3 * 4
> 14

? say("hello");
```

`\\` starts a comment that runs to the end of the line; the main form of a comment is
`/* … */`. White space and line ends carry no meaning.

## Numbers

Whole numbers have no size limit. A literal without a sign is a `uint` (natural number), a
negative number a `sint`; `nat`, `int` and `rat` are short aliases of `uint`, `sint`
and `urat`. `/` is exact and gives a fraction when the division does not come out;
`//` is the whole quotient and `%` the remainder, which is never negative. `^` is the
power.

```
? [7 / 2, 7 // 2, 7 % 2, 12 / 4, 1/3 + 1/6]
> [7/2,3,1,3,1/2]

? [-7 // 2, -7 % 2, 7 // -2, 7 % -2]
> [-4,1,-3,1]

? [2^100, 2^-3, -3^2]
> [1267650600228229401496703205376,1/8,-9]
```

Sound all over: a value is what it says, or there is none. A `uint` minus a larger `uint`
does not silently become a negative number, which protects an index; the result is the state
`underflow` ("Errors are states"). A sum with a negative number, a prefix `+`, or the cast
`sint(…)` gives the `sint`:

```
? [3 + -5, sint(3) - 5, +3 - 5]
> [-2,-2,-2]

? guard(3 - 5, "below 0")
> "below 0"
```

A literal with a dot is a float; a digit is needed on both sides of the dot (`1.0`, not
`1.`). An operation with one float operand gives a float. The default float of the release
has infinite precision (decided). The prototype's float is binary, 64 bits, and where that
would round it stops instead of giving a value rud1 would not give: `0.1` is an error there,
`1/10` is exact. Without a dot a literal stays exact: `1e3` is the whole number 1000,
`2e-3` the rational 1/500. A concession is made only at the interface of a library from outside,
and tightened there as far as it goes.

```
? [1.5 + 1, 7 / 2.0, 1.0e3, 1e3, 2e-3]
> [2.50000000000000,3.50000000000000,1000.00000000000,1000,1/500]

? [0.25 + 0.5 == 0.75, 1/10 + 2/10 == 3/10]
> [1,1]
```

```
[rud1]
0.1 + 0.2
```

Output:

```
006.rud:1:1 error 0.1: the prototype's float (binary64) holds no exact 1/10 [E0f28e9]
```

A comparison gives a `bool`, shown as `1` or `0`. `==` compares values across the kinds of
number, `===` asks for the same kind as well. Comparisons chain. There is no `&&` and no
`||`: `and` and `or` stop at the first argument that decides, and `!` is "not". False
are `0`, `0.0` and `false`; an empty list and an empty string are true.

```
? [1 == 1.0, 1 === 1.0, 1 < 2 <= 2, 1 <=> 2, and(true, false), or(false, true), !false]
> [1,0,1,-1,0,1,1]
```

A number never becomes text by itself, nor text a number: the casts are `str(x)`, `uint(s)`
and `sint(s)`.

```
? [str(42), uint("42"), sint("-7"), "n = " ~ str(5)]
> ["42",42,-7,"n = 5"]
```

## Names

`my(…)` declares names. A name is immutable unless it is declared with `mut my(…)`; an
assignment to an immutable name is refused when the program is compiled. The initializers
run in order, so a later one sees the earlier names.

```
? my(a= 3, b= a + 1); a * b
> 12

? mut my(x= 3); x+= 4; x*= 2; x
> 14
```

```
[rud1]
my(x= 3); x= 4
```

Output:

```
010.rud:1:11 error assignment to immutable 'x'
```

The assignment forms are `=`, the compound ones (`+=`, `-=`, `*=`, `/=`, `//=`, `%=`, `^=`,
`~=`, `~*=`), an element (`v[i]= e`), a field (`p.x= e`) and a list of names. An assignment
is an expression, with the assigned value as its value. The layout used here, no space
before the `=` and one after, is the style of the language, not a rule of the parser.
`{ … }` is a block: a sequence with its own names, whose value is that of its last
expression.

```
? mut my(a= 1, b= 2); [a, b]= [b, a]; [a, b]
> [2,1]

? { my(a= 2); a * 3 } + 1
> 7
```

## Functions

A function is declared in `my(…)` as `name(parameters)= body`. A lambda is `x -> body` or
`(a, b) -> body`, a value like any other. A function calls itself as `__SELF__(…)`, and the functions
of one `my(…)` call each other by name.

```
? my(sq(x)= x^2, cube= x -> x^3, add= (a, b) -> a + b);
? [sq(5), cube(2), add(1, 2)]
> [25,8,3]

? my(fact(n)= if(n < 2, 1, n * __SELF__(n - 1)));
? fact(30)
> 265252859812191058636308480000000
```

A parameter can have a default, which sees the parameters before it. An empty argument
takes the default. An argument can be given by the parameter's name, `name: value`.

```
? my(box(text, left= "[", right= left)= left ~ text ~ right);
? [box("a"), box("a", "<"), box("a", , ">"), box("a", right: ")")]
> ["[a[","<a<","[a>","[a)"]
```

A rest parameter `v[...]` takes the arguments that are left, as a list; `L[...]` in a call
spreads a list over the arguments. A longer `my(…)` is laid out with leading commas, one
item per line.

```
? my
  ( count(first, v[...])= [first, #v, v]
  , L= [3, 4]
  );
? [count(1), count(1, 2, L[...])]
> [[1,0,[]],[1,3,[2,3,4]]]
```

A function that takes a function is called with a lambda, with a placeholder (`_1` is the
first argument, `_2` the second) or with an operator's name (`_+_`, `_*_`). The startup
functions `fold`, `max`, `min`, `prod`, `scan` and `sum` work so. A lambda takes the value
a name has when the lambda is made, and it cannot assign a `mut` of the code around it.

```
? my(v= [3, 1, 2]);
? [fold(_*_, v), fold((a, b) -> a + b, v), scan(_+_, v)]
> [6,6,[3,4,6]]

? [sum(v), sum(v, _1^2), sum(v, x -> 2 * x), max(v)]
> [6,14,12,3]

? mut my(n= 1); my(f= () -> n); n= 2; [f(), n]
> [1,2]
```

Functions are pure unless they have an effect (decided, not built): a pure function gives the
same result for the same arguments, and cannot call an `impure` one; reading the clock or
standard input is `impure`. The compiler finds which functions, loops and modules are
impure, also through a `use` of an impure module, so the mark `impure` is needed only on code
it cannot look into, such as a foreign function. Pure code may be folded at compile time and run
in parallel.

## Conditions and loops

`if` is an expression. `if(c, a, b)` is `a` when `c` is true, else `b`. With more arguments
it is a chain: condition, value, condition, value, and an odd last argument is the "else".
Each part can be a sequence with `;`.

```
? my
  ( fizzbuzz(n)= if
    ( 0 == n % 15, "FizzBuzz"
    , 0 == n %  5, "Buzz"
    , 0 == n %  3, "Fizz"
    , n
    )
  );
? [fizzbuzz(i) | i <- [9..15]]
> ["Fizz","Buzz",11,"Fizz",13,14,"FizzBuzz"]
```

An `if` without an "else" whose condition is false has no value: its result is `undef`
("Errors are states").

`while(c, body)` runs `body` as long as `c` is true; `until(c, body)` runs it and then
tests. `for(i= a, b, body)` counts from `a` to `b`, both included; `for(x <- L, body)` walks a
list. The counter is immutable in the body (`for(mut i= a, b, …)` makes it changeable).

```
? my(step(n)= if(0 != n % 2, 3 * n + 1, n // 2));
? mut my(n= 27, k= 0);
? while
  ( 1 != n
  , n= step(n);
    k+= 1
  );
? k
> 111
```

`leave()` ends the innermost loop or block, `leave(, v)` ends it with the value `v`,
`leave(2)` ends two; `next()` goes to the next round.

```
? mut my(r= []); for(i= 1, 9, if(3 == i, next()); if(6 == i, leave()); r~= [i]); r
> [1,2,4,5]
```

`r` gets its value, and so its kind, at its declaration: a compound assignment such as
`r~= […]` to a `mut` that has no value yet is a compile error.

A loop has a value: that of its last round, or what `leave(, v)` gives it, and `undef` when
no round ran. A script that ends in a loop, or in an `if` without "else", can so end without
a value, which is a failure ("Errors are states"). The compiler warns on every run:

```
[rud1]
for(i= 1, 99, if(i^2 > 200, leave(, i)))
```

Output:

```
019.rud:1:1 warning: the script ends in a 'for', which can be undef (exit status 1): end it with '0;' [W02b6c2]
15
```

End such a script with `0;`.

## Lists

A list is written `[a, b, c]` and can hold anything. `#v` is its length, `~` joins two
lists, `~*` repeats one. `[a..b]` is the range from `a` to `b`, both included, and it is
empty when `b` is below `a`. A step follows a `;`: `[a..b;s]`. A range written backwards
without a step, `[5..1]`, is a warning: it is empty, and a step of `-1` may have been meant.

```
? [[1..5], [1..10;3], [10..1;-3], [5..1;1]]
> [[1,2,3,4,5],[1,4,7,10],[10,7,4,1],[]]

? [#[7, 8, 9], [1, 2] ~ [3], [0] ~* 3]
> [3,[1,2,3],[0,0,0]]
```

A comprehension builds a list from another: `[expression | name <- list, condition]`.
Several generators stand in an inner pair of brackets. A list is lazy, so a range may run
to `oo` when only a part of it is used.

```
? [x^2 | x <- [1..10], 0 == x % 2]
> [4,16,36,64,100]

? [[i, j] | [i <- [1..3], j <- [i..3]]]
> [[1,1],[1,2],[1,3],[2,2],[2,3],[3,3]]

? my(squares= [n^2 | n <- [1..oo]]); squares[0..4]
> [1,4,9,16,25]
```

Positions count from 0; a literal negative one counts from the end (`v[-1]` is `v[#__ - 1]`,
where `__` is the list being indexed), a computed one below 0 is an `error`. An index can
be a range or a list of positions, and then gives a list. A position outside the list is an
`error`.

```
? my(v= [10, 20, 30, 40, 50]);
? [v[0], v[-1], v[1..3], v[0, -1], v[#v-1..0;-1]]
> [10,50,[20,30,40],[10,50],[50,40,30,20,10]]
```

Assigning an element needs a `mut` name, and cannot make the list longer; `~=` does that.
A list is a value: a copy does not change with the original.

```
? mut my(v= [1, 2, 3]); my(w= v); v[0]= 9; v~= [4]; [v, w]
> [[9,2,3,4],[1,2,3]]
```

An element can have a name; a list of named elements serves as a record. A field is read
and assigned as `p.name`, and is still reached by its position. A matrix is a list of rows,
written with `;` between the rows.

```
? my(p= [name: "Ada", born: 1815]); [p.name, p[1], names(p)]
> ["Ada",1815,["name","born"]]

? mut my(q= [x: 1, y: 2]); q.x+= 10; q
> [x:11,y:2]

? my(M= [1, 2, 3; 4, 5, 6]); [M[1][2], M[1], [r[0] | r <- M]]
> [6,[4,5,6],[1,4]]
```

## Strings and bytes

A `str` is text, always well-formed UTF-8, written between double quotes with the escapes
`\n \t \r \e \" \\` and `\u{…}` for a code point in hex. `~` joins, `~*` repeats, `#s` is the
length in code points, and the comparisons compare by code point.

```
? my(s= "héllo"); [#s, s ~ " world", "" != s]
> [5,"héllo world",1]

? ["ab" ~* 3, "abc" < "abd"]
> ["ababab",1]
```

A string has no index. The functions on strings carry the type's name in front, the `str.`
family:

```
? [str.split("a,b,c", ","), str.join(["a", "b"], "-")]
> [["a","b","c"],"a-b"]

? [str.substr("héllo", 1, 3), str.substr("héllo", -2)]
> ["éll","lo"]

? [str.to_characters("hé"), str.to_codepoints("hé")]
> [["h","é"],[104,233]]

? [str.from_codepoints([104, 233]), str.to_titlecase_first("ada")]
> ["hé","Ada"]

? [str.to_uppercase("straße"), str.to_lowercase("ABC")]
> ["STRASSE","abc"]
```

The family, in full: `str.eq_caseless`, `str.from_codepoints`, `str.is_cased`,
`str.is_lowercase`, `str.is_uppercase`, `str.join`, `str.split`, `str.substr`,
`str.to_casefold`, `str.to_characters`, `str.to_codepoints`, `str.to_lowercase`,
`str.to_lowercase_first`, `str.to_titlecase_first`, `str.to_uppercase`,
`str.unicode_version`. Three more names exist and give an `error` that says "not built
yet": `str.is_casefolded`, `str.is_titlecase`, `str.to_titlecase`.

`bytes` is the type of raw bytes, written `b"…"`, with `\x{89}` for a byte in hex. `#b` counts
bytes, `b[i]` is a byte as a number, `b[1..2]` is a `bytes` value of two, and `str(b)` is the text when the bytes are well-formed
UTF-8 (an `error` when they are not). A `str` and a `bytes` value do not mix under `~` or a
comparison.

```
? my(b= b"\x{c3}\x{a9}t");
? [b, #b, b[0], b[-1]]
> [b"\x{c3}\x{a9}t",3,195,116]

? [str(b), #str(b), b ~ b"!"]
> ["ét",2,b"\x{c3}\x{a9}t!"]
```

## Printing

`say(…)` writes the texts of its arguments without anything between them, then a newline;
`print(…)` does the same without the newline. A string is written as it is, any other value
as it is shown. Both give their last argument as their value.

```
[rud1]
my(total= 3, parts= [1, 1/2, "x"]);
print("total: ");
say(total, " items, ", parts);
say("half: ", total / 2, ", as a float: ", total / 2.0);
```

Output:

```
total: 3 items, [1,1/2,"x"]
half: 3/2, as a float: 1.50000000000000
```

`sprintf(format, values)` writes Perl's formats (`%d`, `%-5s`, `%#6x`, `%.3f`, …). It is
written in rud1 itself: it takes the format apart with a template and writes the values with
another, and it is checked against Perl's own `sprintf`. Today it is a library file that a
program stands after; with `use` (proposed) it becomes a module. `str.printf`, the same as a
function of the `str` family, is decided.

## Errors are states

Every result has a state. A value is a result in the state `def`. The states met first:

- `undef`: there is no value. An `if` without "else" whose condition was false, a loop
  without a round, `max([])`, `read_line()` at the end of the input.
- `underflow`: a `uint` that would go below 0, or a read that runs short at the end of
  the input.
- `error`: something went wrong. `1/0`, a position outside a list, a text
  that is no number in `uint(s)`, and `error("message")` in your own code.

A state is no value: there is no literal for one and no `==` against one. An `error` ends
the sequence, the loop and the call it occurs in, and so travels outward until something
catches it; the other states are bound, stored and passed on, the worst one winning. There
are no exceptions, and no `return`: `guard(x, fallback)` is `x` when `x` is defined, else
`fallback`, which is not evaluated before it is needed. `is_defined(x)` tells whether a result
is there, for the states below `error`; an `error` passes through it.

```
? my(half(n)= if(0 == n % 2, n // 2)); [half(10), guard(half(7), "odd")]
> [5,"odd"]

? my(v= [1, 2, 3]); [guard(v[5], -1), guard(1/0, 0), guard(uint("12x"), 0)]
> [-1,0,0]

? [is_defined(max([])), guard(is_defined(1/0), "an error")]
> [0,"an error"]
```

A program whose last value is not defined ends with the dying message on standard error,
`we exited <state> at file:line:column`, with the calls that led there, and with exit status 1.
Nothing is shown as a result. Otherwise the status is 0.

```
[rud1]
my
( price(item)= if("tea" == item, 3, error("no such item: " ~ item))
, total(items)= { mut my(t= 0); for(i <- items, t+= price(i)); t }
);
say(total(["tea", "tea"]));
say(total(["tea", "milk"]));
say("not reached");
```

Output:

```
6
we exited <error("no such item: milk")> at 030.rud:2:37
  in (price at 030.rud:3:53, total at 030.rud:6:5)
```

The exit status of that run is 1. `exit(n)` ends the program at once with status `n` (0 to
255), and `exit(n, message)` writes the message to standard error first. This one ends with
status 2:

```
[rud1]
my(n= guard(uint("12x"), -1)); if(n < 0, exit(2, "not a number")); n
```

Output:

```
not a number
```

The spec's section 8 has every state (also `missing`, `inf`, `overflow` and `pending`), and
an overview of which object can carry which state and how each spreads.

## Reading

`read_line()` gives the next line of standard input as a `str`, its line feed included, and
`undef` when the input has ended. A program takes the line feed off and converts the text
itself. With the input `12 30`, a line feed, `7 -8 1` and a line feed:

```
[rud1]
my(numbers(s)= [sint(w) | w <- str.split(str.split(s, "\n")[0], " "), "" != w]);
mut my(line= read_line());
while
( is_defined(line)
, say(numbers(line), " sum ", sum(numbers(line)));
  line= read_line()
);
0;
```

Output:

```
[12,30] sum 42
[7,-8,1] sum 0
```

`unpack(t)` reads standard input by a template (next section). Files and handles are not
designed yet: standard input is all a program reads.

## Templates and `unpack`

Binary data and text formats are read with a template: a description of the data, in a
language of its own, written `t{…}`. `unpack(t, b)` reads the `bytes` value `b` by the
template `t` and gives a list in which each named field is a named element. In the
author's words, the templates are "a language-within-a-language, PCRE-on-steroids", with no
regular expression underneath.

A few fixed fields. `n` is a 16-bit number with the high byte first, `C` one byte, `V` a
32-bit number with the low byte first; `name:` in front names the field.

```
? my(tpl_header= t{id:n flags:C size:V});
? unpack(tpl_header, b"\x{01}\x{02}\x{07}\x{10}\x{00}\x{00}\x{00}")
> [id:258,flags:7,size:16]
```

A text with its length in front. `C/a` is a count in one byte, then that many bytes of
text; the count is part of the result. A quoted text has to be there as it is written, and
`x*` skips what is left. A text field is `bytes`, to be cast when it is text.

```
? my(r= unpack(t{"ID" name:C/a age:C x*}, b"ID\x{03}Ada\x{24}rest"));
? r
> [b"ID",3,name:b"Ada",age:36,b"rest"]

? str(r.name) ~ " is " ~ str(r.age)
> "Ada is 36"
```

`unpack` has to cover its data, and the data has to cover the template: bytes left over,
or data that ends too early, are an `error` whose message says where.

The template language is large: repeats, counts taken from earlier fields, alternatives,
sets of bytes, patterns that call each other, text and caseless groups. `pack(t, r)` is the
way back from a list to bytes; it reads back what it wrote, and refuses bytes that would read
as another list. The reader never tries again (it races the variants of an alternative
instead), and a template gives one reading of its data.

Much is checked when the program is compiled. In the author's words, "It refuses hostile
input early": a template that hostile input could make slow is refused when compiled (built;
the rule proposed): a repeat whose rounds each leave a trial
under way that reads on beyond the round would take time that grows faster than the data.

```
[rud1]
unpack(t{("a"* "b" || "a")* "!"}, b"aaa!")
```

Output:

```
035.rud:1:8 error template 1:1: each round of this repeat leaves a trial under way that reads on beyond the round, one more each round, as on "aaaaaaaaaaaa"…: reading would take time that grows faster than the data; let the variant or the look-ahead that reads on end where the round ends
```

Written so that each round needs its own first byte, the same template passes:

```
? unpack(t{("x" ("a"* "b" || "a"))* "!"}, b"xaxab!")
> [[[b"x",[b"a"]],[b"x",[[b"a"],b"b"]]],b"!"]
```

This check covers repeats; a bound for every template, as RE2 gives one for its regular
expressions, is not claimed. For the repeats the check cannot see into, the reader is to
keep a memo of each place and item (decided, not built).

## Status words and regions

A declaration can carry a status word, written as a bracket around the item. A use of a
`deprecated` item is a warning at each place of use, when compiled:

```
[rud1]
my(deprecated_( old(x)= x + 1 )_deprecated); old(2)
```

Output:

```
037.rud:1:46 warning: 'old' is deprecated: its declaration at 1:4 stands in deprecated_( … )_deprecated [W0bad3d]
3
```

A region marks a piece of code that may run only under conditions (proposed, designed in
part): `SOx_( … )_SOx` around the code that has to comply.
When a condition does not hold, the program exits. A value made in the region carries a
taint, and a tainted value used outside such a region exits too. A function that knows
nothing of the region passes the taint on to its result, also through a condition it decides.
Values from a database, a request or a file can come in tainted. The compiler tracks the
taint where it can; where it cannot, the taint travels with the value as a state does.
Another region the author names: `requires_fresh_currency_exchange_rates_( … )`.

## Messages and policies

Every message that has more than one way out carries an id, and `rud1 --explain ID` explains
it:

`rud1 --explain W0bad3d` says what `deprecated` means, with an example and the ways out,
among them the policy that moves the warning out of sight.

A policy is set on the command line as `-p.NAME=N` (`--policy.NAME=N`): the level of a
warning, the warnings shown (`-p.warnings=0` shows none), a trace of each `uint` that went
below 0, the most bytes `unpack(t)` keeps of a pipe.

## Deterministic

The prototype's evaluator is a function in Lean: the same program on the same input gives the
same result and output. A template gives one reading of its data. Decided for the release:
`random()` with a seed repeats its numbers; proposed: sums run in parallel add in a fixed
order, so that floats come out the same. Two things are not fixed, by design (decided):
`random()` without a seed, and the order of the lines that pure code running in parallel
writes.

## Where to go next

| Document | Content |
|---|---|
| `rud1_better_best.md` | good, better, best: rud1's choices beside other languages |
| `rud1_examples.md` | longer programs |
| `rud1_gotchas.md` | what might surprise a coder who comes from Perl or gp |
| `rud1_not_yet.md` | what is decided or proposed and not yet built |
| `rud1_spec.md` | the spec, the authority on every rule (a draft, not published yet) |
