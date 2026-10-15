# oniguruma

Oniguruma regular expressions for sysl — the engine TextMate grammars are written for.

*An independent sysl package for Oniguruma. It is not affiliated with or endorsed by Oniguruma's authors or trademark holders.*

```
dependencies {
  oniguruma { git = "github.com/sysl-lang/oniguruma", version = "0.1.0" }
}
```

```sysl
import sh.sysl.oniguruma.{compile, describe, scanner}

main() -> Result[unit, string]
    val re = compile("(?<word>\\w+)@(\\h+)").map_err(describe)?

    re.find("ed@c0ffee").map_err(describe)? match
        Some(m) -> print(m.text(), m.group(1).unwrap(), m.group(2).unwrap())
        None -> print("no address")

    // A grammar rule's patterns, searched as one: leftmost position, then lowest index.
    val rules = scanner(["\\d+", "[a-z]+"]).map_err(describe)?
    val hit = rules.find_next("abc 123", 0, 0).map_err(describe)?.unwrap()

    print(hit.index, hit.found.text())     // 1 abc
    Ok(())
```

The module is `sh.sysl.oniguruma`. Oniguruma is **installed, not vendored** — `brew install
oniguruma`, Debian's `libonig-dev`, Arch's `oniguruma` — and found through `pkg-config`. Vendoring
in this organisation is a claim that the code runs on a board, and a highlighter never will. Built and
tested against Oniguruma 6.9.10 and sysl 0.1.0-alpha.4.

## The API

```sysl
compile(pattern: string) -> Result[&Regex, Error]
compile_with(pattern: string, o: Options, limits: Limits) -> Result[&Regex, Error]

struct Regex
    pattern: string
    capture_count(&self) -> usize
    group_numbers(&self, name: string) -> Buf[usize]
    is_match(&self, s: string) -> Result[bool, Error]
    find(&self, s: string) -> Result[Option[Match], Error]
    search(&self, s: string, from: usize, anchor: usize) -> Result[Option[Match], Error]
    match_at(&self, s: string, at: usize, anchor: usize) -> Result[Option[Match], Error]
    find_all(&self, s: string) -> Result[Buf[Match], Error]

struct Match
    group_count(self) -> usize          // group 0 included
    span(self, i: usize) -> Option[(usize, usize)]
    group(self, i: usize) -> Option[string]
    text(self) -> string
    start(self) -> usize
    end(self) -> usize

scanner(patterns: []const string) -> Result[&Scanner, Error]
scanner_with(patterns: []const string, o: Options, limits: Limits) -> Result[&Scanner, Error]

struct Scanner
    size(&self) -> usize
    find_next(&self, s: string, start: usize, anchor: usize) -> Result[Option[ScanMatch], Error]

struct ScanMatch
    index: usize                        // which pattern won
    found: Match

enum Error
    Pattern(message: string)            // Oniguruma's own sentence
    ScannerPattern(index: usize, message: string)
    RetryLimit                          // the backtracking budget ran out
    OutOfMemory
    Other(code: i32, message: string)

describe(e: Error) -> string
textmate() -> Options                   // the default
ascii() -> Options                      // \w \d \s \b and POSIX brackets all ASCII
default_limits() -> Limits              // 10,000,000 retries per position and per search
version() -> string                     // the linked library's
```

**Every offset is a byte offset into the UTF-8 string**, on a character boundary, so `s[m.start()..<m.end()]`
is the match. A caller holding UTF-16 positions (an editor) converts at its own edge. A `from` or
`start` inside a character is moved to the next boundary; a `match_at` inside a character answers
`None`.

**`\G` is the position a search begins at — that is how Oniguruma reads it.** `search`, `match_at`
and `find_next` take an `anchor` beside the position: where `anchor == from`, `\G` holds there; where
they differ, `\G` holds nowhere (`ONIG_OPTION_NOT_BEGIN_POSITION`). A tokenizer passes the previous
match's end as both on an ordinary step. `find_all` threads it: each search's `\G` is where the last
match ended.

`Regex` and `Scanner` are `&T` with destructors (`onig_free`, `onig_regset_free`, the match
parameters), so nothing is freed by hand. A loop compiling and searching 20,000 and 200,000 times
peaks at the same 2.6 MB resident.

## The dialect, and why it is the default

**`ONIG_SYNTAX_ONIGURUMA` with `ONIG_OPTION_CAPTURE_GROUP` — exactly what vscode-oniguruma compiles
with** (`createOnigScanner`, `options ?? [FindOption.CaptureGroup]`, `syntax ?? Syntax.Default`), the
binding every TextMate grammar in Visual Studio Code runs through, so what a grammar's author tested
against. `CAPTURE_GROUP` is the part that matters: without it, a pattern that names one group stops
numbering its plain `(…)` groups, and a grammar's `captures` table, which counts both, reads the
wrong text.

Under it `\w`, `\d`, `\s`, `\b` and the POSIX brackets are all **Unicode**; `^` and `$` are line
anchors; `(?m)` is Oniguruma's (and Ruby's) dot-matches-newline.

**`\b` is Unicode**, which is the part the Scala port's `021d89a` fixed: there is no boundary on
either side of a non-ASCII letter, mark, number or connector, so `\bif\b` does not fire inside
`síif` — `x\bÁ`, `a\b٣`, `a\bⅧ`, `a\b¹`, `a\b‿` all fail and `\bÁrbol` finds `Árbol` at byte 2.
`b_is_unicode_aware` asserts every line of it.

**`\w` is Unicode too, and here this package and the Scala port part company.** The port reads `\w`
as ASCII while `\b` stays Unicode, which is Ruby's rule — but Ruby's engine is Onigmo, whose `(?d)`
default splits the two. Oniguruma has no such setting: `ONIG_OPTION_WORD_IS_ASCII` moves `\w`, `\W`,
`\b` and `\B` together (`word_is_ascii_moves_w_and_b_together`), so no syntax and option set
reproduces "`\w` ASCII, `\b` Unicode". This package keeps vscode's reading; `Options.word_is_ascii`
is there for a caller who wants ASCII words and accepts ASCII boundaries with them.

## What is bound, and what is not

Bound: compiling (`onig_new`) with the error text (`onig_error_code_to_str`, the `OnigErrorInfo`
included, so an undefined name is quoted), searching and anchored matching under a retry budget
(`onig_search_with_param`, `onig_match_with_param`, `OnigMatchParam`), the region, named groups
(`onig_name_to_group_numbers`), capture counts, and the regset (`onig_regset_new`,
`onig_regset_search_with_param` with `ONIG_REGSET_POSITION_LEAD`, `onig_regset_get_region`).

Not bound, deliberately: encodings other than UTF-8 (a sysl string is UTF-8); the other syntaxes
(the extern for `OnigSyntaxRuby` is declared for a later option, not exposed); callouts and capture
history (no grammar uses them, and callouts would need a trampoline nobody has asked for);
`onig_scan`; `FIND_LONGEST`/`FIND_NOT_EMPTY` (a regset refuses them). Initialisation happens once,
on the first compile, and is not thread-safe — compile once before handing patterns to threads.

## The budget

Every pattern is searched under `Limits`: ten million retries per starting position (Oniguruma's
own default, so vscode-oniguruma's) and ten million per search (Oniguruma sets none). Running out is
`Err(RetryLimit)` — vscode reports the same event as no match. `(a+)+b` against forty `a`s answers
`RetryLimit` in tens of milliseconds; `the_search_budget_bounds_the_sum_of_the_positions` shows the
per-search limit catching a subject where no single position runs out.

## The tests

`sysl test .` — **219 passed**. They port every behavioural test of the user's pure-Scala Oniguruma
(`~/dev/oniguruma`, io.github.edadma:oniguruma 0.0.7) and assert them against the real engine:

| file | from | tests |
|---|---|---|
| `match_tests.sysl` | `MatchSpec` | 53 |
| `backref_tests.sysl` | `MatchBackrefSpec` | 18 |
| `lookaround_tests.sysl` | `MatchLookaroundSpec` | 24 |
| `subroutine_tests.sysl` | `MatchSubroutineSpec` | 13 |
| `property_tests.sysl` | `MatchPropertySpec` | 15 |
| `empty_loop_tests.sysl` | `MatchEmptyLoopSpec` | 19 |
| `flags_tests.sysl` | `MatchFlagsSpec`, and `FlagsSpec` where a binding can see it | 20 |
| `scanner_tests.sysl` | `TmScannerSpec`, `BashGrammarHangSpec`'s scanner sets, `InterruptSpec`'s intent | 26 |
| `corpus_tests.sysl` | `CorpusSpec`, `CompileCorpusSpec`, `BashGrammarHangSpec`'s corpus half | 8 |
| `api_tests.sysl` | this binding's own surface | 22 |
| `tests.sysl` | the helpers, and the library being linked | 1 |

**Where the real engine answers differently from the port, the test asserts the engine** and says so
beside it: a backreference past the pattern's groups (`(\3)|abc`, `\1`) is refused ("invalid backref
number/name") rather than never matching; `\g<0>` and `\g<-1>` with nothing consumed are refused
("never ending recursion"); `(?<=(\w+) )\w+` captures `o`, the shortest the lookbehind allows, not
`hello`; a capture inside a negative lookbehind is refused; a subroutine call's capture overwrites
its group (`(a)\g<1>` leaves group 1 at bytes 1–2); `(a*?)*` matches empty; `\w` matches `Á`.

**The corpus** — `corpus/textmate-corpus.txt`, 3,418 deduplicated patterns from 36 TextMate grammars
(SHA-256 `216235e3e415dfc78e09887a13f870799b042f0b8ba0bfd1b54d18592eb7dc85`), and
`corpus/bash-grammar-patterns.txt`, the bash grammar's 195 patterns in source order (SHA-256
`aa7061c6aabf80d060e65772129d9fa5c78840a5f806bde7f0bace6e3d6672f7`), both copied from the Scala
port's `jvm/src/test/resources` — **compiles 100%**: 3,394 as written, and the other 24 are all `end`
patterns whose `\N` names a group of their `begin` match, which compile once the reference is
resolved the way vscode-textmate's `resolveBackReferences` resolves it. Every pattern runs against its
own text; one (a Go grammar pattern exponential on its own text) runs out of budget, as it would in
vscode. No corpus or bash pattern takes more than 50 ms over the bug-report inputs, an anchored attempt
at every position.

**Not ported, because they test the Scala implementation's internals rather than matching**: the
`Parse*Spec` files (the parser's syntax tree), the `Compile*Spec` files (its bytecode),
`IntervalSetSpec` (its character-set arithmetic), `ParseCorpusSpec` (the parser over the corpus —
`every_corpus_pattern_compiles` is the binding's form of it) and most of `FlagsSpec` (a packed-int set;
the observable half is `every_switch_is_a_bit_of_its_own` and the switch tests). `InterruptSpec`'s
thread interruption has no counterpart in a C call; its intent — a pathological pattern does not hold
the thread — is the retry-budget tests.

**AddressSanitizer**: `SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test .` — 219 passed, no report.
This package is `pkg_config`, so **only the sysl half is instrumented**; Oniguruma itself is
Homebrew's uninstrumented build. The test binary carries `__asan_report_*` imports (`nm -u`), so the
sysl half was instrumented rather than assumed.

**The mutation pass**, each break made on a copy and the suite read: dropping `CAPTURE_GROUP` reddens
7 tests; an end offset one short, 153; `\G` never moved off the start, 3; the per-search limit
dropped, 1; the regset searched in regex-priority order, 5; the subject's end one byte past the
slice, 152; no character-boundary step at a mid-character `from`, 1. Searching the regset with
`ONIG_REGSET_REGEX_LEAD` instead of `POSITION_LEAD` reddens nothing but a budget test, the two leads
being the same rule computed in a different order.

## Licence

ISC for this package. Oniguruma itself is BSD-2-Clause and is not carried here.
