# Proposed syntax extensions to VERS

This document proposes a set of extensions to the
[VERS](https://github.com/package-url/vers-spec) version range notation. The
goal is to make VERS capable of expressing every range primitive defined in
[dag-range-primitives.md](dag-range-primitives.md), while remaining
backward-compatible with existing VERS expressions.

The extensions are additive. Every valid VERS expression today is valid under
this proposal without modification.

This is a working draft for discussion — not a finalized proposal.

## Background

Current VERS expresses a range as:

```
vers:<scheme>/<comparator-list>
```

where the comparator list is a `|`-separated sequence of constraints that are
ANDed together. A version matches if and only if it satisfies every constraint
in the list.

This covers simple linear ranges well. It cannot express:

- Ranges that span multiple version schemes
- Ranges that split across parallel branches (forks)
- Upper bounds that exclude pre-releases without a filter
- Stable-only filters
- Sub-range exclusion (complement)
- Intersection of independently authored constraints

Each of these gaps is addressed below.

## Extension 1: Multi-scheme union

A range may span multiple version schemes by wrapping each scheme's constraints
in parentheses and joining them with `|`:

```
vers:(<scheme>/<comparator-list>)|(<scheme>/<comparator-list>)
```

A version matches the expression if it matches any one of the parenthesized
scheme blocks. Each block is evaluated independently within its own scheme.

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

**Example — Debian epoch bump:**

A vulnerability spans two epoch series. The first epoch has no fix; the second
is fixed at `2:1.2.0-1`.

```
vers:(deb/>=1:0.9.0-1)|(deb/>=2:0.0.1-1|<2:1.2.0-1)
```

Note: the Debian epoch bump can also be expressed in a single scheme block
because the `deb` scheme defines epoch ordering within one comparator space.
The two-block form makes the structural split explicit; either is valid.

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
at the pre-release boundary. The infimum of a version `v`, written `^v`, is
the position immediately before the first pre-release of `v` in the scheme's
ordering. A bound of `<^v` excludes all pre-releases of `v` while including
all versions that sort below them.

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

Versions of `foo` that are compatible (`>=2.0.0|<3.0.0`) and not vulnerable
(`>=2.1.0|<2.2.0` is the vulnerable range):

```
vers:semver/>=2.0.0|<3.0.0&*\>=2.1.0|<2.2.0
```

The right side is the universe minus the vulnerable range; intersecting with
the compatibility range gives `2.0.x` and `2.2.0`–`2.9.x`.

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

**Example — implicit universe with exceptions (Example 17):**

```
vers:semver/*\>=1.2.3\>=2.0.1
```

All versions, minus everything from `1.2.3` onward, minus everything from
`2.0.1` onward. The result is `<1.2.3` and `>=2.0.0,<2.0.1`.

## Operator precedence summary

From highest to lowest:

| Level | Operator | Meaning |
|-------|----------|---------|
| 1 | `<version>`, `>=version`, `!=version`, `^version`, `#filter` | Individual comparators and modifiers |
| 2 | `\|` | AND (comparator list within a segment) |
| 3 | `&` | Intersection |
| 4 | `\` | Set-minus (complement) |
| 5 | `(scheme/...)\|(scheme/...)` | Multi-scheme union |

## Grammar (informal)

```
expression      = multi-union | single-segment
multi-union     = "(" scheme-block ")" ("|" "(" scheme-block ")")+
single-segment  = "vers:" scheme "/" complement-expr
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

For `multi-union`, the `vers:` prefix appears once before the opening
parenthesis:

```
vers:(<scheme-block>)|(<scheme-block>)
```

## Backward compatibility

Every existing VERS expression is valid under this proposal. No existing
comparator syntax is modified. The new characters `@`, `{`, `}`, `^`, `#`,
`\`, `&`, and `(`, `)` are not used in any current VERS comparator and do not
conflict with any existing version scheme string.