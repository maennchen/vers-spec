# Proposed syntax extensions to VERS

This document proposes a set of extensions to the
[VERS](https://github.com/package-url/vers-spec) version range notation. The
goal is to make VERS capable of expressing every range primitive defined in
[dag-range-primitives.md](dag-range-primitives.md).

The extensions are syntactically additive: every valid VERS expression today
still parses under this proposal. They are built on one deliberate semantic
change — the comparator list becomes a conjunction instead of a list of
interval boundaries — which alters the meaning of some existing expressions.
See [Backward compatibility](#backward-compatibility) for the details and the
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

A single-scheme expression without parentheses is unchanged:

```
vers:semver/>=1.1.0|<2.0.0
```

is equivalent to:

```
vers:(semver/>=1.1.0|<2.0.0)
```

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
defined by the version scheme, not the notation. A scheme that defines a
pre-release ordering (e.g. semver's `-alpha` suffix, debian's `~` component)
must specify the cut in terms of that ordering — concretely, the position
immediately before the lowest pre-release of `v`. A scheme with no
pre-release concept must document that the cut of `v` coincides with `v`
itself. A scheme that does not define infimum semantics at all must reject
`$` as invalid for that scheme.

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

A filter is a scheme-defined predicate applied to a segment. It removes
versions that do not satisfy the predicate, regardless of whether they fall
within the bounds.

The syntax is a `#`-prefixed, comma-separated list of filters appended to the
comparator list:

```
vers:<scheme>/<comparator-list>|#<filter>,<filter>
```

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
`#a,b` is equivalent to `#a|#b`, since the comparator list is a conjunction;
the comma form is the canonical one. For example, a `node` scheme could
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

## Extension 4: Intersection

Two or more ranges may be intersected using the `&` operator. A version
matches the intersection if and only if it matches every operand.

```
vers:<scheme>/<left-comparator-list>&<right-comparator-list>
```

`&` binds tighter than `\` (set-minus) but looser than `|` (the comparator
AND list within a segment). This means each side's comparator list is parsed
in full before the intersection is applied, and intersections are resolved
before any subtraction.

Multiple intersections chain left-associatively:

```
A&B&C  ≡  (A&B)&C
```

The primary use case is combining independently authored prospective ranges.
Each author writes their own constraint; the intersection expresses the
requirement that a version must satisfy all of them simultaneously. A
normalizer can reduce the intersection to a canonical single expression.

**Example — two prospective constraints:**

Package A requires `~> 1.3` (expressed as `>=1.3.0|$<2.0.0`) and package B
requires `~> 1.6` (expressed as `>=1.6.0|$<2.0.0`). The combined requirement:

```
vers:semver/>=1.3.0|$<2.0.0&>=1.6.0|$<2.0.0
```

A scheme aware normalizer reduces this to `vers:semver/>=1.6.0|$<2.0.0`.

**Example — safe and compatible (Example 8):**

Versions of `foo` that are compatible (`>=2.0.0|<3.0.0`) and not in the
vulnerable range (`>=2.1.0|<2.2.0`). Example 8 phrases this as an
intersection with a complement, A ∩ (U \ B); since A ∩ (U \ B) = A \ B, the
notation expresses it directly with set-minus (Extension 5):

```
vers:semver/>=2.0.0|<3.0.0\>=2.1.0|<2.2.0
```

The result is `2.0.x` and `2.2.0`–`2.9.x`. The `&` operator itself is for
intersecting independently authored ranges, as in the example above.

**Cross-scheme intersection:** the grammar permits `&` only within a scheme
block, and that is sufficient. Intersecting two multi-scheme ranges
distributes to per-scheme intersections, because cross-scheme terms are
empty (a version has identity in exactly one scheme):

```
((semver/A)|(git/B)) ∩ ((semver/C)|(git/D))  =  (semver/A&C)|(git/B&D)
```

A tool combining independently authored multi-scheme ranges applies this
distribution when constructing the expression; schemes present in only one
of the inputs are dropped.

## Extension 5: Set-minus (complement)

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

Parentheses may be used for clarity but are not required.

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

## Scheme evaluability

The extensions above are designed so that an expression is **self-contained**:
membership is decidable from the expression and a flat list of version
strings alone. Whether that is achievable is a property of the version
scheme, and the scheme registry must classify each scheme:

- **Structure-ordered schemes** (`semver`, `deb`, `otp`, `calver-ym`, ...):
  the scheme defines a total order on version strings, and where the scheme
  is hierarchical, cuts (`$`) expose its series boundaries. All comparators
  are decidable from strings alone; expressions over these schemes are always
  self-contained. Ecosystems that track parallel branches outside the version
  string (e.g. Debian suites) remain structure-ordered — the branch dimension
  lives in the package identifier, one range per stream (see "Forks and
  branches" in [dag-range-primitives.md](dag-range-primitives.md)).

- **Graph-ordered schemes** (`git` commit hashes and tags): identifiers carry
  no order; ordering and branch membership are reachability questions whose
  answers exist only in the repository. Only `=`, `!=`, and `*` are decidable
  from strings alone (identity comparisons). Ordering comparators (`<`, `<=`,
  `>`, `>=`) and cuts are meaningful only to a tool with access to the graph.

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
| 1 | `<version>`, `>=version`, `!=version`, `$<version`, `#filter` | Individual comparators and modifiers |
| 2 | `\|` | AND (comparator list within a segment) |
| 3 | `&` | Intersection |
| 4 | `\` | Set-minus (complement) |
| 5 | `(scheme/...)\|(scheme/...)` | Union of blocks |

## Grammar (ABNF)

```
expression      = "vers:" ( block-union / scheme-block )
block-union     = "(" scheme-block ")" 1*( "|" "(" scheme-block ")" )
scheme-block    = scheme "/" complement-expr
complement-expr = intersect-expr *( "\" intersect-expr )
intersect-expr  = comparator-list *( "&" comparator-list )
comparator-list = comparator *( "|" comparator )
comparator      = version-comparator / filters
version-comparator = [ "$" ] ( "<" / "<=" / ">" / ">=" ) version
                   / ( "=" / "!=" ) version
                   / "*"
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
`(`, `)`, `&`, `\`, `#`, `,`, `$`, `*`, `%`, and the comparator characters
`<`, `>`, `=`, `!`) — must be percent-encoded. Tokenization therefore never
depends on the version scheme's syntax: together with `$` preceding the
comparator, a parser can split any expression into tokens without knowing
the scheme. The safe set covers the characters real schemes use in practice
(semver's `.-+`, deb's `:~`, alpine's `_`), so percent-encoding is the escape
hatch, not the common case.

The grammar enforces a stratification. Within a scheme block, a
`comparator-list` is semantically a single segment — conjoined bounds
describe one interval; `!=` and filters punch holes in it — and intersecting
segments (`&`) yields a segment again, so the positive part of a
`complement-expr` is always one segment, from which each `\` subtracts
another. Unions of independently positive segments exist only at the block
level. Every expression is therefore in disjunctive normal form: a union of
blocks, each an intersection term with complemented segments as negated
literals. This is the normal-form commitment from
[dag-range-primitives.md](dag-range-primitives.md), imposed by the grammar
rather than left as a convention.

## Backward compatibility

**Syntax.** Every existing VERS expression still parses under this proposal,
with one caveat. No existing comparator syntax is modified, and the new
characters `$`, `#`, `\`, `&`, `,`, and `(`, `)` are not used in any current
VERS comparator. The caveat: the lexical rules now require percent-encoding
for any version character outside the safe set (`A–Z a–z 0–9 . _ - ~ + :`).
Versions in existing expressions consist almost universally of safe
characters; an expression whose version strings contain a now-structural
character must percent-encode it when migrating.

**Semantics.** This proposal is not semantically backward-compatible. Current
VERS gives the comparator list interval semantics; this proposal gives it
conjunction semantics. The two readings agree on any list that describes a
single interval — at most one lower and one upper bound, plus any number of
`!=` exclusions — which covers the common case. They disagree on lists that
encode multiple intervals: `vers:semver/>=1.1.0|<1.3.0|>=2.0.0|<3.0.0` means
two disjoint intervals today but is unsatisfiable as a conjunction.

The migration is mechanical. Sort the constraints by version (as the current
containment algorithm already does), split the list at each interval
boundary, and wrap each interval in a union block (Extension 1):

```
vers:semver/>=1.1.0|<1.3.0|>=2.0.0|<3.0.0
→ vers:(semver/>=1.1.0|<1.3.0)|(semver/>=2.0.0|<3.0.0)
```

The rewrite needs no knowledge of the underlying scheme beyond the version
ordering that the current containment algorithm already requires.