# Proposed syntax extensions to VERS

This document proposes a set of extensions to the
[VERS](https://github.com/package-url/vers-spec) version range notation. The
goal is to make VERS capable of expressing every range primitive defined in
[dag-range-primitives.md](dag-range-primitives.md) that describes a set of
versions self-containedly. Two of the primitives' concepts are deliberately
out of scope: labels (`latest` — resolved against mutable registry state
that is part of neither the expression nor the version list, so no
self-contained syntax can express them) and positional nodes (decidable
from the version list, but they make membership context-dependent — whether
a version matches depends on which other versions exist — and no use case
here requires them).

The extensions accept only canonical forms: every constraint is an explicit
comparator followed by a version, so equality is written `=1.2.3` — the bare
`1.2.3` shorthand of current VERS is dropped. They are built on one
deliberate semantic change — the comparator list becomes a conjunction
instead of a list of interval boundaries — which alters the meaning of some
existing expressions. See
[Backward compatibility](#backward-compatibility) for the details and the
migration rule.

This is a working draft for discussion — not a finalized proposal.

## Background

Current VERS expresses a range as:

```
vers:<scheme>/<comparator-list>
```

where the comparator list is a `|`-separated sequence of constraints. The
current specification gives this list **interval semantics**: the constraints
are sorted by version and treated as boundaries of alternating intervals, and
a version matches if it is contained within any of those intervals. For
example, `vers:semver/>=1.1.0|<1.3.0|>=2.0.0|<3.0.0` matches the two disjoint
intervals `[1.1.0, 1.3.0)` and `[2.0.0, 3.0.0)`.

This proposal **reinterprets the comparator list as a conjunction**: a version
matches a list if and only if it satisfies every constraint in it. Union is
expressed explicitly with parenthesized blocks (Extension 1) instead of being
implied by the interval pairing. The motivation: once cuts, filters,
intersection, and subtraction enter the notation, the implicit pairing rule
no longer has a well-defined meaning, while a conjunction gives every
constraint an independent one. This is a breaking semantic change; see
[Backward compatibility](#backward-compatibility).

Even with its interval semantics, current VERS cannot express:

- Ranges that span multiple version schemes
- Branch-precise segments — bounds at series boundaries that stay correct at
  the pre-release boundary (infima/cuts), needed wherever a range crosses a
  fork
- Stable-only filters
- Intersection of independently authored constraints
- Sub-range exclusion as a composable operator (today the author must
  manually rewrite the surrounding range into the intervals around the gap)

Each of these gaps is addressed below. A design principle runs through all of
them: an expression must be **self-contained** — decidable against a flat
list of version strings, without access to the underlying repository —
wherever the version scheme's structure permits it (see "Forks and branches"
in [dag-range-primitives.md](dag-range-primitives.md)). Branch membership is
therefore expressed through the scheme's own structure (cuts), not through a
separate fork construct that only a graph-aware tool could evaluate.

## Extension 1: Union blocks

A range may be the union of several constraint blocks. Each block is wrapped
in parentheses, names its own scheme, and is evaluated independently; the
blocks are joined with `|`:

```
vers:(<scheme>/<comparator-list>)|(<scheme>/<comparator-list>)
```

A version matches the expression if it matches any one of the parenthesized
blocks. The schemes of two blocks may differ — joining incompatible version
spaces — or repeat, uniting disjoint sub-ranges within a single scheme. The
repeated-scheme form replaces the implicit interval pairing of current VERS
(see [Backward compatibility](#backward-compatibility)).

A single-scheme expression is written without parentheses, exactly as today:

```
vers:semver/>=1.1.0|<2.0.0
```

The parenthesized form is defined only for unions of two or more blocks; a
union of one block is the bare scheme-block, so `vers:(semver/>=1.1.0|<2.0.0)`
is not a valid expression.

**Example — version scheme switch (calver → semver):**

A vulnerability spans a transition from a date-based scheme to semver. Every
calver release is affected; the fix exists only in the semver line at `1.4.0`.

```
vers:(calver-ym/>=2021.01)|(semver/>=1.0.0|<1.4.0)
```

**Example — union of two disjoint ranges in one scheme:**

The two disjoint intervals of Example 3, written as two blocks under the same
scheme:

```
vers:(semver/>=1.1.0|<1.3.0)|(semver/>=2.0.0|<3.0.0)
```

**Example — Debian epoch bump:**

A vulnerability spans two epoch series; the fix is `2:1.2.0-1`.

```
vers:(deb/>=1:0.9.0-1|<2:0.0.1-1)|(deb/>=2:0.0.1-1|<2:1.2.0-1)
```

Note: because the `deb` scheme orders epochs before everything else, its
ordering is total across the bump and the same range is expressible as a
single block: `vers:deb/>=1:0.9.0-1|<2:1.2.0-1`. The two-block form merely
makes the structural split explicit; either is valid.

**Example — Erlang/OTP CVE (scheme switch + parallel branches):**

A vulnerability affects the entire R-series (never fixed), the numeric main
line from `17.0`, and three maintenance branches, each fixed independently
(Example 4):

```
vers:(otp-r-series/>=R13B)|(otp/>=17.0|<26.0)|(otp/>=26.0|<26.2.5)|(otp/>=27.0|<27.1.1)|(otp/>=28.0|<28.0.1)
```

The R-series and numeric schemes are incomparable, and each maintenance
branch carries its own fix bound. Because OTP version numbers encode the
branch, every block is decidable from version strings alone.

## Extension 2: Infimum

Some upper bounds cannot be expressed as a concrete version without ambiguity
at the pre-release boundary. The infimum of a version `v` is a cut in the
scheme's ordering (see
[dag-range-primitives.md](dag-range-primitives.md)): the position immediately
below the lowest pre-release of `v`. A bound against the cut is written by
prefixing an ordering comparator with `$`: `$<v` excludes all pre-releases
of `v` while including all versions that sort below them, and `$>=v` starts
at the beginning of the `v` family, including its pre-releases. Because no
version is ever equal to a cut, `$>=v` and `$>v` (and likewise `$<v` and
`$<=v`) coincide. `$` is invalid on `=`, `!=`, and `*`.

```
$<2.0.0
$>=1.4.0
```

The `$` precedes the comparator, not the version: everything after the
comparator remains fully opaque to the notation, so even a scheme whose
version strings contained a `$` would introduce no ambiguity.

A pair of cuts expresses a whole release series — and with it, a branch, for
any scheme that encodes branches in the version structure: `$>=1.4.0|$<1.5.0`
is exactly the `1.4.x` series, past, present, and future. This is how ranges
that cross forks stay self-contained: branch membership is decided by the
version string, not by querying a repository.

The `$` modifier is a notation-level concept; which cut it denotes is
defined by the version scheme, not the notation. A scheme that supports
cuts must specify, for every version `v`, the position of the cut of `v`
in its ordering: semver places it immediately before the lowest pre-release
of `v`; debian places it below the `~`-suffixed variants of `v`. The
notation imposes only one requirement — no version is ever equal to a cut.
A scheme may place the cut of `v` immediately below `v` itself, in which
case `$<v` coincides with `<v` and `$>=v` with `>=v`. A scheme that does
not define cut semantics must reject `$` as invalid for that scheme.

The character `$` is chosen because it requires no escaping in URIs and —
unlike `^` or `~` — carries no conflicting meaning in existing range
notations (caret and tilde ranges).

**Example — Elixir `~> 1.3`:**

`~> 1.3` means `>= 1.3.0` and less than any pre-release of `2.0.0`:

```
vers:semver/>=1.3.0|$<2.0.0
```

`1.4.0-rc.1` is included (within the segment interior); `2.0.0-beta.1` is
excluded (at or beyond the cut of `2.0.0`).

**Example — Elixir `~> 1.3-beta`:**

The lower bound is a pre-release, which admits `1.3.0-beta` itself:

```
vers:semver/>=1.3.0-beta|$<2.0.0
```

**Example — abandoned branch, affected without fix (Example 9):**

A CVE is introduced at `1.3.0`. The `1.3.x` branch was fixed at `1.3.5`; the
`1.4.x` branch was abandoned with no fix — every `1.4.x` version, including
future patches, is affected:

```
vers:(semver/>=1.3.0|<1.3.5)|(semver/$>=1.4.0|$<1.5.0)
```

The second block is the full `1.4` series between two cuts. An open-ended
`>=1.4.0` would be wrong — it would spill past the branch into `1.5.0` and
beyond, which are on the main line and out of scope.

## Extension 3: Filter

A filter is a scheme-defined predicate applied to a segment. A version within
the segment's bounds is excluded from the range unless it satisfies the
predicate.

The syntax is a `#`-prefixed, comma-separated list of filters appended to the
comparator list:

```
vers:<scheme>/<comparator-list>|#<filter>,<filter>
```

The `#` sigil follows purl precedent, where `#` separates the subpath
component. A VERS, like a purl, is an identifier in URI form rather than a
URL to be dereferenced; tools consume the whole string as the range
expression instead of splitting it at the fragment delimiter.

Filters are defined by the version scheme, not the notation. To the notation
a filter is an opaque string (the same lexical rule as a version): the scheme
decides what predicate it denotes, which filters it supports, and even
whether a filter has internal structure of its own. A plain keyword like
`stable` is the common case, but a scheme could define parameterized filters
with their own micro-syntax (in the spirit of CSS `nth-child`); the notation
passes the string through uninterpreted, with structural characters
percent-encoded as in versions. To keep expressions self-contained, a
filter's predicate must be decidable from the version string alone.

For example, the `semver` scheme defines `#stable` to mean: exclude all
versions with a `-` pre-release identifier.

When multiple filters are given, a version must satisfy every one of them.
The filters form a single comma-separated `#` group; a comparator list
contains at most one such group. For example, a `node` scheme could
define `lts` as "even major number" — decidable from the version string, like
`stable`:

```
vers:node/>=18.0.0|#stable,lts
```

Filters are per-segment. In a multi-scheme expression, each scheme block may
carry its own filter:

```
vers:(semver/>=2.0.0|#stable)|(calver-ym/>=2021.01|#stable)
```

**Example — open-ended prospective range, stable only:**

```
vers:semver/>=2.0.0|#stable
```

All stable releases from `2.0.0` onward, excluding pre-releases.

**Example — single segment with filter:**

```
vers:semver/>=1.1.0|<2.0.0|#stable
```

Equivalent to the current VERS behavior in ecosystems that exclude pre-releases
by default, but now expressed explicitly.

## Extension 4: Set-minus (complement)

A range may exclude a sub-range using the `\` operator:

```
vers:<scheme>/<left-comparator-list>\<right-comparator-list>
```

The result is every version matched by the left side that is not matched by the
right side. The `\` operator has lower precedence than `|`, so each side's
comparator list is parsed in full before the subtraction is applied.

Multiple subtractions chain left-associatively:

```
A\B\C  ≡  (A\B)\C
```

**Example — exclude a specific bad version:**

```
vers:semver/>=1.0.0|<2.0.0\=1.2.3
```

All `1.x` releases except `1.2.3`. This is equivalent to the existing
`vers:semver/>=1.0.0|!=1.2.3|<2.0.0` — both forms are valid.

**Example — exclude a sub-range:**

```
vers:semver/>=1.0.0|<3.0.0\>=1.2.0|<1.3.0
```

All `1.x` and `2.x` releases except the broken `1.2.x` range.

**Example — implicit universe with exceptions (Example 17):**

All versions are affected by default; the fixes are `1.2.3` on the `1.x`
branch and `2.0.1` on the `2.x` line, each unaffected from that point onward
on its own branch:

```
vers:semver/*\>=1.2.3|$<2.0.0\>=2.0.1
```

The universe `*` minus the `1.x` fixed range `[1.2.3, inf(2.0.0))` minus
everything from `2.0.1` onward. The `$<2.0.0` bound caps the first exception
at the end of the `1.x` line so that it does not swallow `2.0.0`, which is
affected until `2.0.1`.

## Intersecting ranges

There is no intersection operator, because the comparator list already is
one: the list is a conjunction, so the intersection of two segments is the
concatenation of their comparator lists. The primary use case is combining
independently authored prospective ranges — each author writes their own
constraint, and a version must satisfy all of them simultaneously.

**Example — two prospective constraints:**

Package A requires `~> 1.3` (expressed as `>=1.3.0|$<2.0.0`) and package B
requires `~> 1.6` (expressed as `>=1.6.0|$<2.0.0`). The combined requirement
is the concatenation of the two lists:

```
vers:semver/>=1.3.0|$<2.0.0|>=1.6.0|$<2.0.0
```

A scheme-aware normalizer reduces this to `vers:semver/>=1.6.0|$<2.0.0`.

**Example — safe and compatible (Example 8):**

Versions of `foo` that are compatible (`>=2.0.0|<3.0.0`) and not in the
vulnerable range (`>=2.1.0|<2.2.0`). Example 8 phrases this as an
intersection with a complement, A ∩ (U \ B); since A ∩ (U \ B) = A \ B, the
notation expresses it directly with set-minus (Extension 4):

```
vers:semver/>=2.0.0|<3.0.0\>=2.1.0|<2.2.0
```

The result is `2.0.x` and `2.2.0`–`2.9.x`.

**Whole expressions:** intersection distributes over the union of blocks:
pair every block of one expression with every block of the other; the
result is the union of the non-empty pairwise intersections. Two blocks
under different schemes intersect to the empty set (a version has identity
in exactly one scheme) and are dropped — so schemes present in only one
input vanish. Two blocks under the same scheme intersect by concatenating
their positive comparator lists — merging their filter groups into one
comma-separated `#` group — and keeping the subtractions of both:

```
(A\B) ∩ (C\D)  =  (A|C)\B\D

((semver/A)|(git/B)) ∩ ((semver/C)|(git/D))  =  (semver/A|C)|(git/B|D)
```

where `A|C` stands for the concatenation of the two comparator lists. When
both inputs contain several blocks of the same scheme, the distribution
yields one block per same-scheme pair. A tool combining independently
authored ranges applies this distribution when constructing the expression.

## Scheme evaluability

The extensions above are designed so that an expression is **self-contained**:
membership is decidable from the expression and a flat list of version
strings alone. Whether that is achievable is a property of the version
scheme — and it is two properties, not one. The scheme registry must record
both:

- **Ordering**: does the scheme define a total order on version strings,
  decidable from the strings alone?
- **Series structure**: does the version string encode which series
  (branch) a version belongs to — equivalently, does the scheme define
  cuts (`$`)?

The combinations give three kinds of scheme:

- **Structure-ordered schemes** (`semver`, `deb`, `otp`, ...) — ordered,
  with series structure: all comparators and cuts are decidable from
  strings alone. Ranges can be branch-precise and self-contained at the
  same time — cuts express the series boundaries, and a range that crosses
  a fork is a union of per-branch segments. Ecosystems that track parallel
  branches outside the version string (e.g. Debian suites) also fall here:
  the branch dimension lives in the package identifier, one range per
  stream (see "Forks and branches" in
  [dag-range-primitives.md](dag-range-primitives.md)).

- **Flat-ordered schemes** (`calver-ym`, serial build numbers, ...) —
  ordered, without series structure: all comparators are decidable from
  strings alone and expressions are self-contained, but no expression can
  be branch-scoped — the scheme rejects `$` and the version string carries
  no branch information. If the project's history is linear, nothing is
  lost. If it forks, an interval over this scheme linearizes the fork:
  releases from parallel branches interleave in the order, and a bound
  meant for one branch silently captures the other branch's releases. No
  evaluator can detect this — the expression is valid and evaluates
  deterministically; the result is wrong about the graph, not about the
  order. The remedies lie outside the expression — the package identifier
  is not part of this notation: enumerate the affected versions explicitly,
  separate the streams at the identifier level (as purl does), or switch to
  a version scheme that encodes the branch.

- **Graph-ordered schemes** (`git` commit hashes and tags) — no
  string-decidable order: the only order is the graph's own partial order,
  reachability. Ordering comparators are meaningful — `>introsha|<patchsha`
  is the range "descendants of the introducing commit that are ancestors
  of the fix" (the order is partial: a commit on a side branch is neither,
  and satisfies no bound) — but evaluating membership requires the
  repository. Only `=`, `!=`, and `*` are decidable from strings alone
  (identity comparisons). Cuts are invalid outright: a graph carries no
  series structure for a cut to denote, so the scheme defines no infimum
  semantics and rejects `$` (Extension 2).

A tool that encounters ordering comparators over a graph-ordered scheme and
has no graph access MUST report the range as not evaluable. It must not fall
back to lexicographic or timestamp ordering — silent linearization produces
wrong answers. Authors who need flat-list evaluability over a graph-ordered
scheme can enumerate the affected identifiers explicitly with `=` constraints;
retrospective ranges are finite, so enumeration is always possible.

## Operator precedence summary

From highest to lowest:

| Level | Operator | Meaning |
|-------|----------|---------|
| 1 | `<1.2.3`, `>=1.2.3`, `!=1.2.3`, `$<1.2.3`, `#stable` | Individual comparators and the filter group |
| 2 | `\|` | AND (comparator list within a segment) |
| 3 | `\` | Set-minus (complement) |
| 4 | `(scheme/...)\|(scheme/...)` | Union of blocks |

## Grammar (ABNF)

```
expression      = "vers:" ( block-union / scheme-block )
block-union     = "(" scheme-block ")" 1*( "|" "(" scheme-block ")" )
scheme-block    = scheme "/" complement-expr
complement-expr = comparator-list *( "\" comparator-list )
comparator-list = constraints [ "|" filters ]
constraints     = "*" / version-comparator *( "|" version-comparator )
version-comparator = [ "$" ] ( "<" / "<=" / ">" / ">=" ) version
                   / ( "=" / "!=" ) version
filters         = "#" filter *( "," filter )
scheme          = ALPHA *( ALPHA / DIGIT / "." / "-" )
version         = 1*( safe-char / pct-encoded )
filter          = 1*( safe-char / pct-encoded )
safe-char       = ALPHA / DIGIT / "." / "_" / "-" / "~" / "+" / ":"
pct-encoded     = "%" HEXDIG HEXDIG
```

`ALPHA`, `DIGIT`, and `HEXDIG` are the RFC 5234 core rules. The `scheme` is
case-insensitive with lowercase as the canonical form, matching the current
specification.

`version` and `filter` share one lexical rule: any character outside the
safe set — including every structural character of this notation (`|`, `/`,
`(`, `)`, `\`, `#`, `,`, `$`, `*`, `%`, and the comparator characters
`<`, `>`, `=`, `!`) — must be percent-encoded. Tokenization therefore never
depends on the version scheme's syntax: together with `$` preceding the
comparator, a parser can split any expression into tokens without knowing
the scheme. The safe set covers the characters real schemes use in practice
(semver's `.-+`, deb's `:~`, alpine's `_`), so percent-encoding is the escape
hatch, not the common case.

The grammar enforces a stratification. Within a scheme block, a
`comparator-list` is semantically a single segment — conjoined bounds
describe one interval; `!=` and filters punch holes in it — so the positive
part of a `complement-expr` is always one segment, from which each `\`
subtracts another. Unions of independently positive segments exist only at
the block
level. Every expression is therefore in disjunctive normal form: a union of
blocks, each an intersection term with complemented segments as negated
literals. This is the normal-form commitment from
[dag-range-primitives.md](dag-range-primitives.md), imposed by the grammar
rather than left as a convention.

## Validation

The current specification prevents most empty ranges syntactically, with
rules that presuppose interval semantics: constraints sorted by version,
each version unique in the range, comparators alternating between lower and
upper bounds. Under conjunction semantics the sorting and alternation rules
lose their object — the order of constraints in a list carries no meaning —
and they are repealed. What replaces them is split by what a tool can check
without knowing the scheme.

**Scheme-agnostic rules** — decidable from the expression alone; every tool
must enforce them:

- `*` is exclusive of version comparators, enforced by the grammar:
  `*|<5.0.0` is not a valid comparator list — it is just `<5.0.0`. This
  carries over the current rule that `*` stands alone, relaxed in exactly
  two ways: a filter group may accompany it (`*|#stable`), and it may be
  the positive side of a subtraction (`*\>=1.2.3`).
- `*` must not appear as a subtrahend: `A\*` denotes the empty set.
- Within one comparator list, a version string occurs at most once,
  regardless of comparators. Two constraints on the same version are
  contradictory (`>=1.0.0|<1.0.0`), reducible to a single comparator
  (`>=1.0.0|!=1.0.0` is `>1.0.0`), or — when exactly one of them is a cut —
  expressible with the version on the subtrahend side: `$>=2.0.0|<2.0.0`,
  the pre-releases of `2.0.0`, is written `$>=2.0.0\>=2.0.0`. The rule
  therefore costs no expressiveness. It is scoped to a single list: the
  same version may legitimately recur across a `\`, as in that rewrite, or
  in another block.
- Within one filter group, a filter string occurs at most once.

**Scheme-aware rules.** A valid expression must not denote the empty set —
some possible version of the scheme must be able to match it. Beyond the
string-decidable cases above, emptiness is a semantic property that only
scheme-aware tooling can detect: `>=2.0.0|<1.0.0` is empty because the
scheme orders `1.0.0` below `2.0.0`; `=1.0.0|=2.0.0` is empty because the
two strings denote different versions (two strings that normalize to the
same version, like `1.0` and `1.0.0` in some schemes, are instead merely
redundant); a subtraction may swallow its entire positive segment. Notation-level
validation is therefore necessarily incomplete. A tool that can decide
emptiness MUST report an empty range as an error; a tool that cannot
accepts the expression, which then matches nothing.

Empty means empty over the scheme's possible versions, not over the
versions released so far: a prospective range like `>=3.0.0`, written
before `3.0.0` exists, matches nothing today and is valid.

Redundancy is not invalidity: `>=1.3.0|>=1.6.0` is a valid conjunction
that a scheme-aware normalizer reduces to `>=1.6.0`.

## Backward compatibility

**Syntax.** This proposal accepts only canonical forms; it is not a strict
superset of the current syntax. No existing comparator syntax is modified,
and the new characters `$`, `#`, `\`, `,`, and `(`, `)` are not used in
any current VERS comparator, but two current allowances do not carry over:

- A bare version is no longer shorthand for equality: `vers:npm/1.2.3` must
  be written `vers:npm/=1.2.3`. Every constraint is spelled the same way —
  an explicit comparator followed by a version — which keeps the grammar
  free of a comparator-less special case. Migration is the mechanical
  insertion of `=`.
- The lexical rules now require percent-encoding for any version character
  outside the safe set (`A–Z a–z 0–9 . _ - ~ + :`). Versions in existing
  expressions consist almost universally of safe characters; an expression
  whose version strings contain a now-structural character must
  percent-encode it when migrating. Such an expression does not necessarily
  fail to parse — `>=1.0#beta` parses as a constraint with a `beta` filter
  — so migration tooling must scan version strings for structural
  characters rather than rely on parse errors. This is an accepted
  consequence of a migration that preserves compatibility in spirit rather
  than strictly.

**Semantics.** This proposal is not semantically backward-compatible. Current
VERS gives the comparator list interval semantics; this proposal gives it
conjunction semantics. The two readings agree on any list that describes a
single interval — at most one lower and one upper bound, plus any number of
`!=` exclusions — which covers the common case. They disagree on lists that
encode multiple intervals: `vers:semver/>=1.1.0|<1.3.0|>=2.0.0|<3.0.0` means
two disjoint intervals today but is unsatisfiable as a conjunction. An
enumeration of versions (`vers:pypi/0.0.1|0.0.2`) is the degenerate case —
each `=` constraint is its own single-version interval.

The migration is mechanical. Sort the constraints by version (as the current
containment algorithm already does), split the list at each interval
boundary, and wrap each interval in a union block (Extension 1):

```
vers:semver/>=1.1.0|<1.3.0|>=2.0.0|<3.0.0
→ vers:(semver/>=1.1.0|<1.3.0)|(semver/>=2.0.0|<3.0.0)

vers:pypi/0.0.1|0.0.2
→ vers:(pypi/=0.0.1)|(pypi/=0.0.2)
```

The rewrite needs no knowledge of the underlying scheme beyond the version
ordering that the current containment algorithm already requires.