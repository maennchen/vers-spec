# Range specification on a DAG

A range over a linear version sequence can be described with simple interval
notation: a start bound, an end bound, and a predicate over the versions
between them. A range over a DAG requires richer primitives because the
structure being queried is richer.

This document defines those primitives in set-theoretic terms. No syntax is
proposed.

## The model

Each version scheme *s* defines a set *V<sub>s</sub>* of possible versions,
together with that scheme's identity and ordering rules. The **universe** *U*
is the disjoint union of the per-scheme sets: every element of *U* is a
(scheme, version) pair, and two versions under different schemes are never
equal and never ordered relative to each other.

A range denotes a subset of *U*. The composition operators defined below
(union, intersection, complement) are the ordinary set operations on subsets
of *U*; all cross-scheme behavior follows from the disjointness of the
per-scheme sets, not from additional rules.

## Primitive 1: The version node

The atomic unit of a range is a **node** — a single version within a single
version scheme.

A node is only meaningful relative to its scheme. The string `1.3.7` in semver
and `1.3.7` in debian are different nodes: they may have different ordering
relationships with their neighbors, different pre-release conventions, and
different identity rules.

Nodes have:

- **Identity**: two nodes are equal if and only if they are the same version
  within the same scheme. A version appears at most once in the graph; if it is
  reachable from multiple branches, those branches share the same node (with
  multiple incoming edges), not distinct copies.
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
2.0.0", neither `< 2.0.0` (which would include `2.0.0-rc.1`) nor `<= 1.x.y`
(which requires knowing the highest patch) is sufficient on its own.

This motivates bounds that are **cuts** in the scheme's ordering (in the sense
of a [Dedekind cut](https://en.wikipedia.org/wiki/Dedekind_cut)): positions
that partition the set of all possible versions of a scheme into those below
the cut and those above it, without themselves corresponding to any release.
In semver, the cut associated with a release `M.m.p` separates the versions
whose `(major, minor, patch)` tuple is below `(M, m, p)` from those whose
tuple is `(M, m, p)` or higher — pre-release identifiers are ignored when
locating the cut, so all pre-releases of `M.m.p` fall above it.

Two namings for these cuts are useful:

- `inf(2.0.0)`: the **infimum** (greatest lower bound) of the set of all
  possible versions whose tuple is `(2, 0, 0)` or higher, including their
  pre-releases — the position immediately below the lowest pre-release of
  `2.0.0`. A bound of `< inf(2.0.0)` includes every `1.x.y` release and
  pre-release but excludes all `2.0.0` pre-releases and everything above
  them.
- `sup(1.3.x)`: the **supremum** (least upper bound) of the set of all
  possible `1.3.x` versions — the position immediately above the highest
  possible `1.3` patch. Expresses "all of the 1.3.x series" without knowing
  the highest patch ever released.

The two are dual descriptions of the same kind of object, and they can
coincide: every possible semver version is either in the `1.3.x` series or at
or above the `1.4.0` family, so `sup(1.3.x) = inf(1.4.0)`. A notation
therefore needs only one of the two; the other is definable from it.

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
a concrete version, a positional node, or a cut (infimum/supremum). An absent
start bound means the segment extends to the beginning of the branch.

**End node**: the upper bound of the segment, inclusive or exclusive, or absent
(unbounded). May be a concrete version, a positional node, or a cut
(infimum/supremum).

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
participating scheme. Because the per-scheme version sets are disjoint, no
special cross-scheme rule is needed: ordinary set intersection over the
universe already computes the result per scheme and eliminates versions whose
scheme appears on only one side.

Example: in `(semver-segment ∪ git-segment) ∩ semver-segment`, the git-only
versions on the left have no semver identity, so no element of the right-hand
side can equal them; they are eliminated, and the result is the intersection
of the two semver segments.

### Complement

The **complement** of a segment S within a universe U (U \ S, where `\` means
"minus" or "except" — see [set difference](https://en.wikipedia.org/wiki/Complement_(set_theory)))
contains every version in U that is not in S. In version range terms: it is
how exclusions are expressed — "all versions in this branch except the patched
ones" is the branch segment minus the patched sub-segment.

## Composition in DNF and CNF

A complex range is a boolean formula over segment membership, in which a
complemented segment acts as a negated literal. Two normal forms are useful
for different use cases:

**DNF (disjunctive normal form):** a union of intersection terms. "Versions
in segment A, or in segment B, but not in the patched sub-segment C" is

> (A ∪ B) \ C  =  (A ∩ C̄) ∪ (B ∩ C̄)

where C̄ is the complement of C within the universe. DNF is most natural for
*affected version sets*, where the range is the union of vulnerable segments
with patches subtracted out.

**CNF (conjunctive normal form):** an intersection of union clauses — "must
satisfy constraint A and constraint B, where each constraint may admit
multiple alternatives":

> (A₁ ∪ A₂) ∩ (B₁ ∪ B₂)

CNF is more natural for *dependency requirements*, where multiple independent
constraints must all be satisfied simultaneously.

The two forms are equally expressive: any formula over union, intersection,
and complement converts into either normal form using De Morgan's laws
together with the distributivity of ∩ over ∪ (and of ∪ over ∩), although the
conversion can grow the formula.

Because of this equal expressiveness, a notation based on these primitives
should commit to exactly one of the two forms and disallow mixing. Allowing
arbitrary nesting of unions and intersections would make ranges harder to
reason about and implementations significantly more complex.
