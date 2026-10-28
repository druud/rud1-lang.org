# rud1: not yet implemented

A program written from the spec can end in an error because the form is not built. This
file lists those forms by area, in the order of the spec: what the prototype says today,
and what to write meanwhile.

Three words are used with one meaning each:

- **built**: the prototype does it.
- **decided**: designed, a rule of the spec; it waits for a build.
- **proposed**: written in the spec, not confirmed by the author.

The Status column has two more: **backlog** (wanted, not designed) and **open** (the spec
lists the question as open, or two of its lines disagree). A rule that the spec states
without a mark counts as decided here. A dash in Status: the spec has no plan for the form
as written; the note in Today says more.

The prototype is the one written in Lean, rud1 version 0.1.66; there is no other implementation.
Its executable is named `rud1_run` and files end in `.rud`; `rud1 'code'` and
`rud1 file.rud` below stand for a shell function around it.

How to read the tables:

- Form is a whole program, run as `rud1 'code'`. Today is all it printed. Run from a file,
  a position `1:1` shows as `file.rud:1:1`.
- `L:C error …` is a compile error. `we exited <state> at L:C` is a run that ended in a
  failed result, with its message in the brackets. Both end with exit
  status 1.
- Meanwhile is a program that works today, with its result after `\\>`. A dash: nothing
  shorter than writing it out yourself.
- A form without a row is not thereby built: the list follows the audit of generation 30
  and the spec's own marks.

## Lexical and literals (spec §1)

| Form | Status | Today | Meanwhile |
|---|---|---|---|
| `0x1F` | — | `1:1 error malformed number '0x1F': a number in a base is \x{1F}` (stays an error, spec §1) | `\x{1F} \\ 31` |
| `[1, 2] >>+<< [3, 4]` | decided | `1:1 error not in the core yet: '>>+<<'` | a comprehension: Lists and numbers, (a) |
| `my(_<+>_ : _+_ = (a, b) -> a + b); 1 <+> 2` | decided | `1:5 error expected ')', got op '<'` | `my(plus(a, b)= a + b); plus(1, 2) \\ 3` |
| `my(x= 5); f"x = {x}"` | proposed | `1:11 error not in the core yet: prefixed literal 'f"…"'` | `my(x= 5); "x = " ~ str(x) \\ "x = 5"` |

The other hyper operators (`>>*`, `*<<`, `>>[`) give the same message with their own
token. A number in a base can be read by a template today (`$bin`, `$oct`, `$dec`, `$hex`).

## Declarations and types (spec §4, §6)

Declared types are built (generation 45): a declaration, a parameter, a return type
(`my(sint f(uint a)= a + 1)`) and a group type (`sint my(a= 1, b= 2)`) cast the value to the type,
along the promotion diamond. Where the compiler knows the kind of the value, one that does not
fit is a compile error (`my(uint x= -1)`); elsewhere it is an `error` when the program runs. What
the type pass of spec §10 adds beyond that (dispatch, variants, binding by type) is below.

| Form | Status | Today | Meanwhile |
|---|---|---|---|
| `type T = uint {default: none}; my(T dn); dn= 1` | decided | `1:16 error a range of a type takes whole numbers: got ident 'default'` (decided: a compile error at `my(T dn)`, the value is given at the declaration) | `my(dn= 1); dn \\ 1` |
| `my(x); x= 1; x` | decided | `1:8 error assignment to immutable 'x'` (late init) | `mut my(x); x= 1; x \\ 1` |
| `type Pct = uint {0..100}; my(f(uint a)= a); f(Pct(5))` | decided | `5` (the type is built, generation 65; a `Pct` passes where a `uint` is wanted until the type pass of spec §10 refuses it, H1) | — |
| `i64(5)` | decided | `1:1 error undeclared 'i64'` (also `f64`) | — |
| `#bool()` | decided | `1:2 error wrong number of arguments to 'bool'` (multis and index types, spec §6.4) | — |
| `my(r); state(r)` | decided | `1:8 error undeclared 'state'` | `my(r); is_defined(r) \\ 0` |
| `my(x= [1][5]); origin(x)` | decided | `1:16 error undeclared 'origin'` | — |

The check that `my(uint x= -1)` does not make, written out:

```
[rud1] my(x= uint(-1)); x  \\> we exited <error("uint(-1): -1 is negative")> at 1:7
```

## Functions (spec §4a, §8)

| Form | Status | Today | Meanwhile |
|---|---|---|---|
| `my(f(lazy v[...])= 1); f(1, 2)` | decided | `1:6 error not in the core yet: a lazy or sym rest parameter` | `my(f(v[...])= #v); f(1, 2) \\ 2` |
| `my(f(sym v[...])= 1); f(1, 2)` | proposed | `1:6 error not in the core yet: a lazy or sym rest parameter` | — |
| `my(f(lazy x= 1)= x); f()` | decided | `1:6 error not in the core yet: default on a lazy or sym parameter` | — |
| `my(f(lazy x)= x + x); f({say("once"); 2})` | decided | `once`<br>`once`<br>`4` (by name: decided is by need, memoized, and a compile error for an argument with an effect) | `my(f(x)= x + x); f({say("once"); 2})` prints `once`<br>`4` |
| `my(sq(x)= x^2); sq` | decided | `sq` (decided: `(x) -> x^2`) | — |
| `x -> x^2` | decided | `(lam (params x) (pow x 2))` (the core term) | — |
| `my(sint diff(x, y)= x - y); diff(3, 5)` | decided | `-2` (the return type is built; variants by return type are not) | `my(diff(x, y)= +x - y); diff(3, 5) \\ -2` |
| `my(f(sint n, str s)= s ~* n); f("ab", 2)` | decided | `1:33 error sint n: a str is no sint` (binding by type) | `my(f(sint n, str s)= s ~* n); f(s: "ab", n: 2) \\ "abab"` |
| `my(f(e)= 5); guard(error("x"), f)` | decided | `f` (the handler `f` is not called) | `guard(error("x"), 5) \\ 5` |
| `{ amend((r) -> 9); 5 }` | proposed | `1:3 error undeclared 'amend'` | — |
| `my(f(x, y)= x - y, g= f(_, 2)); g(5)` | open | `1:25 error undeclared '_'` | `my(f(x, y)= x - y, g= x -> f(x, 2)); g(5) \\ 3` |
| `-_(3)` | decided | `1:2 error undeclared '_'` (also `#_`, `!_`, `_[_]`) | `my(neg(x)= -x); neg(3) \\ -3` |
| `len(today())` | decided | `10` (decided: `10`, the top level's impurity inferred, spec §2, generation 53; a compile error under `policy.main.impure=1`) | — |
| `my(f()= len(today())); f()` | decided | `10` (decided: `10`, `f` inferred impure; a compile error under `policy.main.impure=1`) | — |
| `state.set(impure); impure my(f()= len(today())); f()` | decided | `1:1 error undeclared 'state'` | `my(f()= len(today())); f() \\ 10` |
| `my(main()= 144)` | decided | `main` (decided: `main()` runs, `144`; spec §5b, generation 37) | `my(main()= 144); main() \\ 144` |
| `my(at.main_start= () -> 1); 2` | decided | `1:6 error expected ')', got punct '.'` | — |

The names of the binary operators are built: `[rud1] _+_(1, 2) \\ 3`.

## Lists and numbers (spec §5, §7, §8.1)

| Form | Status | Today | Meanwhile |
|---|---|---|---|
| `[1, 2] + [10, 20]` | decided | `we exited <error("[1, 2] + [10, 20]: no '+' for a list and a list")> at 1:1` | \(a\) below |
| `[[1, 2], [3, 4]] * [[5, 6], [7, 8]]` | decided | `we exited <error("[[1, 2], [3, 4]] * [[5, 6], [7, 8]]: no '*' for a list and a list")> at 1:1` | \(b\) below |
| `my(L= [10, 20, 30]); L[1..]` | open | `1:27 error unexpected punct ']'` | `my(L= [10, 20, 30]); L[1..#L-1] \\ [20,30]` |
| `[0..^3]` | open | `1:6 error '^' needs a name, got uint '3'` | `[0..2] \\ [0,1,2]` |
| `my(v= [10, 20, 30]); v[^1]` | open | `1:25 error '^' needs a name, got uint '1'` | \(c\) below |
| `Map(["a", 42; "b", -1])` | open | `1:1 error undeclared 'Map'` | `my(m= [a: 42, b: -1]); m.a \\ 42` (a record: fixed names) |
| `3 - 5` | decided | `we exited <underflow> at 1:1` (the state built in generation 38; decided: under `{underflow: keep}` its payload is the `sint` result) | `+3 - 5 \\ -2` |
| `sint(5/2)` | open | `we exited <error("sint(5/2): a urat is not cast to a whole number")> at 1:1` (also `sint(2.5)`, `uint(2.5)`) | — |
| `2^0.5` | open | `we exited <error("2 ^ 0.500000000000000: the exponent is a whole number")> at 1:1` (a float exponent) | — |
| `-0.0` | decided | `0.00000000000000` (no negative zero: a bug of the prototype's float) | — |
| `oo // 2` | open | `we exited <error("not in the core yet: '//' with oo")> at 1:1` (also `%`, `^`, `sqrt`; the other operations on `oo` built in generation 55, spec §8.1) | — |
| `#[x \| x <- [1..oo]]` | open | `out of steps`, at `-s 100000` (the walk takes a step a cell; only a range without an end has the length `+oo`; a list made from it, a range of floats, a list of steps are walked) | `#[1..oo] > 1000 \\ 1` |
| `my(pow(str s, uint k)= s ~* k); "2"^3` | decided | `we exited <error("\"2\" ^ 3: no '^' for a str and a uint")> at 1:33` (dispatch through the lexical scope) | `"2" ~* 3 \\ "222"` |

- \(a\) `[rud1] my(a= [1, 2], b= [10, 20]); [a[i] + b[i] | i <- [0..#a-1]] \\ [11,22]`
- \(c\) `[rud1] my(v= [10, 20, 30]); [v[i] | i <- [0..#v-1], 1 != i] \\ [10,30]`

(b):

```
[rud1]
my
( A= [[1, 2], [3, 4]]
, B= [[5, 6], [7, 8]]
, N= #B
, P= #B[0]
);
[[sum([r[k] * B[k][j] | k <- [0..N-1]]) | j <- [0..P-1]] | r <- A]  \\> [[19,22],[43,50]]
```

The comprehensions with `#a-1` are an `error` for an empty list (`uint - uint` below 0).
§8.1 of the spec gives rules for `inf`; its §11 lists the `inf` algebra as open.

## Strings (spec §7a, §5a)

| Form | Status | Today | Meanwhile |
|---|---|---|---|
| `str.to_titlecase("o'neil")` | decided | `we exited <error("str.to_titlecase(\"o'neil\"): not built yet (it waits for the word boundaries of UAX #29)")> at 1:1` | `str.to_titlecase_first("o'neil") \\ "O'neil"` |
| `str.is_titlecase("A")` | decided | `we exited <error("str.is_titlecase(\"A\"): not built yet (it waits for the word boundaries of UAX #29)")> at 1:1` | — |
| `str.is_casefolded("a")` | decided | `we exited <error("str.is_casefolded(\"a\"): not built yet (it waits for normalization)")> at 1:1` | `"a" == str.to_casefold("a") \\ 1` (without normalization) |
| `"é" == str([101, 769])` | decided | `0` (code points; YA: `==` follows canonical equivalence, true) | — |
| `str.to_graphemes("x")` | decided | `1:1 error no function 'str.to_graphemes'` | `str.to_characters("x") \\ ["x"]` (code points) |
| `str.printf("%5d", 42)` | decided | `1:1 error no function 'str.printf'` (the format language: open) | `my(s= str(42)); " " ~* (5 - #s) ~ s \\ "   42"` |
| `printf("%d\n", 42)` | decided | `1:1 error undeclared 'printf'` (also `sayf`) | `say(42); \\ 42` |
| `str.pretty([1, 2])` | decided | `1:1 error no function 'str.pretty'` | `str.to_text([1, 2]) \\ "[1,2]"` |
| `str.substr("hello", 1, 2, "XX")` | decided | `1:1 error call of 'str.substr': takes 2 to 3 arguments, got 4` | `my(s= "hello"); str.substr(s, 0, 1) ~ "XX" ~ str.substr(s, 3) \\ "hXXlo"` |
| `my(f= str.is_cased); f("a")` | decided | `1:7 error undeclared 'str'` (a startup `str.` function as a value) | `my(f(s)= str.is_cased(s)); f("a") \\ 1` |
| `str.uc("a")` | proposed | `1:1 error no function 'str.uc'` (short names: a module, which needs `use`) | `alias {uc: to_uppercase}; str.uc("a") \\ "A"` |
| `re"a+"` | decided | `1:1 error not in the core yet: prefixed literal 're"…"'` (also `match`, `regex`: undeclared) | `unpack(t{%[^a]* "a"+ x*}, b"caab") \\ [b"c",[b"a",b"a"],b"b"]` (a template) |
| `to_characters("ab")` | decided | `1:1 error undeclared 'to_characters'` (dispatch on the type of the first argument; for the type pass) | `str.to_characters("ab") \\ ["a","b"]` |
| `uint.split(91, base: 2)` | open | `1:1 error no function 'uint.split'` | \(a\) below |

(a):

```
[rud1]
my(BASE= 2, digits(n)= if(n < BASE, [n], __SELF__(n // BASE) ~ [n % BASE]));
digits(91)  \\> [1,0,1,1,0,1,1]
```

## Input and output (spec §5a, §5c)

Built: `read_line()` on standard input, `say` and `print` on standard output, `exit(n)` and
`exit(n, msg)` (the message goes to standard error). Nothing else.

| Form | Status | Today | Meanwhile |
|---|---|---|---|
| `write(stdout, "x")` | open | `1:1 error undeclared 'write'` (handles, also `stderr`) | `print("x"); \\ x` |
| `stdout.say("x")` | open | `1:1 error undeclared 'stdout'` | `say("x"); \\ x` |
| `my(f= open("data.txt")); read(f)` | open | `1:7 error undeclared 'open'` | the shell: `rud1 file.rud < data.txt > out.txt`, and (a) below |
| `read_line(1)` | open | `1:1 error call of 'read_line': too many arguments` (the other read verbs, timeouts, layers: their names are open) | — |
| `now()`, `today()`, `time()`, `process.…` in a function not marked `impure`, or a hook | decided | (runs: effects are not inferred yet; decided: the function inferred impure, in a hook a compile error, spec §2, generation 53) | — |
| a mark from pure code: a fire-and-forget write to a logging pipe | decided | (no form to try; its name and the pipe are open) | — |

\(a\) Every line of standard input, here the three lines `3`, `4`, `5`:

```
[rud1]
mut my(total= 0);
mut my(line= read_line());
while
( is_defined(line)
, total+= sint.read(str.split(line, "\n")[0]);
  line= read_line()
);
total  \\> 12
```

## Templates (spec §5d)

Built: `unpack(t, b)` and `unpack(t, b, given: g)` on a `bytes` value, with every form that
the template parser reads; `pack(t, r)`, the simple version (generation 41, spec §5d "The writer's rules");
the template parser reads, and the `template` value (`t{…}`, `template(s)`, `~`, `~*`).
Sentences of the spec that still say "`unpack` is not built" are older than that build.

| Form | Status | Today | Meanwhile |
|---|---|---|---|
| the uncondensed tree (every item with position and length) | decided; how it is asked for: open | no form to try | — |
| `unpack(t{"\N{DIGIT ONE}"}, b"1")` | decided | `1:8 error template 1:1: \N{name} is not built yet (it waits for the table of names)` | `unpack(t{"\x{31}"}, b"1") \\ [b"1"]` |
| `unpack(t{&X}, b"e\x{cc}\x{81}")` | decided | `1:8 error template 1:1: the built-in pattern &X is not built yet` | `unpack(t{(?u: a2)}, b"e\x{cc}\x{81}") \\ ["é"]` (code points, counted by hand) |
| `unpack(b"C n", b"A\x{00}\x{01}")` | decided | `we exited <error("unpack(b\"C n\", b\"A\\x{00}\\x{01}\"): the first argument is not a template (template(s) makes one of a str)")> at 1:1` (a template in a `bytes` value) | `unpack(template(b"C n"), b"A\x{00}\x{01}") \\ [65,1]` |
| `unpack(q{C n}, b"A\x{00}\x{01}")` | decided | `we exited <error("unpack(\"C n\", b\"A\\x{00}\\x{01}\"): the first argument is not a template (template(s) makes one of a str)")> at 1:1` | `unpack(template(q{C n}), b"A\x{00}\x{01}") \\ [65,1]` |
| `my(hex= t{%[0-9a-f]}); unpack(t{&hex}, b"a")` | proposed | `1:31 error template 1:1: the pattern &hex is not defined` | `unpack(t{&hex ::= %[0-9a-f] ; &hex}, b"a") \\ [b"a"]` |
| `template(t{C}, ranking: 2)` | decided | `1:1 error wrong number of arguments to 'template'` (settings from a program; also `use template(…)`, a configuration) | "or else" between the variants: (a) below |
| `unpack(t{f}, b"\x{00}\x{00}\x{80}\x{7f}")` | — | `we exited <error("unpack(t{f}, b\"\\x{00}\\x{00}\\x{80}\\x{7f}\"): template 1:1: 'f' on an infinity or a NaN: not read yet")> at 1:1` (a float that is an infinity or a NaN) | — |
| hooks (a function on a group or a brace pair) | open: not designed | no form to try | — |
| `P"%d"` | open | `1:1 error not in the core yet: prefixed literal 'P"…"'` (formats as templates: ideas, not decided) | — |

\(a\) Two variants that can read the same text are a compile error while ranking is
strict; `||` ("or else") says that the first one wins:

```
[rud1] unpack(t{&hex ::= %[0-9A-Fa-f]+ ; &dec ::= %[0-9]+ ; &hex | &dec}, b"15")
\\ 1:8 error template 1:45: variants 1 and 2 can read the same text, "1": make one of them narrower, or write '||' where the first is to win
```

```
[rud1] unpack(t{&hex ::= %[0-9A-Fa-f]+ ; &dec ::= %[0-9]+ ; &hex || &dec}, b"15")  \\> [[b"15"]]
```

## Modules (spec §9a, §9b)

All of §9a is proposed. A program is one file; the startup module (`sum`, `max`,
`say`, …) is the only module, and it is always there.

| Form | Status | Today | Meanwhile |
|---|---|---|---|
| `my(nt= use("numtheory")); nt` | proposed | `1:8 error undeclared 'use'` | the functions in the same file |
| `divisors(12)` | decided | `1:1 error undeclared 'divisors'` (a name of gp) | \(a\) below |
| `CORE.say("hi")` | decided | `1:1 error undeclared 'CORE'` (spec §9a, generation 37: `CORE.x` names the startup binding) | `say("hi") \\ hi` |
| `__NAME__` | proposed | `1:1 error undeclared '__NAME__'` (also `__MAIN__`, `__DIE__`) | — |
| `__CORE__.sum([1, 2])` | proposed | `1:1 error undeclared '__CORE__'` | `sum([1, 2]) \\ 3` |
| `CORE.exit_status()` | decided | `1:1 error undeclared 'CORE'` | `{ defer({ say("status ", exit_status()) }); exit(3) } \\ status 3` (exit status 3) |
| `exit.max` | decided | `we exited <error("exit.max: a function has no named parts: only a list has")> at 1:1` (the largest exit status, a constant of the host) | — |
| `naming(p_= {private}); 1` | proposed | `1:1 error undeclared 'naming'` | — |
| `const(1 + 2)` | proposed | `1:1 error undeclared 'const'` | — |

- \(a\) `[rud1] my(divisors(n)= [d | d <- [1..n], 0 == n % d]); divisors(12) \\ [1,2,3,4,6,12]`

Certified modules and capabilities (spec §9b) are proposed and have no form to try
beyond `use`.

## Policies and configuration (spec §8, §9a)

The spec lets a policy (of a company, of a project) turn a warning into an error or into
silence. The switches, the configuration file (its format is open) and strict mode (open)
are not built: `rud1_run` has no option for one (`rud1_run -h` lists
`-c`, `-e`, `-f`) and no form in a program sets one. Every check has its default. The one
policy that can be asked for today stands in the header of a template:

```
[rud1] unpack(t{%[.]}, b".")
\\ 1:8 error template 1:3: the byte '.' of a set is escaped (\.) or stands in an inner bracket ([.])
```

```
[rud1] unpack(t{(*policy.punctuation.bare: 2) %[.]}, b".")  \\> [b"."]
```

## Concurrency (spec §9a)

Proposed, all of it: `async` and `await`, the state `pending`, the parallel
forms with gp's names.

| Form | Status | Today | Meanwhile |
|---|---|---|---|
| `async(1)` | proposed | `1:1 error undeclared 'async'` (also `await`) | — |
| `parapply(x -> x^2, [1, 2, 3])` | proposed | `1:1 error undeclared 'parapply'` (also `parvector`, `parsum`, `parselect`) | \(a\) below |
| `parfor(i <- [1..3], say(i))` | proposed | `1:1 error undeclared 'parfor'` | `for(i <- [1..3], say(i)); 0` prints `1`<br>`2`<br>`3`<br>`0` |

- \(a\) `[rud1] [x^2 | x <- [1, 2, 3]] \\ [1,4,9]`

## The symbolic layer (spec §9)

§9 states it; §11 lists the quoting as open ("tentative").

| Form | Status | Today | Meanwhile |
|---|---|---|---|
| `'(1 + 2)` | open | `1:1 error not in the core yet: '''` (a quoted expression; also `'x`) | — |
| `my(y= 1); $y` | open | `1:11 error not in the core yet: '$'` (a splice) | — |
| `eval(1)` | — | `1:1 error undeclared 'eval'` (decided: no `eval`) | a function value: `my(f()= 1); f() \\ 1` |
| `my(f(sym e)= e[0]); f(1 + 2)` | decided | `we exited <error("3[0]: a uint has no positions")> at 1:14`<br>`  in f at 1:21` (a `sym` parameter is by name; its expression can't be inspected) | `my(f(sym e)= e); f(1 + 2) \\ 3` |
| `x^2 + 1` | open | `1:1 error undeclared 'x'` (a free variable; polynomials are to be a library) | `my(p(x)= x^2 + 1); p(3) \\ 10` |

## Tools (spec §10)

| Form | Status | Today | Meanwhile |
|---|---|---|---|
| a compiler mode that writes the standard version of a program | backlog | no such mode | `rud1_run -c`: the core term, (a) below |
| a tool that finds holes in templates (two templates that read one text differently, bytes that do not come back) | backlog: not designed | nothing | the compile error for two variants that read one text (Templates, (a)) |
| a REPL (spec §4a: `x= 1` at its top level declares) | decided | no executable | `rud1 'code'` |

\(a\) `rud1_run -c -e '1 + 2'`:

```
[rud1] 1 + 2
\\ (add 1 2)
\\ 3
```

## Built, but different from the spec today

None of the confirmed bugs of the audit of generation 30 is left.

Repaired in generation 37: output lost on a full disk is told (spec §5a).
`rud1_run -e 'say("hi"); 5' > /dev/full` gave status 0 and nothing on `stderr`; now
`rud1_run: cannot write standard output: resource exhausted (error code: 28, no space left on device)` and status 1 (an `exit(n)` with n not 0 keeps its status). A name twice in a
destructuring assignment is a compile error (spec §2):\
`[rud1] mut my(a= 0); [a, a]= [3, 4]; a \\ 1:15 error name 'a' twice in one destructuring assignment`

Repaired in generation 32: `defer` evaluates the strict arguments of its call when the
`defer` is met (spec §8, "Go rule").\
`[rud1] my(f(x)= say("f ", x)); mut my(a= 1); { defer(f(a)); a= 2; say("body") }; 0`\
`\\ body`\
`\\ f 1`\
`\\ 0`

Repaired in generation 31 (the other ten items of this list as it stood after the audit),
each with what it prints now:

- Two spreads in one call are a compile error (spec §4a).\
  `[rud1] my(f(a, b, c)= [a, b, c]); f([1][...], 0, [3][...])`\
  `\\ 1:43 error a second spread in one call: at most one (a list takes any number: call(f, [L[...], M[...]]))`
- The same name twice in one destructuring declaration is a compile error (spec §4).\
  `[rud1] my([q, q]= [1, 2]); q \\ 1:1 error duplicate declaration 'q'`
- A template can be made of a `bytes` value (spec §5d: the way to names that are not
  UTF-8).\
  `[rud1] template(b"C n") \\ t{C n}`
- A pattern that is recursive through another pattern can be a variant.\
  `[rud1] unpack(t{&val ::= &arr | $dec+ ; &arr ::= "[" &val "]" ; &val}, b"7") \\ [[7]]`
- A repeat of a pattern inside itself ends.\
  `[rud1] unpack(t{&p ::= "(" &p* ")" ; &p}, b"()") \\ [[b"(",[],b")"]]`
- A failure deep inside a recursive pattern is final: once a variant or a round has read
  bytes and then enters a recursive pattern, nothing else takes over.\
  `[rud1] unpack(t{&r ::= "(" &r ")" | "a" ; &r || x*}, b"((a")`\
  `\\ we exited <error("unpack(t{&r ::= \"(\" &r \")\" | \"a\" ; &r || x*}, b\"((a\"): template 1:15: the data ends inside the literal \")\": 1 wanted at byte 3, 0 left")> at 1:1`
- A failure after `(*commit)` is final out of alternatives and repeats too.\
  `[rud1] unpack(t{"a" (*commit) ("b" | "c") || x*}, b"ad")`\
  `\\ we exited <error("unpack(t{\"a\" (*commit) (\"b\" | \"c\") || x*}, b\"ad\"): template 1:16: the literal \"b\" is not at byte 1: \"d\"")> at 1:1`
- A size on a pattern that is one set is applied.\
  `[rud1] unpack(t{&[3]d}, b"1")`\
  `\\ we exited <error("unpack(t{&[3]d}, b\"1\"): template 1:1: the size 3 at byte 0 goes beyond the end, 1 left")> at 1:1`
- A value of the call used as a count is a whole number.\
  `[rud1] unpack(t{(*given: at) @[=at] x*}, b"hello", given: [at: "abc"])`\
  `\\ we exited <error("unpack(t{(*given: at) @[=at] x*}, b\"hello\", given: [at: \"abc\"]): template 1:14: =at is a value of the call that is no whole number")> at 1:1`
- Inside `(?u: … )` the room that `*` leaves for what follows is counted in code points
  where the items count them.\
  `[rud1] unpack(t{(?u: s:a* t:a2)}, b"ab\x{c3}\x{a9}\x{c3}\x{a9}") \\ [s:"ab",t:"éé"]`

## Known limits of the prototype

- Fuel and steps. A run has a budget of evaluation depth, 100000 by default, and one of steps,
  1000000000 by default: a round of a loop and a call take a step each. A loop and a chain of tail
  calls take no depth; a recursion that is not a tail call does, and about 15000 levels of it use
  the budget up, and what was written before is lost with it.\
  `[rud1] say("start"); my(f(n)= if(0 == n, 0, 1 + __SELF__(n - 1))); f(20000) \\ out of fuel`
- `rud1_run -f N` sets the depth, `rud1_run -s N` the steps. The same program with `-f 1000000`:\
  `[rud1] say("start"); my(f(n)= if(0 == n, 0, 1 + __SELF__(n - 1))); f(20000)`\
  `\\ start`\
  `\\ 20000`
- Indexing a list is linear in the index, and a list made by a range or a
  comprehension is by name: it is computed again at each use (decided: memoized).
  So a loop over the indices is quadratic.\
  `[rud1] my(N= 2000, L= [1..N]); mut my(s= 0); for(i <- [0..N-1], s+= L[i]); s \\ 2001000`
- A walk is linear.\
  `[rud1] my(N= 2000, L= [1..N]); mut my(s= 0); for(x <- L, s+= x); s \\ 2001000`\
  On the machine of this run the first took 6.1 s and the second 0.02 s; with
  `N= 4000` the first took 22 s.
- `read_line()` in a loop is quadratic in the number of lines. On 4000 lines
  `abcdefghij` (`yes abcdefghij | head -4000`):\
  `[rud1] mut my(n= 0, l= read_line()); while(is_defined(l), n+= 1; l= read_line()); n \\ 4000`\
  That took 0.17 s; 8000 lines 0.60 s, 16000 lines 2.16 s.
- Not run again here, from the audit: no end within 20 s for 200000 lines; 35 s for a
  `b"…"` literal of 240000 bytes. From the Handover: `rud1_run` writes its output after
  the run and fetches standard input whole at the first read, so it is not interactive.
- Since generation 38 a `uint` below 0 is the state `underflow`, not an `error`; it carries no
  message (under `{underflow: keep}`, not built, the `sint` result).\
  `[rud1] 3 - 5 \\ we exited <underflow> at 1:1`
- Some origins are terms of the core, not places in the source.\
  `[rud1] my(v= [10, 20, 30, 40]); v[0, 9] \\ we exited <error("[10, 20, 30, 40][9]: index 9 is outside 0 .. 3")> at (index #0 #2)`
- A column counts bytes, not characters: this `zz` is the sixth character.\
  `[rud1] "é"; zz \\ 1:7 error undeclared 'zz'`
- Values: a float shows 15 digits; there is no negative zero; a function shows as its
  name or as a core term (rows above).
