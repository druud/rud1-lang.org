# rud1 examples

*This document speaks from the future: it shows rud1 as of its first fully operating
release. Every program in it runs on today's prototype too; what the prototype does not do yet
is the document "Not yet".*

A cookbook for rud1 (on the site `rud1-lang.org`): programs, each with the output it gives.
How far today's prototype is, is the document "Not yet".

Running: `rud1 'code'` or `rud1 file.rud`; source files end in `.rud`. The value of the last expression is shown, unless the source ends
in `;`. A one-liner is given as `[rud1] code \\> result`: `\\` starts a comment, so the line
without its `[rud1]` is valid code; the `>` marks the value, as in the REPL. A longer program is a block that starts with `[rud1]`,
with its result after `\\>` or in a block headed "Output:".

## One-liners

**Arithmetic.** `/` is exact, `//` and `%` are the Euclidean quotient and its remainder, which is never negative.

```
[rud1] 1 + 2 * 3  \\> 7
[rud1] [7 / 2, 7 // 2, 7 % 2, -7 // 2, -7 % 2]  \\> [7/2,3,1,-4,1]
[rud1] [2^10, -3^2, 2^3^2]  \\> [1024,-9,512]
[rud1] [1 < 2 <= 3, 7 <=> 3, abs(-5)]  \\> [1,1,5]
[rud1] [1.5 + 0.25, 3.0 / 4]  \\> [1.75000000000000,0.750000000000000]
```

**Big numbers.** Whole numbers have no upper limit.

```
[rud1] 2^100  \\> 1267650600228229401496703205376
```

**Rationals.**

```
[rud1] 1/3 + 1/6  \\> 1/2
[rud1] [(2/3)^3, 2^-3, (1/2)^-2]  \\> [8/27,1/8,4]
[rud1] sum([1/k | k <- [1..10]])  \\> 7381/2520
[rud1] [1/4 + 0.5, 0.5 == 1/2]  \\> [0.750000000000000,1]
```

**Ranges.** A step follows `;`; a range among other elements is spliced in.

```
[rud1] [1..10]  \\> [1,2,3,4,5,6,7,8,9,10]
[rud1] [[1..9;2], [10..1;-3]]  \\> [[1,3,5,7,9],[10,7,4,1]]
[rud1] [1, 5..7, 9]  \\> [1,5,6,7,9]
[rud1] my(L= [1..oo]); L[0..4]  \\> [1,2,3,4,5]
```

**Comprehensions.** Filters follow the generator; several generators stand in a bracket.

```
[rud1] [x^2 | x <- [1..8]]  \\> [1,4,9,16,25,36,49,64]
[rud1] [x | x <- [1..30], 0 == x % 7]  \\> [7,14,21,28]
[rud1] [[a, b] | [a <- [1..3], b <- [a..3]]]  \\> [[1,1],[1,2],[1,3],[2,2],[2,3],[3,3]]
[rud1] [[i * j | j <- [1..4]] | i <- [1..3]]  \\> [[1,2,3,4],[2,4,6,8],[3,6,9,12]]
```

**Sums and folds.** `fold(f, L)` combines the elements from the left, `f(f(L[0], L[1]), L[2])` and so on; `fold(f, a, L)` starts from `a`. `scan` gives every value the fold passes through.

```
[rud1] sum([1..10], _1^2)  \\> 385
[rud1] [fold(_+_, [1, 2, 3, 4]), fold(_*_, 1, [1..6])]  \\> [10,720]
[rud1] scan(_+_, [1, 2, 3, 4])  \\> [1,3,6,10]
[rud1] [max(3, 9, 4), min([5, 2, 8]), fold(max, [3, 9, 4])]  \\> [9,2,9]
```

**Strings.** `~` joins, `~*` repeats, `#` counts code points; `str` gives the text of a number.

```
[rud1] ["abc" ~ "def", "ab" ~* 3, #"héllo"]  \\> ["abcdef","ababab",5]
[rud1] [str(42) ~ "!", sint("42") + 1]  \\> ["42!",43]
[rud1] str.split("a,b,c", ",")  \\> ["a","b","c"]
[rud1] str.join(["a", "b", "c"], "-")  \\> "a-b-c"
[rud1] say("x = ", 5);  \\> x = 5
```

## Functions

**Recursion.** A function calls itself as `__SELF__(…)`; the functions of one `my(…)`
see each other, in any order, and call each other by name (`ev` and `od` below).

```
[rud1] my(fact(n)= if(n < 2, 1, n * __SELF__(n - 1))); fact(30)  \\> 265252859812191058636308480000000
[rud1] my(ev(n)= if(0 == n, 1, od(n - 1)), od(n)= if(0 == n, 0, ev(n - 1))); ev(10)  \\> 1
```

A body of several steps is a block; `leave(2, 0)` leaves the loop and the block with 0.

```
[rud1]
my
( is_prime(n)=
  { mut my(d= 2);
    while(d * d <= n, if(0 == n % d, leave(2, false)); d+= 1);
    n > 1
  }
);
[n | n <- [1..40], is_prime(n)]  \\> [2,3,5,7,11,13,17,19,23,29,31,37]
```

**Lambdas.**

```
[rud1] my(sq= x -> x^2); [sq(k) | k <- [1..5]]  \\> [1,4,9,16,25]
[rud1] my(add= (a, b) -> a + b); add(2, 3)  \\> 5
[rud1] my(f= x -> y -> x + y); f(1)(2)  \\> 3
[rud1] my(compose(f, g)= x -> f(g(x)), h= compose(x -> x + 1, x -> x * 2)); h(5)  \\> 11
```

**Defaults and named arguments.** A default may use the parameters before it; an empty
argument takes the default.

```
[rud1] my(f(a, b= 5, c= a + b)= [a, b, c]); [f(1), f(1, , 9)]  \\> [[1,5,6],[1,5,9]]
[rud1] my(area(w, h= w)= w * h); [area(3), area(3, 4), area(h: 2, w: 5)]  \\> [9,12,10]
[rud1] my(f(a, b, c)= [a, b, c]); f(c: 9, a: 1, b: 5)  \\> [1,5,9]
```

**Rest and spread.** `v[...]` takes the remaining arguments as a list, `v[1...]` at least
one; `L[...]` spreads a list over the parameters.

```
[rud1] my(f(a, v[...])= [a, v]); [f(1), f(1, 2, 3)]  \\> [[1,[]],[1,[2,3]]]
[rud1] my(mean(v[1...])= sum(v) / #v); mean(1, 2, 6)  \\> 3
[rud1] my(f(a, b, c)= a + b * c, L= [2, 3]); f(1, L[...])  \\> 7
```

**Operators as values.** An operator between two `_` is a function.

```
[rud1] fold(_~_, ["a", "b", "c"])  \\> "abc"
[rud1] [_<=>_(1, 2), _^_(2, 10)]  \\> [-1,1024]
```

**Placeholders.** `_1`, `_2` in an argument make that argument a function of them.

```
[rud1] my(twice(f, x)= f(f(x))); twice(_1 * 3, 5)  \\> 45
[rud1] fold(_1 + 2 * _2, [1, 2, 3])  \\> 11
```

## Lists and records

**Indexing.** Positions count from 0; a literal negative one counts from the end (`v[-1]` is
`v[#__ - 1]`, `__` the list being indexed); a computed one below 0 is the state `underflow`.

```
[rud1] my(v= [10, 20, 30, 40]); [v[0], v[-1], v[1..2], #v]  \\> [10,40,[20,30],4]
[rud1] mut my(v= [1, 2, 3]); v[1]= 9; v~= [4]; v  \\> [1,9,3,4]
[rud1] my(M= [1, 2, 3; 4, 5, 6]); [M[1][2], M[0], [r[0] | r <- M]]  \\> [6,[1,2,3],[1,4]]
```

**Gather and scatter.** Several positions give a list; assigned to, they take a list.

```
[rud1] my(v= [10, 20, 30, 40]); v[3, 0]  \\> [40,10]
[rud1] mut my(v= [1, 2, 3]); v[0, 2]= v[2, 0]; v  \\> [3,2,1]
```

**Stepped ranges.** As a list (a list of steps is taken in turn), and as positions: every
second element, the reverse.

```
[rud1] [[0..20;5], [20..0;-5]]  \\> [[0,5,10,15,20],[20,15,10,5,0]]
[rud1] my(w= [4, 2, 4, 2, 4, 6, 2, 6]); [7..40;w]  \\> [7,11,13,17,19,23,29,31,37]
[rud1] my(v= [0, 1, 2, 3, 4]); [v[0 .. #v - 1; 2], v[#v - 1 .. 0; -1]]  \\> [[0,2,4],[4,3,2,1,0]]
```

**Named lists.** A record is a list with names on its elements.

```
[rud1] my(p= [x: 1, y: 2]); [p.x, p[1], #p, names(p)]  \\> [1,2,2,["x","y"]]
[rud1] mut my(p= [x: 1, y: 2]); p.x+= 10; p  \\> [x:11,y:2]
[rud1] [x: 1] ~ [2, y: 3]  \\> [x:1,2,y:3]
[rud1] my(L= [[x: 1, y: 2], [x: 3, y: 4]]); [p.x * p.y | p <- L]  \\> [2,12]
```

**Destructuring.**

```
[rud1] my([a, b]= [1, 2]); a + b  \\> 3
[rud1] my(divmod(a, b)= [a // b, a % b], [q, r]= divmod(17, 5)); [q, r]  \\> [3,2]
[rud1] mut my(a= 1, b= 2); [a, b]= [b, a]; [a, b]  \\> [2,1]
```

## Loops

A script that ends in a loop is ended with `0;`: a loop that ran no round has no value.

**`for`.**

```
[rud1] mut my(s= 0); for(i= 1, 100, s+= i); s  \\> 5050
[rud1] mut my(s= 0); for([a, b] <- [[1, 2], [3, 4]], s+= a * b); s  \\> 14
```

```
[rud1]
my(N= 3);
for(i= 1, N, say(str.join([str(i * j) | j <- [1..N]], " ")));
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
[rud1] mut my(i= 0, s= 0); while(i < 5, i+= 1; s+= i); [i, s]  \\> [5,15]
[rud1] mut my(n= 27, k= 0); while(1 != n, n= if(0 != n % 2, 3 * n + 1, n / 2); k+= 1); k  \\> 111
[rud1] mut my(n= 1234); until(n > 1000, n*= 2); n  \\> 2468
```

**`leave`.** `leave(, v)` ends the loop with the value `v`; `leave(2, v)` ends two levels.

```
[rud1] my(first(L, p)= for(x <- L, if(p(x), leave(, x)))); first([3, 8, 12, 5], _1 > 10)  \\> 12
[rud1] mut my(k= 0); while(true, k+= 1; if(k^2 > 50, leave())); k  \\> 8
[rud1] my(f(L)= { for(x <- L, if(x > 1, leave(2, x))); 0 }); [f([1, 2, 3]), f([1, 1])]  \\> [2,0]
```

**`next`.** `next()` goes to the next round of its loop, `next(2)` to the next round of the loop one level up.

```
[rud1] mut my(s= 0); for(i= 1, 10, if(0 == i % 2, next()); s+= i); s  \\> 25
[rud1] mut my(s= 0); for(x <- [1, 2], for(y <- [10, 20], if(20 == y, next(2)); s+= x + y)); s  \\> 23
```

## Strings and Unicode

**The `str.` family.** A `str` has no index; `str.to_characters` splits it into its code points, each a
`str` of one.

```
[rud1] str.to_characters("héllo")  \\> ["h","é","l","l","o"]
[rud1] [str.substr("hello", 1, 3), str.substr("hello", -3)]  \\> ["ell","llo"]
[rud1] [str.split("a,,b", ","), str.split("héé", "")]  \\> [["a","","b"],["h","é","é"]]
[rud1] my(c= str.to_characters("hello")); str.join(c[#c - 1 .. 0; -1])  \\> "olleh"
[rud1] str.join([str.to_titlecase_first(w) | w <- str.split("a red fox", " ")], " ")  \\> "A Red Fox"
```

**Case mapping.** Unicode's full mappings, untailored.

```
[rud1] [str.to_uppercase("straße"), str.to_lowercase("ΟΔΟΣ")]  \\> ["STRASSE","οδος"]
[rud1] [str.to_titlecase_first("ijs"), str.to_casefold("ß")]  \\> ["Ijs","ss"]
[rud1] [str.eq_caseless("Straße", "STRASSE"), "Straße" == "STRASSE"]  \\> [1,0]
[rud1] [str.is_lowercase("abc"), str.is_uppercase("abc"), str.unicode_version()]  \\> [1,0,[17,0,0]]
```

**Code points.**

```
[rud1] str.to_codepoints("hé")  \\> [104,233]
[rud1] str.from_codepoints([c + 1 | c <- str.to_codepoints("HAL")])  \\> "IBM"
[rud1] ["\u{e9}" == "é", #"e\u{301}", "a\tb"]  \\> [1,2,"a\x{09}b"]
```

**`bytes`.** Written `b"…"`; `str(b)` decodes it as UTF-8, or is an `error`. A slice of it is a `bytes` value, and a slice can be replaced
(`b[0..0]= b"zz"`); `bytes(s)` encodes a `str` as UTF-8; an index gives one byte, a `bytes` value, `uint` of it its
number, and a walk gives the bytes as numbers.

```
[rud1] [#"hé", #b"h\x{c3}\x{a9}", str(b"h\x{c3}\x{a9}")]  \\> [2,3,"hé"]
[rud1] [b"abc"[0], uint(b"abc"[-1]), b"abc"[0..1], b"abc"[0, 2]]  \\> [b"a",99,b"ab",b"ac"]
[rud1] b"\x{89}PNG" ~ b"\r\n"  \\> b"\x{89}PNG\x{0d}\x{0a}"
[rud1] [b"ab" == b"ab", b"ab" < b"b", guard(str(b"\x{ff}"), "not UTF-8")]  \\> [1,1,"not UTF-8"]
```

## Errors

**`guard`.** `guard(e, d)` is `e`, or `d` when `e` is an `error` or not defined.

```
[rud1] guard(1/0, 0)  \\> 0
[rud1] my(v= [1, 2]); guard(v[5], "none")  \\> "none"
[rud1] my(safe_div(a, b)= guard(a / b, 0)); [safe_div(6, 3), safe_div(1, 0)]  \\> [2,0]
```

**`error(msg)`.** An `error` that nothing guards ends the program, with status 1.

```
[rud1] my(half(n)= if(0 != n % 2, error("odd"), n / 2)); [half(8), guard(half(7), -1)]  \\> [4,-1]
```

```
[rud1]
my(half(n)= if(0 != n % 2, error("odd: " ~ str(n)), n / 2));
say(half(8));
say(half(7));
say("not reached");
```

Output:

```
4
we exited <error("odd: 7")> at 091.rud:1:28
  in half at 091.rud:3:5
```

Exit status: 1.

**`is_defined`.** A name declared without a value, `max` of an empty list and `read_line()`
at the end of input are not defined (the second line: run with an empty standard input).

```
[rud1] my(r); [is_defined(r), is_defined(1), is_defined(max([]))]  \\> [0,1,0]
[rud1] is_defined(read_line())  \\> 0
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
the template has them. It has to cover the data; a text field is `bytes`. Every template is
given a name, `tpl_…`; the table at the end of the section lists them, with what each one
reads.
The examples are sessions, as the REPL shows them: `? ` starts what is typed, `> `
the value it shows; an entry that ends in `;` shows nothing.

**Fixed fields.** `C` is a byte, `n` and `N` are 16 and 32 bits big-endian, `v` and `V`
little-endian; `c s l` are signed, little-endian; `a4` is four bytes of text.

```
? my
  ( tpl_byte_be16_le16= t{C n v}
  , tpl_signed_8_16_32= t{c s l}
  , tpl_riff_head=      t{magic:a4 size:V}
  , tpl_byte_be16=      t{C n}
  );
? unpack(tpl_byte_be16_le16, b"\x{ff}\x{01}\x{02}\x{03}\x{00}")
> [255,258,3]

? unpack(tpl_signed_8_16_32, b"\x{ff}\x{fe}\x{ff}\x{fd}\x{ff}\x{ff}\x{ff}")
> [-1,-2,-3]

? unpack(tpl_riff_head, b"RIFF\x{24}\x{00}\x{00}\x{00}")
> [magic:b"RIFF",size:36]

? guard(unpack(tpl_byte_be16, b"\x{01}"), "too short")
> "too short"
```

**Counts.** A number after a number code gives a list, `*` runs to the end, `x*` takes the rest as
skipped bytes.

```
? my
  ( tpl_three_bytes_be16s= t{C3 n*}
  , tpl_two_pairs_rest=    t{(n C)2 x*}
  );
? unpack(tpl_three_bytes_be16s, b"\x{01}\x{02}\x{03}\x{00}\x{04}\x{00}\x{05}")
> [[1,2,3],[4,5]]

? unpack(tpl_two_pairs_rest, b"\x{00}\x{01}\x{02}\x{00}\x{03}\x{04}zz")
> [[[1,2],[3,4]],b"zz"]
```

**A length-prefixed text.** `C/a` is a count byte and that many bytes; the count is an item
of the result too. The other two end a text at a mark.

```
? my
  ( tpl_name_age=       t{name:C/a age:C}
  , tpl_cstring_byte=   t{name:a* "\x{0}" n:C}
  , tpl_two_lines_rest= t{a["\n"] a["\n"] a*}
  );
? my(r= unpack(tpl_name_age, b"\x{03}Ada\x{24}"));
? r
> [3,name:b"Ada",age:36]

? [str(r.name), r.age]
> ["Ada",36]

? unpack(tpl_cstring_byte, b"abc\x{00}\x{07}")
> [name:b"abc",b"\x{00}",n:7]

? unpack(tpl_two_lines_rest, b"one\ntwo\nlast")
> [b"one\x{0a}",b"two\x{0a}",b"last"]
```

**A repeated group.** `=n:C` names a count, `[=n]` uses it.

```
? my(tpl_pk_files= t{"PK" =n:C files:(name:C/a size:v)[=n]});
? my(f= unpack(tpl_pk_files, b"PK\x{02}\x{01}a\x{05}\x{00}\x{02}bc\x{07}\x{00}"));
? f.n
> 2

? [str(e.name) | e <- f.files]
> ["a","bc"]

? f.files[1]
> [2,name:b"bc",size:7]
```

**A struct with names.** A struct `{…}` names every member and reports them in the order of
the names.

```
? my
  ( tpl_struct_id_flags= t{{id:n flags:C}}
  , tpl_head_rest=       t{head:{id:n flags:C} x*}
  );
? unpack(tpl_struct_id_flags, b"\x{00}\x{07}\x{01}")
> [[flags:1,id:7]]

? unpack(tpl_head_rest, b"\x{00}\x{07}\x{01}abc").head.id
> 7
```

**Bit fields.** `B4` takes the high bits of a byte first, `b4` the low bits; bit fields in a
row share bytes.

```
? my
  ( tpl_ip_version_ihl= t{version:B4 ihl:B4}
  , tpl_nibbles=        t{low:b4 high:b4}
  , tpl_dos_date=       t{day:b5 month:b4 year:b7}
  );
? unpack(tpl_ip_version_ihl, b"\x{45}")
> [version:4,ihl:5]

? unpack(tpl_nibbles, b"\x{45}")
> [low:5,high:4]

? unpack(tpl_dos_date, b"\x{4c}\x{5d}")
> [day:12,month:10,year:46]
```

**A search or jump.** `@"PK"` searches forward; `.` marks the start, `@[=at]` jumps to that
many bytes after it. A jump leaves the count of bytes passed.

```
? my
  ( tpl_find_pk= t{@"PK" sig:a2 x*}
  , tpl_jump_at= t{. =at:C @[=at] v:C @*}
  );
? unpack(tpl_find_pk, b"xxPKab")
> [2,sig:b"PK",b"ab"]

? unpack(tpl_jump_at, b"\x{03}\x{00}\x{00}\x{09}\x{08}")
> [at:3,2,v:9,1]
```

**A text pattern with a set.** `%[a-z]` is one byte of the set, with `+` or `*` a run; `$dec`
and `$hex` read digits as a number; `&s` is a white space byte, `&w` a word byte.

```
? my
  ( tpl_key_digits=      t{key:%[a-z]+ "=" value:%[0-9]+}
  , tpl_rgb_hex=         t{"#" r:$hex[2] g:$hex[2] b:$hex[2]}
  , tpl_host_header=     t{(?i: "host:") &s* name:&w+}
  , tpl_key_value_pairs= t{pairs:(key:%[a-z _]+ "=" value:$dec+ ";")*}
  );
? my(kv= unpack(tpl_key_value_pairs, b"max_w=640;h=480;"));
? unpack(tpl_key_digits, b"width=640")
> [key:b"width",b"=",value:b"640"]

? unpack(tpl_rgb_hex, b"#ff8000")
> [b"#",r:255,g:128,b:0]

? unpack(tpl_host_header, b"HOST: example")
> [b"HOST:",b" ",name:b"example"]

? [[str(p.key), p.value] | p <- kv.pairs]
> [["max_w",640],["h",480]]
```

**Alternatives.** `|` reads the variant that fits; `||` is "or else"; a test `??[…]` at the head
of a variant chooses by a count read before. A pattern `&name ::= … ;` is defined once and
used by its name.

```
? my
  ( tpl_gif_head=       t{kind:("GIF87a" | "GIF89a") w:v}
  , tpl_number_or_word= t{(num:$dec+ || word:&w+)}
  , tpl_tagged_value=   t{=tag:C ( ??[=tag == 1] num:n | ??[=tag == 2] text:C/a )}
  , tpl_typed_values=   t{&val ::= int:("I" n:C) | str:("S" s:C/a) ; &val*}
  );
? unpack(tpl_gif_head, b"GIF89a\x{10}\x{00}")
> [kind:[b"GIF89a"],w:16]

? unpack(tpl_number_or_word, b"abc")
> [[word:b"abc"]]

? unpack(tpl_tagged_value, b"\x{02}\x{02}hi")
> [tag:2,[2,text:b"hi"]]

? unpack(tpl_typed_values, b"I\x{05}S\x{02}hi")
> [[[int:[b"I",n:5]],[str:[b"S",2,s:b"hi"]]]]
```

### A PNG: signature and chunks

The bytes are a PNG of 2 by 1 pixels, made with python 3.12.3 (`zlib`, `struct`). A template
may span lines and hold comments, `\\` as in rud1 code (Perl's `#` with a blank after it works too).

```
? my
  ( PNG=
      b"\x{89}PNG\x{0d}\x{0a}\x{1a}\x{0a}\x{00}\x{00}\x{00}\x{0d}IHDR\x{00}\x{00}\x{00}"
    ~ b"\x{02}\x{00}\x{00}\x{00}\x{01}\x{08}\x{00}\x{00}\x{00}\x{00}\x{d1}I V\x{00}\x{00}"
    ~ b"\x{00}\x{10}tEXtTitle\x{00}two pixelsD!.'\x{00}\x{00}\x{00}\x{0b}IDATx\x{9c}c`\x{f8}"
    ~ b"\x{0f}\x{00}\x{01}\x{02}\x{01}\x{00}B\x{be}\x{bc}h\x{00}\x{00}\x{00}\x{00}IEND\x{ae}"
    ~ b"B`\x{82}"
  , tpl_png= t{ "\x{89}PNG\r\n\x{1a}\n"                            \\ the signature
                =len:N "IHDR"
                ihdr:([=len] width:N height:N depth:C color:C x*)  \\ a group of len bytes
                crc:N
                chunks:(=len:N type:a4 data:a[=len] crc:N)*
              }
  , r= unpack(tpl_png, PNG)
  );
? [r.ihdr.width, r.ihdr.height, r.ihdr.depth]
> [2,1,8]

? [str(c.type) | c <- r.chunks]
> ["tEXt","IDAT","IEND"]

? [c.len | c <- r.chunks]
> [16,11,0]

? [c.crc | c <- r.chunks]
> [1143025191,1119796328,2923585666]

? r.chunks[0].data
> b"Title\x{00}two pixels"
```

### A gzip member

The bytes are `hello hello hello` and a line feed, 18 bytes, compressed with python 3.12.3
(`gzip.GzipFile`, the name `hi.txt`, `mtime=1`). The flag bits say which parts follow;
`data:a*` leaves the last eight bytes to `crc` and `isize`.

```
? my
  ( GZ=
      b"\x{1f}\x{8b}\x{08}\x{08}\x{01}\x{00}\x{00}\x{00}\x{02}\x{ff}hi.txt\x{00}\x{cb}H"
    ~ b"\x{cd}\x{c9}\x{c9}W\x{c8}@\x{90}\x{5c}\x{00};|\x{8a}\x{df}\x{12}\x{00}\x{00}\x{00}"
  , tpl_gzip_member=
      t{ "\x{1f}\x{8b}" method:C
         text:b1 =fhcrc:b1 =fextra:b1 =fname:b1 =fcomment:b1 =rsv:b3 ??[=rsv == 0]
         mtime:V xfl:C os:C
         extra:(v/a)[=fextra]
         name:(a* "\x{0}")[=fname]
         comment:(a* "\x{0}")[=fcomment]
         hcrc:(v)[=fhcrc]
         data:a* crc:V isize:V
       }
  , r= unpack(tpl_gzip_member, GZ)
  );
? [r.method, r.mtime, r.os]
> [8,1,255]

? str(r.name[0][0])
> "hi.txt"

? #r.data
> 11

? [crc: r.crc, isize: r.isize]
> [crc:3750394939,isize:18]
```

### The templates of this section

Every template above, by name, with what it reads.

| Name | Template, and what it reads |
|---|---|
| `tpl_byte_be16` | `t{C n}`<br>Reads a byte, then a 16-bit number (big-endian). |
| `tpl_byte_be16_le16` | `t{C n v}`<br>Reads a byte, a 16-bit number big-endian, and a 16-bit number little-endian. |
| `tpl_cstring_byte` | `t{name:a* "\x{0}" n:C}`<br>Reads a text up to a zero byte, which is kept apart, and then a byte. |
| `tpl_dos_date` | `t{day:b5 month:b4 year:b7}`<br>Reads two bytes of bit fields, low bits first: the day (5 bits), the month (4) and the year (7). |
| `tpl_find_pk` | `t{@"PK" sig:a2 x*}`<br>Skips ahead to the first "PK", takes those two bytes, and keeps the rest. |
| `tpl_gif_head` | `t{kind:("GIF87a" \| "GIF89a") w:v}`<br>Reads the GIF signature of either version, then the width (16 bits, little-endian). |
| `tpl_gzip_member` | `t{"\x{1f}\x{8b}" method:C text:b1 =fhcrc:b1 =fextra:b1 =fname:b1 =fcomment:b1 =rsv:b3 ??[=rsv == 0] mtime:V xfl:C os:C extra:(v/a)[=fextra] name:(a* "\x{0}")[=fname] comment:(a* "\x{0}")[=fcomment] hcrc:(v)[=fhcrc] data:a* crc:V isize:V}`<br>Reads a gzip member: the header, whose flag bits say which optional parts follow, then the compressed data, its CRC and its size. |
| `tpl_head_rest` | `t{head:{id:n flags:C} x*}`<br>Reads a struct "head" of an id (16 bits) and a flags byte, then the rest. |
| `tpl_host_header` | `t{(?i: "host:") &s* name:&w+}`<br>Reads "host:" in any case, any white space, and then a word as the name. |
| `tpl_ip_version_ihl` | `t{version:B4 ihl:B4}`<br>Splits one byte in two halves, the high half first: the version and the header length. |
| `tpl_jump_at` | `t{. =at:C @[=at] v:C @*}`<br>Reads a byte that says where to go, goes there, reads one byte, and skips to the end. |
| `tpl_key_digits` | `t{key:%[a-z]+ "=" value:%[0-9]+}`<br>Reads lower-case letters, "=", and digits, which are kept as text. |
| `tpl_key_value_pairs` | `t{pairs:(key:%[a-z _]+ "=" value:$dec+ ";")*}`<br>Reads any number of "key=number;" pairs, each number as a number. |
| `tpl_name_age` | `t{name:C/a age:C}`<br>Reads a text with its length in a byte before it, then an age byte. |
| `tpl_nibbles` | `t{low:b4 high:b4}`<br>Splits one byte in two halves, the low half first. |
| `tpl_number_or_word` | `t{(num:$dec+ \|\| word:&w+)}`<br>Reads a number if digits stand there, and a word otherwise. |
| `tpl_pk_files` | `t{"PK" =n:C files:(name:C/a size:v)[=n]}`<br>Reads "PK" and a count, then that many files, each a name with its length and a size (16 bits, little-endian). |
| `tpl_png` | `t{"\x{89}PNG\r\n\x{1a}\n" =len:N "IHDR" ihdr:([=len] width:N height:N depth:C color:C x*) crc:N chunks:(=len:N type:a4 data:a[=len] crc:N)*}`<br>Reads a PNG: the signature, the IHDR chunk with the image's width, height, depth and colour type, then the other chunks up to the end, each with its length, type, data and CRC. |
| `tpl_rgb_hex` | `t{"#" r:$hex[2] g:$hex[2] b:$hex[2]}`<br>Reads "#" and three colour values of two hex digits each. |
| `tpl_riff_head` | `t{magic:a4 size:V}`<br>Reads four bytes of text (the magic), then a size (32 bits, little-endian). |
| `tpl_signed_8_16_32` | `t{c s l}`<br>Reads three signed numbers, little-endian, of 8, 16 and 32 bits. |
| `tpl_struct_id_flags` | `t{{id:n flags:C}}`<br>Reads a struct of an id (16 bits) and a flags byte, and gives them by name. |
| `tpl_tagged_value` | `t{=tag:C ( ??[=tag == 1] num:n \| ??[=tag == 2] text:C/a )}`<br>Reads a tag byte: tag 1 is followed by a 16-bit number, tag 2 by a text with its length before it. |
| `tpl_three_bytes_be16s` | `t{C3 n*}`<br>Reads three bytes, then 16-bit numbers up to the end. |
| `tpl_two_lines_rest` | `t{a["\n"] a["\n"] a*}`<br>Reads two lines, each with its line feed, then the rest. |
| `tpl_two_pairs_rest` | `t{(n C)2 x*}`<br>Reads two pairs of a 16-bit number and a byte, then skips the rest. |
| `tpl_typed_values` | `t{&val ::= int:("I" n:C) \| str:("S" s:C/a) ; &val*}`<br>Reads values up to the end, each "I" with a byte, or "S" with a text that has its length before it. |

## An exercise: the Mondrian anagram

"Piet Mondrian" and "I paint modern" have the same letters. Ignore case and give the name a
trailing space, and both are texts of 14 characters: `"piet mondrian "` and `"i paint modern"`.
The exercise: turn the first into the second. Three ways follow, from the most direct to the most
general; each works on positions only, so it does to any other text of 14 characters what it does
to the name.

### Two templates

`tpl_read` cuts the bytes into eleven pieces, each named by the position it starts at
(`p3:a4` takes four bytes, `"t mo"`, which stay together); `tpl_write` writes the same
pieces in the order of the target. A text of bytes is written `b"…"`.

```
? my
  ( tpl_read=  t{p0:a p1:a p2:a p3:a4 p7:a p8:a p9:a p10:a p11:a p12:a p13:a}
  , tpl_write= t{p10:a p13:a p0:a p11:a p1:a p7:a p3:a4 p8:a p2:a p9:a p12:a}
  , anagram(b)= str(pack(tpl_write, unpack(tpl_read, b)))
  );
? anagram(b"piet mondrian ")
> "i paint modern"

? anagram(b"abcdefghijklmn")
> "knalbhdefgicjm"
```

### Five swaps

`v.swap(r1, r2)` is `swap(v, r1, r2)`: the list `v` with the blocks at the two ranges
traded, which may differ in length. Here the swaps work on the list of characters, and
`str.join` makes a text of them again; a `str` takes the same swaps by code point. A search found no way with four swaps.

```
? my
  ( c= "piet mondrian ".to_characters()
    .swap([0..0], [1..1])
    .swap([1..12], [13..13])
    .swap([3..3], [4..9])
    .swap([3..6], [7..7])
    .swap([3..10], [12..12])
  );
? str.join(c)
> "i paint modern"
```

### A pick-and-drop language

A small language of edits, run by a rud1 program. The text lies on a ring; the cursor stands in a
gap between two characters, at first the gap before the `p` (position 0). `nR` and `nL`
move the cursor `n` gaps right or left, `n-` picks up the `n` characters right of the
cursor, `+` drops what was picked up at the cursor; a missing `n` is 1. The number at the
end says where the result starts: the position of its first character on the ring. While
characters are picked up, the ring is that much shorter; a pick may run across position 0.

The program is read with `tpl_program`; the text is cut and joined with `tpl_cut`,
`tpl_cut3` and `tpl_join`, whose positions are given when they are called (`given:`).
Both programs shown are 20 characters long, the fewest a search found; the last line runs the first
on the text of the first subsection, with the same result.

```
[rud1]
my
( tpl_program= t{ops:(n:%[0-9]* op:%[RL\+\-])* end:$dec+}
, tpl_cut=     t{(*given: at) head:a[=at] tail:a*}
, tpl_cut3=    t{(*given: at k) head:a[=at] blk:a[=k] tail:a*}
, tpl_join=    t{head:a* blk:a* tail:a*}
);
my(count(n)= if("" == str(n), 1, uint(str(n))));
my
( run(prog, buf)=
  { my(len= #buf);
    mut my(s= buf, c= 0, hand= b"");
    my(p= unpack(tpl_program, prog));
    for
    ( o <- p.ops
    , my(k= count(o.n), op= str(o.op));
      if
      ( "R" == op, c= (c + k) % (len - #hand)
      , "L" == op, c= (c - k) % (len - #hand)
      , "-" == op
      , if
        ( c + k <= len
        , my(r= unpack(tpl_cut3, s, given: [at: c, k: k]));
          hand= r.blk;
          s= pack(tpl_join, [head: r.head, blk: b"", tail: r.tail]);
          c= c % (len - k)
        , my(m= c + k - len);
          my(r= unpack(tpl_cut3, s, given: [at: m, k: c - m]));
          hand= pack(tpl_join, [head: r.tail, blk: r.head, tail: b""]);
          s= r.blk;
          c= 0
        )
      , my(r= unpack(tpl_cut, s, given: [at: c]));
        s= pack(tpl_join, [head: r.head, blk: hand, tail: r.tail]);
        c= (c + #hand) % len;
        hand= b""
      )
    );
    my(r= unpack(tpl_cut, s, given: [at: p.end]));
    str(pack(tpl_join, [head: r.tail, blk: r.head, tail: b""]))
  }
);
for
( prog <- [b"R10-R+-L+3L9-L+6-L+2", b"R10-R+-L+2L7-R+4-R+3"]
, say(str(prog), "  ", run(prog, b"piet mondrian "))
);
say(run(b"R10-R+-L+3L9-L+6-L+2", b"abcdefghijklmn"));
```

Output:

```
R10-R+-L+3L9-L+6-L+2  i paint modern
R10-R+-L+2L7-R+4-R+3  i paint modern
knalbhdefgicjm
```

### The templates of this section

| Name | Template, and what it reads |
|---|---|
| `tpl_cut` | `t{(*given: at) head:a[=at] tail:a*}`<br>Cuts a text in two at the position `at`, given when it is called. |
| `tpl_cut3` | `t{(*given: at k) head:a[=at] blk:a[=k] tail:a*}`<br>Cuts a text in three: `at` bytes, then `k` bytes, then the rest; both numbers given when it is called. |
| `tpl_join` | `t{head:a* blk:a* tail:a*}`<br>Three texts one after the other; used to write them (`pack`). |
| `tpl_program` | `t{ops:(n:%[0-9]* op:%[RL\+\-])* end:$dec+}`<br>Reads a program of the pick-and-drop language: steps, each an optional count of digits and one of `R`, `L`, `+`, `-`, then the closing number. |
| `tpl_read` | `t{p0:a p1:a p2:a p3:a4 p7:a p8:a p9:a p10:a p11:a p12:a p13:a}`<br>Reads 14 bytes as eleven pieces, each named by its position; `p3` is four bytes. |
| `tpl_write` | `t{p10:a p13:a p0:a p11:a p1:a p7:a p3:a4 p8:a p2:a p9:a p12:a}`<br>The same eleven pieces in the order of the target; used to write them (`pack`). |

## Classic programs

From the `rosetta` directory of the rud1 tree, unchanged: `basic` is written as a coder new to the
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
for(i= 1, 100, say(fizzbuzz(i)));
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
  { mut my(s= [true] ~* (n + 1), p= 2);
    while
    ( p * p <= n
    , if(s[p], for(m <- [p * p .. n; p], s[m]= false));
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
my(gcd(a, b)= if(0 == b, a, __SELF__(b, a % b)));
say([gcd(1071, 462), gcd(462, 1071), gcd(17, 0), gcd(0, 0), gcd(2^64, 6^40)]);
```

Output:

```
[21,21,17,0,1099511627776]
```

### Fibonacci sequence (`rosetta/seasoned/Fibonacci_sequence.rud`)

```
[rud1]
/* Rosetta Code: Fibonacci sequence (fast doubling: about log2(n) steps) */
my
( fib2(n)=                                         \\ [F(n), F(n + 1)]
  if
  ( 0 == n
  , [0, 1]
  , my([a, b]= __SELF__(n // 2), c= a * (2 * b - a), d= a^2 + b^2);
    if(0 != n % 2, [d, c + d], [c, d])
  )
, fib(n)= fib2(n)[0]
);
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
( step(n)= if(0 != n % 2, 3 * n + 1, n / 2)
, hailstone(n)= if(1 == n, [1], [n] ~ __SELF__(step(n)))
, hlen(n)=
  { mut my(m= n, k= 1);
    while(1 != m, m= step(m); k+= 1);
    k
  }
, longest(N)=           /* [n, length] of the longest sequence with n <= N */
  { mut my(best= [1, 1]);
    for(n= 1, N, my(k= hlen(n)); if(k > best[1], best= [n, k]));
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
my(numbers(s)= [int(w) | w <- str.split(str.split(s, "\n")[0], " "), "" != w]);
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
