# Introduction to rud1

rud1 is a mostly functional programming language inspired by pari/gp, Perl and C++. Its
prototype is written in Lean; ports to C++, Rust and the JVM are to follow it. The spec states the aim as making
things "easy for the coder, without giving up quality or provability", and as much as
possible is decided when a program is compiled: "whatever can feasibly be solved (or proved)
at compile time is". The project's domain is rud1-lang.org.

This document covers what is **built**: every example was run on the prototype as it is
written here, and the output shown is what came out. Where a thing is **decided** (designed,
waiting for a build) or **proposed** (not confirmed by the author), it says so.

## Running code

`rud1 'code'` runs a one-liner, `rud1 file.rud` a file. `rud1` is a small shell
function around the executable, which is named `rud1_run` (`rud1_run -e 'code'`,
`rud1_run file.rud`); source files end in `.rud`.

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

With a final `;`, the usual form of a script, only the output is left:

```
[rud1] say("hello");  \\ hello
[rud1] 2 + 3 * 4  \\ 14
```

In this document a one-liner is followed by `\\` and what it gave; `\\` starts a comment
that runs to the end of the line, so each example line is valid code. The main form of a
comment is `/* … */`. White space and line ends carry no meaning.

## Numbers

Whole numbers have no size limit. A literal without a sign is a `uint` (natural number), a
negative number a `sint`; `nat`, `int` and `rat` are short aliases of `uint`, `sint`
and `urat` (built). `/` is exact and gives a `urat` (a fraction), or a `srat` with a signed
operand, when the division does not come out; `//` is the whole quotient and `%` the remainder, which is never negative.
`^` is the power.

```
[rud1] [7 / 2, 7 // 2, 7 % 2, 12 / 4, 1/3 + 1/6]  \\ [7/2,3,1,3,1/2]
[rud1] [-7 // 2, -7 % 2, 7 // -2, 7 % -2]  \\ [-4,1,-3,1]
[rud1] [2^100, 2^-3, -3^2]  \\ [1267650600228229401496703205376,1/8,-9]
```

A literal with a dot is a float (binary, 64 bits, in the prototype; a default float of
infinite precision is decided, not built); a digit is needed on both sides of the
dot (`1.0`, not `1.`). A float shows all its 15 digits. An operation with one float operand
gives a float.

```
[rud1] [1.5 + 1, 7 / 2.0, 1e3]  \\ [2.50000000000000,3.50000000000000,1000.00000000000]
[rud1] [0.1 + 0.2 == 0.3, 1/10 + 2/10 == 3/10]  \\ [0,1]
```

A comparison gives a `bool`, shown as `1` or `0`. `==` compares values across the kinds of
number, `===` asks for the same kind as well. Comparisons chain. There is no `&&` and no
`||`: `and` and `or` are functions that stop at the first argument that decides, and `!` is
"not". False are `0`, `0.0` and `false`; an empty list and an empty string are true.

```
[rud1] [1 == 1.0, 1 === 1.0, 1 < 2 <= 2, 1 <=> 2, and(1, 0), or(0, 3), !0]  \\ [1,0,1,-1,0,1,1]
```

A `uint` minus a larger `uint` is not a negative number: where the value leaves the expression
it is the state `underflow` ("Errors are states" below tells what that means), while inside
it the sign is held back, so `3 - 5 + 4` is `2`. A sum with a negative number, or the cast `sint(…)`, gives the
`sint`:

```
[rud1] [3 + -5, sint(3) - 5]  \\ [-2,-2]
[rud1] 3 - 5  \\ we exited <underflow> at 011.rud:1:1
```

A number never becomes text by itself, nor text a number: the casts are `str(x)`, `uint(s)`
and `sint(s)`.

```
[rud1] [str(42), uint("42"), sint("-7"), "n = " ~ str(5)]  \\ ["42",42,-7,"n = 5"]
```

## Names

`my(…)` declares names. A name is immutable unless it is declared with `mut my(…)`; an
assignment to an immutable name is refused when the program is compiled. The initializers
run in order, so a later one sees the earlier names.

```
[rud1] my(a= 3, b= a + 1); a * b  \\ 12
[rud1] my(x= 3); x= 4  \\ 014.rud:1:11 error assignment to immutable 'x'
[rud1] mut my(x= 3); x+= 4; x*= 2; x  \\ 14
```

The assignment forms are `=`, the compound ones (`+=`, `-=`, `*=`, `/=`, `//=`, `%=`, `^=`,
`~=`, `~*=`), an element (`v[i]= e`), a field (`p.x= e`) and a list of names. An assignment
is an expression, with the assigned value as its value. The layout used here, no space
before the `=` and one after, is the style of the language, not a rule of the parser.
`{ … }` is a block: a sequence with its own names, whose value is that of its last
expression.

```
[rud1] mut my(a= 1, b= 2); [a, b]= [b, a]; [a, b]  \\ [2,1]
[rud1] { my(a= 2); a * 3 } + 1  \\ 7
```

## Functions

A function is declared in `my(…)` as `name(parameters)= body`. A lambda is `x -> body` or
`(a, b) -> body`, a value like any other. A function can call itself, and functions that
follow one another in a `my(…)` can call each other.

```
[rud1] my(sq(x)= x^2, cube= x -> x^3); [sq(5), cube(2)]  \\ [25,8]
[rud1] my(add= (a, b) -> a + b); add(1, 2)  \\ 3
[rud1] my(fact(n)= if(n < 2, 1, n * fact(n - 1))); fact(30)  \\ 265252859812191058636308480000000
```

A parameter can have a default, which sees the parameters before it. An empty argument
takes the default. An argument can be given by the parameter's name, `name: value`.

```
[rud1]
my(box(text, left= "[", right= left)= left ~ text ~ right);
[box("a"), box("a", "<"), box("a", , ">"), box("a", right: ")")]  \\ ["[a[","<a<","[a>","[a)"]
```

A rest parameter `v[...]` takes the arguments that are left, as a list; `L[...]` in a call
spreads a list over the arguments. A longer `my(…)` is laid out with leading commas, one
item per line.

```
[rud1]
my
( count(first, v[...])= [first, #v, v]
, L= [3, 4]
);
[count(1), count(1, 2, L[...])]  \\ [[1,0,[]],[1,3,[2,3,4]]]
```

A function that takes a function is called with a lambda, with a placeholder (`_1` is the
first argument, `_2` the second) or with an operator's name (`_+_`, `_*_`). The startup
functions `fold`, `max`, `min`, `prod`, `scan` and `sum` work so. A lambda takes the value
a name has when the lambda is made, and it cannot assign a `mut` of the code around it.

```
[rud1] my(v= [3, 1, 2]); [fold(_*_, v), fold((a, b) -> a + b, v), scan(_+_, v)]  \\ [6,6,[3,4,6]]
[rud1] my(v= [3, 1, 2]); [sum(v), sum(v, _1^2), sum(v, x -> 2 * x), max(v)]  \\ [6,14,12,3]
[rud1] mut my(n= 1); my(f= () -> n); n= 2; [f(), n]  \\ [1,2]
```

## Conditions and loops

`if` is an expression. `if(c, a, b)` is `a` when `c` is true, else `b`. With more arguments
it is a chain: condition, value, condition, value, and an odd last argument is the "else".
Each part can be a sequence with `;`.

```
[rud1]
my
( fizzbuzz(n)= if
  ( 0 == n % 15, "FizzBuzz"
  , 0 == n %  5, "Buzz"
  , 0 == n %  3, "Fizz"
  , n
  )
);
[fizzbuzz(i) | i <- [9..15]]  \\ ["Fizz","Buzz",11,"Fizz",13,14,"FizzBuzz"]
```

An `if` without an "else" whose condition is false has no value: its result is `undef`
("Errors are states").

`while(c, body)` runs `body` as long as `c` is true; `until(c, body)` runs it and then
tests. `for(i=a, b, body)` counts from `a` to `b`, both included; `for(x <- L, body)` walks a
list. The counter is immutable in the body (`for(mut i=a, b, …)` makes it changeable).

```
[rud1]
my(step(n)= if(n % 2, 3 * n + 1, n // 2));
mut my(n= 27, k= 0);
while
( 1 != n
, n= step(n);
  k+= 1
);
k  \\ 111
```

`leave()` ends the innermost loop or block, `leave(, v)` ends it with the value `v`,
`leave(2)` ends two; `next()` goes to the next round.

```
[rud1] mut my(r); for(i=1, 9, if(3 == i, next()); if(6 == i, leave()); r~= [i]); r  \\ [1,2,4,5]
```

`r` has no initial value: a `mut` that is first used in `r~= […]` starts as the empty list
(with `+=` as 0, with `*=` as 1).

A loop has a value: that of its last round, or what `leave(, v)` gives it, and `undef` when
no round ran. A script that ends in a loop, or in an `if` without "else", can so end without
a value, which is a failure ("Errors are states"). The compiler warns on every run:

```
[rud1]
for(i=1, 99, if(i^2 > 200, leave(, i)))
```

Output:

```
029.rud:1:1 warning: the script ends in a 'for', which can be undef (exit status 1): end it with '0;'
15
```

End such a script with `0;`, as the examples here do.

## Lists

A list is written `[a, b, c]` and can hold anything. `#v` is its length, `~` joins two
lists, `~*` repeats one. `[a..b]` is the range from `a` to `b`, both included, and it is
empty when `b` is below `a`. A step follows a `;`: `[a..b;s]`.

```
[rud1] [[1..5], [1..10;3], [10..1;-3], [5..1]]  \\ [[1,2,3,4,5],[1,4,7,10],[10,7,4,1],[]]
[rud1] [#[7, 8, 9], [1, 2] ~ [3], [0] ~* 3]  \\ [3,[1,2,3],[0,0,0]]
```

A comprehension builds a list from another: `[expression | name <- list, condition]`.
Several generators stand in an inner pair of brackets. A list is lazy, so a range may run
to `oo` when only a part of it is used.

```
[rud1] [x^2 | x <- [1..10], 0 == x % 2]  \\ [4,16,36,64,100]
[rud1] [[i, j] | [i <- [1..3], j <- [i..3]]]  \\ [[1,1],[1,2],[1,3],[2,2],[2,3],[3,3]]
[rud1] my(squares= [n^2 | n <- [1..oo]]); squares[0..4]  \\ [1,4,9,16,25]
```

Positions count from 0; a literal negative one counts from the end (`v[-1]` is `v[#__ - 1]`,
where `__` is the list being indexed), a computed one below 0 is an `error`. An index can
be a range or a
list of positions, and then gives a list. A position outside the list is an `error`.

```
[rud1]
my(v= [10, 20, 30, 40, 50]);
[v[0], v[-1], v[1..3], v[0, -1], v[#v-1..0;-1]]  \\ [10,50,[20,30,40],[10,50],[50,40,30,20,10]]
```

Assigning an element needs a `mut` name, and cannot make the list longer; `~=` does that.
A list is a value: a copy does not change with the original.

```
[rud1] mut my(v= [1, 2, 3]); my(w= v); v[0]= 9; v~= [4]; [v, w]  \\ [[9,2,3,4],[1,2,3]]
```

An element can have a name; a list of named elements serves as a record. A field is read
and assigned as `p.name`, and is still reached by its position. A matrix is a list of rows,
written with `;` between the rows.

```
[rud1] my(p= [name: "Ada", born: 1815]); [p.name, p[1], names(p)]  \\ ["Ada",1815,["name","born"]]
[rud1] mut my(p= [x: 1, y: 2]); p.x+= 10; p  \\ [x:11,y:2]
[rud1] my(M= [1, 2, 3; 4, 5, 6]); [M[1][2], M[1], [r[0] | r <- M]]  \\ [6,[4,5,6],[1,4]]
```

## Strings and bytes

A `str` is text, always well-formed UTF-8, written between double quotes with the escapes
`\n \t \r \e \" \\` and `\u{…}` for a code point in hex. `~` joins, `~*` repeats, `#s` is the
length in code points, and the comparisons compare by code point.

```
[rud1] my(s= "héllo"); [#s, s ~ " world", "" != s]  \\ [5,"héllo world",1]
[rud1] ["ab" ~* 3, "abc" < "abd"]  \\ ["ababab",1]
```

A string has no index. The functions on strings carry the type's name in front, the `str.`
family:

```
[rud1] [str.split("a,b,c", ","), str.join(["a", "b"], "-")]  \\ [["a","b","c"],"a-b"]
[rud1] [str.substr("héllo", 1, 3), str.substr("héllo", -2)]  \\ ["éll","lo"]
[rud1] [str.to_characters("hé"), str.to_codepoints("hé")]  \\ [["h","é"],[104,233]]
[rud1] [str.from_codepoints([104, 233]), str.to_titlecase_first("ada")]  \\ ["hé","Ada"]
[rud1] [str.to_uppercase("straße"), str.to_lowercase("ABC")]  \\ ["STRASSE","abc"]
[rud1] str.join([str(x) | x <- [1, 2, 3]], ", ")  \\ "1, 2, 3"
```

The family, in full: `str.eq_caseless`, `str.from_codepoints`, `str.is_cased`,
`str.is_lowercase`, `str.is_uppercase`, `str.join`, `str.split`, `str.substr`,
`str.to_casefold`, `str.to_characters`, `str.to_codepoints`, `str.to_lowercase`,
`str.to_lowercase_first`, `str.to_titlecase_first`, `str.to_uppercase`,
`str.unicode_version`. Three more names exist and give an `error` that says "not built
yet": `str.is_casefolded`, `str.is_titlecase`, `str.to_titlecase`. A formatted print
(`printf`, `str.printf`) is not built.

`bytes` is the type of raw bytes, written `b"…"`, with `\x{89}` for a byte in hex. `#b` counts
bytes, `b[i]` is a byte as a number, and `str(b)` is the text when the bytes are well-formed
UTF-8 (an `error` when they are not). A `str` and a `bytes` value do not mix under `~` or a
comparison.

```
[rud1] my(b= b"\x{c3}\x{a9}t"); [b, #b, b[0], b[-1]]  \\ [b"\x{c3}\x{a9}t",3,195,116]
[rud1] my(b= b"\x{c3}\x{a9}t"); [str(b), #str(b), b ~ b"!"]  \\ ["ét",2,b"\x{c3}\x{a9}t!"]
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

## Errors are states

Every result has a state. A value is a result in the state `def`. Two other states matter
from the first day:

- `undef`: there is no value. An `if` without "else" whose condition was false, a loop
  without a round, `max([])`, `read_line()` at the end of the input.
- `underflow`: a `uint` below 0 where the value leaves the expression (`3 - 5`).
- `error`: something went wrong. `1/0`, a position outside a list, a text
  that is no number in `uint(s)`, and `error("message")` in your own code.

A state is no value: there is no literal for one and no `==` against one. An `error` ends
the sequence, the loop and the call it occurs in, and so travels outward until something
catches it. `guard(x, fallback)` is `x` when `x` is defined, else `fallback`, which is not
evaluated before it is needed. `is_defined(x)` tells whether a result is there, for the
states below `error`; an `error` passes through it.

```
[rud1] my(half(n)= if(0 == n % 2, n // 2)); [half(10), guard(half(7), "odd")]  \\ [5,"odd"]
[rud1] my(v= [1, 2, 3]); [guard(v[5], -1), guard(1/0, 0), guard(uint("12x"), 0)]  \\ [-1,0,0]
[rud1] [is_defined(max([])), guard(is_defined(1/0), "an error")]  \\ [0,"an error"]
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
we exited <error("no such item: milk")> at 054.rud:2:37
  in price at 054.rud:3:53
  in total at 054.rud:6:5
```

The exit status of that run is 1. `exit(n)` ends the program at once with status `n` (0 to
255), and `exit(n, message)` writes the message to standard error first. This one ends with
status 2:

```
[rud1] my(n= guard(uint("12x"), -1)); if(n < 0, exit(2, "not a number")); n  \\ not a number
```

The other states (`missing`: a `my(r)` without a value, or an argument left out; `inf`;
`overflow`; `pending`) are in the spec, section 8.

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

Files are not built: standard input is all a program reads.

## Templates and `unpack`

Binary data and text formats are read with a template: a description of the data, in a
language of its own, written `t{…}`. `unpack(t, b)` reads the `bytes` value `b` by the
template `t` and gives a list in which each named field is a named element.

A few fixed fields. `n` is a 16-bit number with the high byte first, `C` one byte, `V` a
32-bit number with the low byte first; `name:` in front names the field.

```
[rud1]
my(header= t{id:n flags:C size:V});
unpack(header, b"\x{01}\x{02}\x{07}\x{10}\x{00}\x{00}\x{00}")  \\ [id:258,flags:7,size:16]
```

A text with its length in front. `C/a` is a count in one byte, then that many bytes of
text; the count is part of the result. A quoted text has to be there as it is written, and
`x*` skips what is left. A text field is `bytes`, to be cast when it is text.

```
[rud1]
my(r= unpack(t{"ID" name:C/a age:C x*}, b"ID\x{03}Ada\x{24}rest"));
say(r);
say(str(r.name), " is ", r.age);
```

Output:

```
[b"ID",3,name:b"Ada",age:36,b"rest"]
Ada is 36
```

`unpack` has to cover its data, and the data has to cover the template: bytes left over,
or data that ends too early, are an `error` whose message says where.

The template language is large: repeats, counts taken from earlier fields, alternatives,
sets of bytes, patterns that call each other, text and caseless groups. The spec's section
5d describes it, and marks what is built. `pack(t, r)` is the way back from a list to bytes (it
reads back what it wrote, and refuses bytes that would read as another list), and
`unpack(t)` reads standard input.

## Where to go next

| Document | Content |
|---|---|
| `rud1_differences.md` | rud1 beside other languages |
| `rud1_examples.md` | longer programs |
| `rud1_gotchas.md` | what might surprise a coder who comes from Perl or gp |
| `rud1_not_yet.md` | what is decided or proposed and not yet built |
| `rud1_spec.md` | the spec, the authority on every rule (a draft, not published yet) |
