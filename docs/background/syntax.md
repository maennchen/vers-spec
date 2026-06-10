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
implied by the interval pairing. The motivation: once cuts, filters, fork
blocks, intersection, and subtraction enter the notation, the implicit
pairing rule no longer has a well-defined meaning, while a conjunction gives
every constraint an independent one. This is a breaking semantic change; see
[Backward compatibility](#backward-compatibility).

Even with its interval semantics, current VERS cannot express:

- Ranges that span multiple version schemes
- Ranges that split across parallel branches (forks)
- Upper bounds that exclude pre-releases of the bound (infima)
- Stable-only filters
- Intersection of independently authored constraints
- Sub-range exclusion as a composable operator (today the author must
  manually rewrite the surrounding range into the intervals around the gap)

Each of these gaps is addressed below.

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

## Extension 2: Fork

A fork is a point in the version graph where one line of development diverges
into two or more parallel branches. A range that crosses a fork must enumerate
each branch explicitly.

The syntax for a fork block is:

```
@<version>{<comparator-list>}
```

`@<version>` identifies the fork point — the node at which the branch
diverges. The comparator list inside `{}` applies only to that branch.
The fork point version itself is governed by the constraints outside the block.

Multiple fork blocks may appear in a single comparator list, one per branch.
The main line and each fork are evaluated independently; a version matches if
it satisfies the constraints on its branch.

**Example — CVE spanning parallel maintenance branches:**

A vulnerability is introduced at `1.3.0`. The `1.3.x` branch received a fix
at `1.3.5`; the `1.4.x` branch was abandoned with no fix.

```
vers:semver/>=1.3.0|@1.3.0{<1.3.5}|@1.4.0{}
```

- `>=1.3.0` covers the introduction point on the main line.
- `@1.3.0{<1.3.5}` applies `<1.3.5` on the branch that diverged at `1.3.0`.
- `@1.4.0{}` declares that the `1.4.x` branch is in scope with no upper bound
  — every version on it is affected.

An empty `{}` means the branch is affected without bound. A branch not
mentioned is out of scope for this range entirely.

**Example — Erlang/OTP CVE (scheme switch + multiple forks):**

```
vers:(otp-r-series/>=R13B)|(otp/>=17.0|<26.0|@26.0{>=26.0|<26.2.5}|@27.0{>=27.0|<27.1.1}|@28.0{>=28.0|<28.0.1})
```

The R-series and numeric schemes are expressed as separate blocks (Extension 1).
Within the numeric block, three maintenance branches are enumerated with fork
syntax.

## Extension 3: Infimum

Some upper bounds cannot be expressed as a concrete version without ambiguity
at the pre-release boundary. The infimum of a version `v`, written `^v`, is a
cut in the scheme's ordering (see
[dag-range-primitives.md](dag-range-primitives.md)): the position immediately
below the lowest pre-release of `v`. A bound of `<^v` excludes all
pre-releases of `v` while including all versions that sort below them.

```
<^<version>
```

The `^` prefix is a notation-level concept; its semantics are defined by the
version scheme, not the notation. A scheme that defines a pre-release ordering
(e.g. semver's `-alpha` suffix, debian's `~` component) must specify what
`^v` means in terms of that ordering — concretely, the position immediately
before the lowest pre-release of `v`. A scheme with no pre-release concept
must document that `^v` is identical to `v`. A scheme that does not define
infimum semantics at all must reject `^` as invalid for that scheme.

**Example — Elixir `~> 1.3`:**

`~> 1.3` means `>= 1.3.0` and less than any pre-release of `2.0.0`:

```
vers:semver/>=1.3.0|<^2.0.0
```

`1.4.0-rc.1` is included (within the segment interior); `2.0.0-beta.1` is
excluded (at or beyond `^2.0.0`).

**Example — Elixir `~> 1.3-beta`:**

The lower bound is a pre-release, which admits `1.3.0-beta` itself:

```
vers:semver/>=1.3.0-beta|<^2.0.0
```

## Extension 4: Filter

A filter is a stability predicate applied to a segment. It removes versions
that do not meet the specified stability level, regardless of whether they fall
within the bounds.

The syntax is a `#`-prefixed keyword appended to the comparator list:

```
vers:<scheme>/<comparator-list>|#<filter>
```

Filter names are defined by the version scheme, not the notation. The notation
specifies that `#<name>` applies a named predicate to the segment; the scheme
is responsible for defining what that predicate means for its version strings
and which filter names it supports.

For example, the `semver` scheme defines `#stable` to mean: exclude all
versions with a `-` pre-release identifier.

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

## Extension 5: Intersection

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

Package A requires `~> 1.3` (expressed as `>=1.3.0|<^2.0.0`) and package B
requires `~> 1.6` (expressed as `>=1.6.0|<^2.0.0`). The combined requirement:

```
vers:semver/>=1.3.0|<^2.0.0&>=1.6.0|<^2.0.0
```

A scheme aware normalizer reduces this to `vers:semver/>=1.6.0|<^2.0.0`.

**Example — safe and compatible (Example 8):**

Versions of `foo` that are compatible (`>=2.0.0|<3.0.0`) and not in the
vulnerable range (`>=2.1.0|<2.2.0`). Example 8 phrases this as an
intersection with a complement, A ∩ (U \ B); since A ∩ (U \ B) = A \ B, the
notation expresses it directly with set-minus (Extension 6):

```
vers:semver/>=2.0.0|<3.0.0\>=2.1.0|<2.2.0
```

The result is `2.0.x` and `2.2.0`–`2.9.x`. The `&` operator itself is for
intersecting independently authored ranges, as in the example above.

**Cross-scheme intersection:** intersection is computed per-scheme
independently. A version with no identity in a scheme cannot satisfy a
constraint in that scheme, so it is eliminated from the result. Schemes
present on only one side of `&` are dropped from the output.

## Extension 6: Set-minus (complement)

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

**Example — subtracting from the implicit universe:**

```
vers:semver/*\=1.2.3\=2.0.0
```

All versions except `1.2.3` and `2.0.0`. Equivalent to the existing
`vers:semver/!=1.2.3|!=2.0.0`; the set-minus form additionally scales to
excluding whole sub-ranges, which `!=` cannot express.

The full form of Example 17 — exceptions that are open-ended *per branch* —
requires combining `\` with fork blocks (Extension 2) on the right-hand side,
so that each subtracted segment is scoped to its branch.

## Operator precedence summary

From highest to lowest:

| Level | Operator | Meaning |
|-------|----------|---------|
| 1 | `<version>`, `>=version`, `!=version`, `^version`, `#filter` | Individual comparators and modifiers |
| 2 | `\|` | AND (comparator list within a segment) |
| 3 | `&` | Intersection |
| 4 | `\` | Set-minus (complement) |
| 5 | `(scheme/...)\|(scheme/...)` | Union of blocks |

## Grammar (informal)

```
expression      = "vers:" ( block-union | scheme-block )
block-union     = "(" scheme-block ")" ("|" "(" scheme-block ")")+
scheme-block    = scheme "/" complement-expr
complement-expr = intersect-expr ("\" intersect-expr)*
intersect-expr  = comparator-list ("&" comparator-list)*
comparator-list = comparator ("|" comparator)*
comparator      = version-comparator | fork-block | filter
version-comparator = ("<" | "<=" | ">" | ">=" | "=" | "!=") "^"? version
                   | "*"
fork-block      = "@" version "{" comparator-list? "}"
filter          = "#" filter-name
filter-name     = "stable"
```

## Backward compatibility

**Syntax.** Every existing VERS expression still parses under this proposal.
No existing comparator syntax is modified, and the new characters `@`, `{`,
`}`, `^`, `#`, `\`, `&`, and `(`, `)` are not used in any current VERS
comparator and do not conflict with any existing version scheme string.

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