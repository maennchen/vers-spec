# Range specification on a DAG

A range over a linear version sequence can be described with simple interval
notation: a start bound, an end bound, and a predicate over the versions
between them. A range over a DAG requires richer primitives because the
structure being queried is richer.

This document defines those primitives in set-theoretic terms. No syntax is
proposed.

## Primitive 1: The version node

The atomic unit of a range is a **node** — a single version within a single
version scheme.

A node is only meaningful relative to its scheme. The string `1.3.7` in semver
and `1.3.7` in debian are different nodes: they may have different ordering
relationships with their neighbors, different pre-release conventions, and
different identity rules.

Nodes have:

- **Identity**: two nodes are equal if and only if they are the same version
  within the same scheme.
- **Ordering**: within a branch, nodes are ordered. Across branches, nodes may
  be incomparable (as established in the version graph document).
- **Properties**: attributes defined by the scheme, such as stability (stable,
  pre-release, release candidate), build metadata, or epoch.

### Positional nodes

Some nodes are not a fixed string but a *position* in the scheme's ordering
defined by a comparator. For example, "the first stable release >= 1.3.0 in
semver" is a positional node — its concrete identity depends on what has
actually been released. Positional nodes are useful for expressing open-ended
bounds that resolve against a known version set.

### Infima and suprema

Some bounds cannot be expressed as either a concrete version or a simple
comparator without ambiguity at the pre-release boundary. Consider the semver
pre-release ordering:

```
1.3.0 < 1.4.0-alpha.1 < 1.4.0 < 2.0.0-rc.1 < 2.0.0
```

To express "all versions from 1.3.0 up to but not including any pre-release of
2.0.0", neither `< 2.0.0` (which would exclude `2.0.0-rc.1`) nor `<= 1.x.y`
(which requires knowing the highest patch) is sufficient on its own.

This motivates **infima and suprema**: the greatest lower bound and least upper
bound of a set of versions defined by the scheme's structure, without
corresponding to any specific release:

- `inf(2.0.0)`: the greatest lower bound of all versions that sort as 2.0.0 or
  later — the position immediately before the first pre-release of `2.0.0`. A
  bound of `< inf(2.0.0)` includes all `1.x.y-pre` releases but excludes all
  `2.0.0` pre-releases.
- `sup(1.3.x)`: the least upper bound of the entire `1.3` minor series — the
  position immediately after the last possible `1.3.x` patch, before `1.4.0`.
  Expresses "all of the 1.3.x series" without knowing the highest patch ever
  released.

Infima and suprema are positions in the scheme's ordering, not version strings.
What constitutes a "minor series boundary" or a "pre-release floor" is
determined by the scheme's own structure.

## Primitive 2: The segment

A **segment** is a connected, directed path through the DAG within a single
version scheme: from a start node to an end node, staying on one branch.

A segment has the following components:

**Scheme**: the version scheme governing node identity and ordering within this
segment. All nodes in a segment share the same scheme.

**Start node**: the lower bound of the segment, inclusive or exclusive. May be
a concrete version, a positional node, or a goalpost value. An absent start
bound means the segment extends to the beginning of the branch.

**End node**: the upper bound of the segment, inclusive or exclusive, or absent
(unbounded). May be a concrete version, a positional node, or a goalpost value.

**Filter predicate**: a boolean condition on node properties, applied within the
segment after the bounds are resolved. Examples:

- exclude all pre-release versions
- exclude versions matching a specific pattern
- include only versions with a given stability level

Filters are scheme-aware: what counts as "pre-release" is defined by the scheme
itself (semver's `-alpha` suffix, debian's `~` component, etc.).

### Scoped filters

A filter need not apply uniformly across the whole segment. Filters can be
*scoped* to a position-relative condition within the segment. A concrete
example from Elixir's `~>` operator illustrates this:

`~> 1.3` expresses: start at `>= 1.3.0`, end at `< 2.0.0`, exclude all
pre-releases. Stated as a segment:

- Start: `>= 1.3.0`
- End: `< 2.0.0`
- Filter: exclude pre-releases

The filter removes all pre-release nodes within the segment, including
`1.4.0-beta.1` and `2.0.0-rc.1` (which would otherwise fall below `2.0.0`).

An alternate reading, `~> 1.3` with pre-releases allowed within the 1.3.x
line but not at the 2.0.0 boundary:

- Start: `>= 1.3.0`
- End: `< inf(2.0.0)`
- Filter: none

The end bound `< inf(2.0.0)` already excludes `2.0.0-rc.1` while permitting
`1.4.0-beta.1` — no filter is needed. The infimum does the work.

## Primitive 3: The fork

A **fork** is a point in the DAG where one branch diverges into two or more.
A fork is identified by the node at which the divergence occurs.

A range that crosses a fork must **explicitly enumerate** which branches to
follow beyond the fork point. Branches not named are outside the range.
Linear ordering cannot be assumed to carry meaning across a fork boundary.

### Example: a vulnerability spanning two branches

A vulnerability is introduced before a fork at node `1.3.5`, where the
`1.3.x` and `1.4.x` branches diverge. The fix lands at `1.3.7` on the
`1.3.x` branch and at `1.4.3` on the `1.4.x` branch.

```
... → 1.3.5 → 1.3.6 → 1.3.7 (fixed)
         ↘
          1.4.0 → 1.4.1 → 1.4.2 → 1.4.3 (fixed)
```

The affected range must be described as two separate segments:

- Segment A: from the introduction point to `1.3.6` (inclusive), on the
  `1.3.x` branch.
- Segment B: from the fork node `1.3.5` to `1.4.2` (inclusive), on the
  `1.4.x` branch.

`1.4.0` is in segment B even though `1.4.0 > 1.3.7` numerically. The
numerical ordering across branches does not imply fix inheritance. The fork
makes the two branches independent paths, each requiring its own segment.

## Primitive 4: Composition operators

### Union

The **union** of two or more segments (∪, see [set union](https://en.wikipedia.org/wiki/Union_(set_theory)))
contains every version that appears in at least one of them. In version range
terms: a version is in the union if it matches any of the constituent segments.

A union can span multiple version schemes. The result is a heterogeneous set;
membership is tested per-scheme for each candidate version.

### Intersection

The **intersection** of two or more segments (∩, see [set intersection](https://en.wikipedia.org/wiki/Intersection_(set_theory)))
contains only versions that appear in all of them. In version range terms: a
version is in the intersection only if it matches every constituent segment
simultaneously.

A version can only satisfy an intersection if it has identity in every
participating scheme. Versions that exist in only some schemes are eliminated.

**Intersection is computed per scheme independently.** For each scheme that
appears on both sides of an intersection, the intersection over that scheme is
computed and preserved. Schemes that appear on only one side are dropped.

Example 1: `(semver-segment ∪ git-segment) ∩ semver-segment`

The git-only versions in the union have no semver identity, so they cannot
satisfy the right-hand semver segment. They are eliminated. The result is the
semver intersection only.

Example 2: `(semver-segment ∪ git-segment) ∩ (git-segment ∪ semver-segment)`

Both semver and git appear on both sides. The semver intersection is computed
(versions satisfying the semver parts of both sides), and the git intersection
is computed independently (versions satisfying the git parts of both sides).
The final result is the union of those two per-scheme intersections. Versions
with identity in either scheme that satisfy both sides survive.

### Complement

The **complement** of a segment S within a universe U (U \ S, where `\` means
"minus" or "except" — see [set difference](https://en.wikipedia.org/wiki/Complement_(set_theory)))
contains every version in U that is not in S. In version range terms: it is
how exclusions are expressed — "all versions in this branch except the patched
ones" is the branch segment minus the patched sub-segment.

## Composition in DNF and CNF

A complex range is a boolean formula over segment membership. Two normal forms
are useful for different use cases:

**DNF (disjunctive normal form):** a union of intersections — "versions in
segment A, or in segment B, but not in the patched sub-segment C":

> (A ∪ B) \ C

DNF is most natural for *affected version sets*, where the range is the union
of vulnerable segments with patches subtracted out.

**CNF (conjunctive normal form):** an intersection of unions — "must satisfy
constraint A and constraint B, where each constraint may admit multiple
alternatives":

> (A₁ ∪ A₂) ∩ (B₁ ∪ B₂)

CNF is more natural for *dependency requirements*, where multiple independent
constraints must all be satisfied simultaneously.

Any range expressible in one form is expressible in the other by de Morgan's
laws, but one form is typically more compact for a given use case.

Since both forms are equally expressive, a notation based on these primitives
should commit to exactly one of them and disallow mixing. Allowing arbitrary
nesting of unions and intersections would make ranges harder to reason about
and implementations significantly more complex.
