# rud1 examples

A cookbook for rud1 (site to come: `rud1-lang.org`): programs that run on today's prototype,
each with the output it gave. Everything shown is **built**; what is decided and not built is
named in one line where a reader would look for it, and never shown.

Running: `rud1 'code'` or `rud1 file.rud`, where `rud1` is a shell function of yours around
the executable, which is named `rud1_run` (`rud1_run -e 'code'`, `rud1_run file.rud`);
source files end in `.rud`. The value of the last expression is shown, unless the source ends
in `;`. A one-liner is given as `[rud1] code \\ result`: `\\` starts a comment, so the line
without its `[rud1]` is valid code. A longer program is a block that starts with `[rud1]`,
with its result after `\\` or in a block headed "Output:".

## One-liners

**Arithmetic.** `/` is exact, `//` and `%` are the floored quotient and its remainder.

```
[rud1] 1 + 2 * 3 \\ 7
[rud1] [7 / 2, 7 // 2, 7 % 2, -7 // 2, -7 % 2] \\ [7/2,3,1,-4,1]
[rud1] [2^10, -3^2, 2^3^2] \\ [1024,-9,512]
[rud1] [1 < 2 <= 3, 7 <=> 3, abs(-5)] \\ [1,1,5]
[rud1] [0.1 + 0.2, 3.0 / 4] \\ [0.300000000000000,0.750000000000000]
```

**Big numbers.** Whole numbers have no upper limit.

```
[rud1] 2^100 \\ 1267650600228229401496703205376
```

**Rationals.**

```
[rud1] 1/3 + 1/6 \\ 1/2
[rud1] [(2/3)^3, 2^-3, (1/2)^-2] \\ [8/27,1/8,4]
[rud1] sum([1/k | k <- [1..10]]) \\ 7381/2520
[rud1] [1/3 + 0.5, 0.1 == 1/10] \\ [0.833333333333333,1]
```

**Ranges.** A step follows `;`; a range among other elements is spliced in.

```
[rud1] [1..10] \\ [1,2,3,4,5,6,7,8,9,10]
[rud1] [[1..9;2], [10..1;-3]] \\ [[1,3,5,7,9],[10,7,4,1]]
[rud1] [1, 5..7, 9] \\ [1,5,6,7,9]
[rud1] my(L= [1..oo]); L[0..4] \\ [1,2,3,4,5]
```

**Comprehensions.** Filters follow the generator; several generators stand in a bracket.

```
[rud1] [x^2 | x <- [1..8]] \\ [1,4,9,16,25,36,49,64]
[rud1] [x | x <- [1..30], 0 == x % 7] \\ [7,14,21,28]
[rud1] [[a, b] | [a <- [1..3], b <- [a..3]]] \\ [[1,1],[1,2],[1,3],[2,2],[2,3],[3,3]]
[rud1] [[i * j | j <- [1..4]] | i <- [1..3]] \\ [[1,2,3,4],[2,4,6,8],[3,6,9,12]]
```

**Sums and folds.**

```
[rud1] sum([1..10], _1^2) \\ 385
[rud1] [fold(_+_, [1, 2, 3, 4]), fold(_*_, 1, [1..6])] \\ [10,720]
[rud1] scan(_+_, [1, 2, 3, 4]) \\ [1,3,6,10]
[rud1] [max(3, 9, 4), min([5, 2, 8]), fold(max, [3, 9, 4])] \\ [9,2,9]
```

**Strings.** `~` joins, `~*` repeats, `#` counts code points; `str` gives the text of a number.

```
[rud1] ["abc" ~ "def", "ab" ~* 3, #"héllo"] \\ ["abcdef","ababab",5]
[rud1] [str(42) ~ "!", sint("42") + 1] \\ ["42!",43]
[rud1] str.split("a,b,c", ",") \\ ["a","b","c"]
[rud1] str.join(["a", "b", "c"], "-") \\ "a-b-c"
[rud1] say("x = ", 5); \\ x = 5
```

## Functions

**Recursion.** Functions of one `my(…)` see each other.

```
[rud1] my(fact(n)= if(n < 2, 1, n * fact(n - 1))); fact(30) \\ 265252859812191058636308480000000
[rud1] my(ev(n)= if(0 == n, 1, od(n - 1)), od(n)= if(0 == n, 0, ev(n - 1))); ev(10) \\ 1
```

A body of several steps is a block; `leave(2, 0)` leaves the loop and the block with 0.

```
[rud1]
my
( is_prime(n)=
  { mut my(d= 2);
    while(d * d <= n, if(0 == n % d, leave(2, 0)); d+= 1);
    n > 1
  }
);
[n | n <- [1..40], is_prime(n)] \\ [2,3,5,7,11,13,17,19,23,29,31,37]
```

**Lambdas.**

```
[rud1] my(sq= x -> x^2); [sq(k) | k <- [1..5]] \\ [1,4,9,16,25]
[rud1] my(add= (a, b) -> a + b); add(2, 3) \\ 5
[rud1] my(f= x -> y -> x + y); f(1)(2) \\ 3
[rud1] my(compose(f, g)= x -> f(g(x)), h= compose(x -> x + 1, x -> x * 2)); h(5) \\ 11
```

**Defaults and named arguments.** A default may use the parameters before it; an empty
argument takes the default.

```
[rud1] my(f(a, b= 5, c= a + b)= [a, b, c]); [f(1), f(1, , 9)] \\ [[1,5,6],[1,5,9]]
[rud1] my(area(w, h= w)= w * h); [area(3), area(3, 4), area(h: 2, w: 5)] \\ [9,12,10]
[rud1] my(f(a, b, c)= [a, b, c]); f(c: 9, a: 1, b: 5) \\ [1,5,9]
```

**Rest and spread.** `v[...]` takes the remaining arguments as a list, `v[1...]` at least
one; `L[...]` spreads a list over the parameters.

```
[rud1] my(f(a, v[...])= [a, v]); [f(1), f(1, 2, 3)] \\ [[1,[]],[1,[2,3]]]
[rud1] my(mean(v[1...])= sum(v) / #v); mean(1, 2, 6) \\ 3
[rud1] my(f(a, b, c)= a + b * c, L= [2, 3]); f(1, L[...]) \\ 7
```

**Operators as values.** An operator between two `_` is a function.

```
[rud1] fold(_~_, ["a", "b", "c"]) \\ "abc"
[rud1] [_<=>_(1, 2), _^_(2, 10)] \\ [-1,1024]
```

**Placeholders.** `_1`, `_2` in an argument make that argument a function of them.

```
[rud1] my(twice(f, x)= f(f(x))); twice(_1 * 3, 5) \\ 45
[rud1] fold(_1 + 2 * _2, [1, 2, 3]) \\ 11
```

## Lists and records

**Indexing.** Positions count from 0; a literal negative one counts from the end (`v[-1]` is
`v[#__ - 1]`, `__` the list being indexed); a computed one below 0 is an `error`.

```
[rud1] my(v= [10, 20, 30, 40]); [v[0], v[-1], v[1..2], #v] \\ [10,40,[20,30],4]
[rud1] mut my(v= [1, 2, 3]); v[1]= 9; v~= [4]; v \\ [1,9,3,4]
[rud1] my(M= [1, 2, 3; 4, 5, 6]); [M[1][2], M[0], [r[0] | r <- M]] \\ [6,[1,2,3],[1,4]]
```

**Gather and scatter.** Several positions give a list; assigned to, they take a list.

```
[rud1] my(v= [10, 20, 30, 40]); v[3, 0] \\ [40,10]
[rud1] mut my(v= [1, 2, 3]); v[0, 2]= v[2, 0]; v \\ [3,2,1]
```

**Stepped ranges.** As a list (a list of steps is taken in turn), and as positions: every
second element, the reverse.

```
[rud1] [[0..20;5], [20..0;-5]] \\ [[0,5,10,15,20],[20,15,10,5,0]]
[rud1] my(w= [4, 2, 4, 2, 4, 6, 2, 6]); [7..40;w] \\ [7,11,13,17,19,23,29,31,37]
[rud1] my(v= [0, 1, 2, 3, 4]); [v[0 .. #v - 1; 2], v[#v - 1 .. 0; -1]] \\ [[0,2,4],[4,3,2,1,0]]
```

**Named lists.** A record is a list with names on its elements.

```
[rud1] my(p= [x: 1, y: 2]); [p.x, p[1], #p, names(p)] \\ [1,2,2,["x","y"]]
[rud1] mut my(p= [x: 1, y: 2]); p.x+= 10; p \\ [x:11,y:2]
[rud1] [x: 1] ~ [2, y: 3] \\ [x:1,2,y:3]
[rud1] my(L= [[x: 1, y: 2], [x: 3, y: 4]]); [p.x * p.y | p <- L] \\ [2,12]
```

**Destructuring.**

```
[rud1] my([a, b]= [1, 2]); a + b \\ 3
[rud1] my(divmod(a, b)= [a // b, a % b], [q, r]= divmod(17, 5)); [q, r] \\ [3,2]
[rud1] mut my(a= 1, b= 2); [a, b]= [b, a]; [a, b] \\ [2,1]
```

## Loops

A script that ends in a loop is ended with `0;`: a loop that ran no round has no value.

**`for`.**

```
[rud1] mut my(s= 0); for(i=1, 100, s+= i); s \\ 5050
[rud1] mut my(s= 0); for([a, b] <- [[1, 2], [3, 4]], s+= a * b); s \\ 14
```

```
[rud1]
my(N= 3);
for(i=1, N, say(str.join([str(i * j) | j <- [1..N]], " ")));
0;
```

Output:

```
1 2 3
2 4 6
3 6 9
```

**`while` and `until`.** `until` tests after the body, so its body runs at least once.

```
[rud1] mut my(i= 0, s= 0); while(i < 5, i+= 1; s+= i); [i, s] \\ [5,15]
[rud1] mut my(n= 27, k= 0); while(1 != n, n= if(n % 2, 3 * n + 1, n // 2); k+= 1); k \\ 111
[rud1] mut my(n= 1234); until(n > 1000, n*= 2); n \\ 2468
```

**`leave`.** `leave(, v)` ends the loop with the value `v`; `leave(2, v)` ends two levels.

```
[rud1] my(first(L, p)= for(x <- L, if(p(x), leave(, x)))); first([3, 8, 12, 5], _1 > 10) \\ 12
[rud1] mut my(k= 0); while(1, k+= 1; if(k^2 > 50, leave())); k \\ 8
[rud1] my(f(L)= { for(x <- L, if(x > 1, leave(2, x))); 0 }); [f([1, 2, 3]), f([1, 1])] \\ [2,0]
```

**`next`.** `next()` goes to the next round, `next(2)` to the next round of the outer loop.

```
[rud1] mut my(s= 0); for(i=1, 10, if(0 == i % 2, next()); s+= i); s \\ 25
[rud1] mut my(s= 0); for(x <- [1, 2], for(y <- [10, 20], if(20 == y, next(2)); s+= x + y)); s \\ 23
```

## Strings and Unicode

**The `str.` family.** A `str` is not indexed; `str.to_characters` gives its code points as
a list of texts.

```
[rud1] str.to_characters("héllo") \\ ["h","é","l","l","o"]
[rud1] [str.substr("hello", 1, 3), str.substr("hello", -3)] \\ ["ell","llo"]
[rud1] [str.split("a,,b", ","), str.split("héé", "")] \\ [["a","","b"],["h","é","é"]]
[rud1] my(c= str.to_characters("hello")); str.join(c[#c - 1 .. 0; -1]) \\ "olleh"
[rud1] str.join([str.to_titlecase_first(w) | w <- str.split("a red fox", " ")], " ") \\ "A Red Fox"
```

**Case mapping.** Unicode's full mappings, untailored.

```
[rud1] [str.to_uppercase("straße"), str.to_lowercase("ΟΔΟΣ")] \\ ["STRASSE","οδος"]
[rud1] [str.to_titlecase_first("ijs"), str.to_casefold("ß")] \\ ["Ijs","ss"]
[rud1] [str.eq_caseless("Straße", "STRASSE"), "Straße" == "STRASSE"] \\ [1,0]
[rud1] [str.is_lowercase("abc"), str.is_uppercase("abc"), str.unicode_version()] \\ [1,0,[17,0,0]]
```

**Code points.**

```
[rud1] str.to_codepoints("hé") \\ [104,233]
[rud1] str.from_codepoints([c + 1 | c <- str.to_codepoints("HAL")]) \\ "IBM"
[rud1] ["\u{e9}" == "é", #"e\u{301}", "a\tb"] \\ [1,2,"a\x{09}b"]
```

**`bytes`.** Written `b"…"`; `str(b)` decodes it as UTF-8, or is an `error`. Not built:
`bytes(s)`, slices and a walk over a `bytes` value.

```
[rud1] [#"hé", #b"h\x{c3}\x{a9}", str(b"h\x{c3}\x{a9}")] \\ [2,3,"hé"]
[rud1] [b"abc"[0], b"abc"[-1]] \\ [97,99]
[rud1] b"\x{89}PNG" ~ b"\r\n" \\ b"\x{89}PNG\x{0d}\x{0a}"
[rud1] [b"ab" == b"ab", b"ab" < b"b", guard(str(b"\x{ff}"), "not UTF-8")] \\ [1,1,"not UTF-8"]
```

## Errors

**`guard`.** `guard(e, d)` is `e`, or `d` when `e` is an `error` or not defined.

```
[rud1] guard(1/0, 0) \\ 0
[rud1] my(v= [1, 2]); guard(v[5], "none") \\ "none"
[rud1] my(safe_div(a, b)= guard(a / b, 0)); [safe_div(6, 3), safe_div(1, 0)] \\ [2,0]
```

**`error(msg)`.** An `error` that nothing guards ends the program, with status 1.

```
[rud1] my(half(n)= if(n % 2, error("odd"), n // 2)); [half(8), guard(half(7), -1)] \\ [4,-1]
```

```
[rud1]
my(half(n)= if(n % 2, error("odd: " ~ str(n)), n // 2));
say(half(8));
say(half(7));
say("not reached");
```

Output:

```
4
we exited <error("odd: 7")> at 091.rud:1:23
  in half at 091.rud:3:5
```

Exit status: 1.

**`is_defined`.** A name declared without a value, `max` of an empty list and `read_line()`
at the end of input are not defined (the second line: run with an empty standard input).

```
[rud1] my(r); [is_defined(r), is_defined(1), is_defined(max([]))] \\ [0,1,0]
[rud1] is_defined(read_line()) \\ 0
```

**`defer` and `exit`.** What a block defers runs when the block ends, the last deferred
first. `exit(n, msg)` ends the program with status `n`; the message of a status other than 0
goes to standard error, and what was deferred still runs.

```
[rud1]
{ defer(say("d1"));
  defer(say("d2"));
  say("working");
  exit(2, "giving up");
  say("not reached")
}
```

Output:

```
working
giving up
d2
d1
```

Exit status: 2.

## Reading standard input

`read_line()` gives the input up to and including the next line feed, unchanged, and is not
defined at the end of input. Run as `rud1_run FILE < input`.

```
[rud1]
my(name= str.split(read_line(), "\n")[0]);
say("Hello, ", name, ".");
```

Input:

```
Ada
```

Output:

```
Hello, Ada.
```

Every line, to the end of input:

```
[rud1]
mut my(n= 0, total= 0);
while
( is_defined(my(line= read_line()))
, n+= 1;
  total+= sum([sint(w) | w <- str.split(str.split(line, "\n")[0], " "), "" != w])
);
say(n, " lines, total ", total);
```

Input:

```
1 2 3
40

5 -1
```

Output:

```
4 lines, total 50
```

## Templates and `unpack`

`unpack(t, b)` reads a `bytes` value by a template `t{…}` and gives a list, with names where
the template has them. It has to cover the data; a text field is `bytes`. `pack`, and
`unpack(t)` on standard input, are decided, not built.

**Fixed fields.** `C` is a byte, `n` and `N` are 16 and 32 bits big-endian, `v` and `V`
little-endian; `c s l` are signed, little-endian; `a4` is four bytes of text.

```
[rud1] unpack(t{C n v}, b"\x{ff}\x{01}\x{02}\x{03}\x{00}") \\ [255,258,3]
[rud1] unpack(t{c s l}, b"\x{ff}\x{fe}\x{ff}\x{fd}\x{ff}\x{ff}\x{ff}") \\ [-1,-2,-3]
[rud1] unpack(t{magic:a4 size:V}, b"RIFF\x{24}\x{00}\x{00}\x{00}") \\ [magic:b"RIFF",size:36]
[rud1] guard(unpack(t{C n}, b"\x{01}"), "too short") \\ "too short"
```

**Counts.** A number after a number code gives a list, `*` runs to the end, `x*` takes the rest as
skipped bytes.

```
[rud1] unpack(t{C3 n*}, b"\x{01}\x{02}\x{03}\x{00}\x{04}\x{00}\x{05}") \\ [[1,2,3],[4,5]]
[rud1] unpack(t{(n C)2 x*}, b"\x{00}\x{01}\x{02}\x{00}\x{03}\x{04}zz") \\ [[[1,2],[3,4]],b"zz"]
```

**A length-prefixed text.** `C/a` is a count byte and that many bytes; the count is an item
of the result too. The other two end a text at a mark.

```
[rud1] unpack(t{name:C/a age:C}, b"\x{03}Ada\x{24}") \\ [3,name:b"Ada",age:36]
[rud1] my(r= unpack(t{name:C/a age:C}, b"\x{03}Ada\x{24}")); [str(r.name), r.age] \\ ["Ada",36]
[rud1] unpack(t{name:a* "\x{0}" n:C}, b"abc\x{00}\x{07}") \\ [name:b"abc",b"\x{00}",n:7]
[rud1] unpack(t{a["\n"] a["\n"] a*}, b"one\ntwo\nlast") \\ [b"one\x{0a}",b"two\x{0a}",b"last"]
```

**A repeated group.** `=n:C` names a count, `[=n]` uses it.

```
[rud1]
my(T= t{"PK" =n:C files:(name:C/a size:v)[=n]});
my(r= unpack(T, b"PK\x{02}\x{01}a\x{05}\x{00}\x{02}bc\x{07}\x{00}"));
[r.n, [str(f.name) | f <- r.files], r.files[1]] \\ [2,["a","bc"],[2,name:b"bc",size:7]]
```

**A struct with names.** A struct `{…}` names every member and reports them in the order of
the names.

```
[rud1] unpack(t{{id:n flags:C}}, b"\x{00}\x{07}\x{01}") \\ [[flags:1,id:7]]
[rud1] my(r= unpack(t{head:{id:n flags:C} x*}, b"\x{00}\x{07}\x{01}abc")); r.head.id \\ 7
```

**Bit fields.** `B4` takes the high bits of a byte first, `b4` the low bits; bit fields in a
row share bytes.

```
[rud1] unpack(t{version:B4 ihl:B4}, b"\x{45}") \\ [version:4,ihl:5]
[rud1] unpack(t{low:b4 high:b4}, b"\x{45}") \\ [low:5,high:4]
[rud1] unpack(t{day:b5 month:b4 year:b7}, b"\x{4c}\x{5d}") \\ [day:12,month:10,year:46]
```

**A search or jump.** `@"PK"` searches forward; `.` marks the start, `@[=at]` jumps to that
many bytes after it. A jump leaves the count of bytes passed.

```
[rud1] unpack(t{@"PK" sig:a2 x*}, b"xxPKab") \\ [2,sig:b"PK",b"ab"]
[rud1] unpack(t{. =at:C @[=at] v:C @*}, b"\x{03}\x{00}\x{00}\x{09}\x{08}") \\ [at:3,2,v:9,1]
```

**A text pattern with a set.** `%[a-z]` is one byte of the set, with `+` or `*` a run; `$dec`
and `$hex` read digits as a number; `&s` is a white space byte, `&w` a word byte.

```
[rud1] unpack(t{key:%[a-z]+ "=" value:%[0-9]+}, b"width=640") \\ [key:b"width",b"=",value:b"640"]
[rud1] unpack(t{"#" r:$hex[2] g:$hex[2] b:$hex[2]}, b"#ff8000") \\ [b"#",r:255,g:128,b:0]
[rud1] unpack(t{(?i: "host:") &s* name:&w+}, b"HOST: example") \\ [b"HOST:",b" ",name:b"example"]
```

```
[rud1]
my(r= unpack(t{pairs:(key:%[a-z _]+ "=" value:$dec+ ";")*}, b"max_w=640;h=480;"));
[[str(p.key), p.value] | p <- r.pairs] \\ [["max_w",640],["h",480]]
```

**Alternatives.** `|` reads the variant that fits; `||` is "or else"; a test `??[…]` at the head
of a variant chooses by a count read before. A pattern `&name ::= … ;` is defined once and
used by its name.

```
[rud1] unpack(t{kind:("GIF87a" | "GIF89a") w:v}, b"GIF89a\x{10}\x{00}") \\ [kind:[b"GIF89a"],w:16]
[rud1] unpack(t{(num:$dec+ || word:&w+)}, b"abc") \\ [[word:b"abc"]]
```

```
[rud1]
my(T= t{=tag:C ( ??[=tag == 1] num:n | ??[=tag == 2] text:C/a )});
unpack(T, b"\x{02}\x{02}hi") \\ [tag:2,[2,text:b"hi"]]
```

```
[rud1]
my(T= t{&val ::= int:("I" n:C) | str:("S" s:C/a) ; &val*});
unpack(T, b"I\x{05}S\x{02}hi") \\ [[[int:[b"I",n:5]],[str:[b"S",2,s:b"hi"]]]]
```

### A PNG: signature and chunks

The bytes are a PNG of 2 by 1 pixels, made with python 3.12.3 (`zlib`, `struct`). A template
may span lines and hold `#` comments.

```
[rud1]
my
( PNG=
    b"\x{89}PNG\x{0d}\x{0a}\x{1a}\x{0a}\x{00}\x{00}\x{00}\x{0d}IHDR\x{00}\x{00}\x{00}"
  ~ b"\x{02}\x{00}\x{00}\x{00}\x{01}\x{08}\x{00}\x{00}\x{00}\x{00}\x{d1}I V\x{00}\x{00}"
  ~ b"\x{00}\x{10}tEXtTitle\x{00}two pixelsD!.'\x{00}\x{00}\x{00}\x{0b}IDATx\x{9c}c`\x{f8}"
  ~ b"\x{0f}\x{00}\x{01}\x{02}\x{01}\x{00}B\x{be}\x{bc}h\x{00}\x{00}\x{00}\x{00}IEND\x{ae}"
  ~ b"B`\x{82}"
, T= t{ "\x{89}PNG\r\n\x{1a}\n"                            # the signature
        =len:N "IHDR"
        ihdr:([=len] width:N height:N depth:C color:C x*)  # a group of len bytes
        crc:N
        chunks:(=len:N type:a4 data:a[=len] crc:N)*
      }
, r= unpack(T, PNG)
);
say(r.ihdr.width, " x ", r.ihdr.height, ", depth ", r.ihdr.depth);
for(c <- r.chunks, say(str(c.type), " ", c.len, " bytes, crc ", c.crc));
r.chunks[0].data
```

Output:

```
2 x 1, depth 8
tEXt 16 bytes, crc 1143025191
IDAT 11 bytes, crc 1119796328
IEND 0 bytes, crc 2923585666
b"Title\x{00}two pixels"
```

### A gzip member

The bytes are `hello hello hello` and a line feed, 18 bytes, compressed with python 3.12.3
(`gzip.GzipFile`, the name `hi.txt`, `mtime=1`). The flag bits say which parts follow;
`data:a*` leaves the last eight bytes to `crc` and `isize`.

```
[rud1]
my
( GZ=
    b"\x{1f}\x{8b}\x{08}\x{08}\x{01}\x{00}\x{00}\x{00}\x{02}\x{ff}hi.txt\x{00}\x{cb}H"
  ~ b"\x{cd}\x{c9}\x{c9}W\x{c8}@\x{90}\x{5c}\x{00};|\x{8a}\x{df}\x{12}\x{00}\x{00}\x{00}"
, T= t{ "\x{1f}\x{8b}" method:C
        text:b1 =fhcrc:b1 =fextra:b1 =fname:b1 =fcomment:b1 =rsv:b3 ??[=rsv == 0]
        mtime:V xfl:C os:C
        extra:(v/a)[=fextra]
        name:(a* "\x{0}")[=fname]
        comment:(a* "\x{0}")[=fcomment]
        hcrc:(v)[=fhcrc]
        data:a* crc:V isize:V
      }
, r= unpack(T, GZ)
);
say("method ", r.method, ", mtime ", r.mtime, ", os ", r.os);
say("name ", str(r.name[0][0]), ", ", #r.data, " bytes of deflate data");
[crc: r.crc, isize: r.isize]
```

Output:

```
method 8, mtime 1, os 255
name hi.txt, 11 bytes of deflate data
[crc:3750394939,isize:18]
```

## Classic programs

From the prototype's `rosetta` directory, unchanged: `basic` is written as a coder new to the
language writes, `seasoned` as one who knows it.

### FizzBuzz (`rosetta/basic/FizzBuzz.rud`)

```
[rud1]
/* Rosetta Code: FizzBuzz */
my
( fizzbuzz(n)= if
  ( 0 == n % 15, "FizzBuzz"
  , 0 == n %  5, "Buzz"
  , 0 == n %  3, "Fizz"
  , n
  )
);
for(i=1, 100, say(fizzbuzz(i)));
0;
```

Output, the first 15 of 100 lines:

```
1
2
Fizz
4
Buzz
Fizz
7
8
Fizz
Buzz
11
Fizz
13
14
FizzBuzz
```

### Sieve of Eratosthenes (`rosetta/basic/Sieve_of_Eratosthenes.rud`)

```
[rud1]
/* Rosetta Code: Sieve of Eratosthenes */
my
( primes(n)=
  { mut my(s= [1] ~* (n + 1), p= 2);
    while
    ( p * p <= n
    , if(s[p], for(m <- [p * p .. n; p], s[m]= 0));
      p+= 1
    );
    [k | k <- [2..n], s[k]]
  }
);
say(primes(100));
```

Output:

```
[2,3,5,7,11,13,17,19,23,29,31,37,41,43,47,53,59,61,67,71,73,79,83,89,97]
```

### Greatest common divisor (`rosetta/basic/Greatest_common_divisor.rud`)

```
[rud1]
/* Rosetta Code: Greatest common divisor */
my(gcd(a, b)= if(0 == b, a, gcd(b, a % b)));
say([gcd(1071, 462), gcd(462, 1071), gcd(17, 0), gcd(0, 0), gcd(2^64, 6^40)]);
```

Output:

```
[21,21,17,0,1099511627776]
```

### Fibonacci sequence (`rosetta/seasoned/Fibonacci_sequence.rud`)

```
[rud1]
/* Rosetta Code: Fibonacci sequence */
my(fib(n)= fold((p, k) -> [p[1], p[0] + p[1]], [0, 1], [1..n])[0]);
say([fib(n) | n <- [0..20]]);
say(fib(100));
```

Output:

```
[0,1,1,2,3,5,8,13,21,34,55,89,144,233,377,610,987,1597,2584,4181,6765]
354224848179261915075
```

### Hailstone sequence (`rosetta/seasoned/Hailstone_sequence.rud`)

```
[rud1]
/* Rosetta Code: Hailstone sequence */
my
( step(n)= if(n % 2, 3 * n + 1, n // 2)
, hailstone(n)= if(1 == n, [1], [n] ~ hailstone(step(n)))
, hlen(n)=
  { mut my(m= n, k= 1);
    while(1 != m, m= step(m); k+= 1);
    k
  }
, longest(N)=           /* [n, length] of the longest sequence with n <= N */
  { mut my(best= [1, 1]);
    for(n=1, N, my(k= hlen(n)); if(k > best[1], best= [n, k]));
    best
  }
, h= hailstone(27)
);
say(#h, " elements: ", h[0..3], " ... ", h[-4 .. -1]);
say(longest(703));
```

Output:

```
112 elements: [27,82,41,124] ... [8,4,2,1]
[703,171]
```

### Reverse a string (`rosetta/seasoned/Reverse_a_string.rud`)

```
[rud1]
/* Rosetta Code: Reverse a string (by code points) */
my(reverse(s)= fold((r, c) -> c ~ r, "", str.to_characters(s)));
say(reverse("asdf"));
say(reverse("møøse"));
```

Output:

```
fdsa
esøøm
```

### A+B (`rosetta/seasoned/A+B.rud`)

Its standard input is `rosetta/in/A+B.in`.

```
[rud1]
/* Rosetta Code: A+B */
my(numbers(s)= [sint(w) | w <- str.split(str.split(s, "\n")[0], " "), "" != w]);
say(sum(numbers(read_line())));
```

Input:

```
 2  -3 
```

Output:

```
-1
```
