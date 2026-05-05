# Use cases for version range notation

A version range is always written by someone, at some point in time, for some
purpose. The requirements on a range notation differ significantly depending on
that purpose.

This document surveys the main use cases, organized around two axes:

- **Retrospective vs. prospective:** Does the author have full knowledge of the
  version graph at authoring time, or must they make assumptions about versions
  that do not yet exist?
- **Graph visibility:** Independently of what the author knows, how much of the
  version graph can the *tool evaluating* the range actually see?

## Retrospective use cases

In retrospective use cases, the author has access to the complete version graph
at authoring time: every released version, every branch, and the relationships
between them.

### Vulnerability affected ranges

A security researcher, vendor, or CVE Numbering Authority (CNA) declares which
versions of a package are affected by a known vulnerability.

**Author knows:** all versions released up to the disclosure date; which
branches exist; on which branch the fix was applied and in which version.

**Semantics needed:** a precise, possibly disjoint set of versions across
multiple branches; the ability to exclude specific fixed versions within an
otherwise affected range; no need to predict future releases (the set of
affected versions is closed at disclosure time).

Real-world systems that express vulnerability ranges include
[NVD](https://nvd.nist.gov/), [OSV](https://osv.dev/), and
[GitHub Security Advisories](https://github.com/advisories).

### End-of-life and end-of-support declarations

A project or distributor declares which versions are no longer receiving
updates, security patches, or support.

**Author knows:** the full release history; which branches are closed and will
receive no further releases.

**Semantics needed:** precise set membership over a known, finite graph. The
declaration describes a closed set of versions, not an open-ended future range.

[endoflife.date](https://endoflife.date/) aggregates such declarations across
many projects and ecosystems.

### Audit and compliance

A security or legal team checks whether any installed package version falls
within a prohibited range (e.g. "we must not ship versions with known CVEs")
or verifies that all deployed versions satisfy a requirement (e.g. "all
instances must be running >=2.3.1").

**Author knows:** the full list of installed or deployed versions; the
known-bad or known-required ranges derived from advisories or policy.

**Semantics needed:** membership test over a finite, known set of versions.
The evaluation is a point-in-time check: does version X belong to set S?

## Prospective use cases

In prospective use cases, the author writes the range *before* future versions
exist. They must make assumptions — explicit or implicit — about how the version
scheme will evolve and what future releases will contain.

### Package manager dependency requirements

A package author declares which versions of a dependency their package is
compatible with, as part of a package manifest.

**Author knows:** all versions released up to authoring time. They must assume
that future releases within the declared range will exist and remain compatible.

**Semantics needed:** open-ended bounds that include future releases; reliance
on version scheme conventions to give those bounds meaning. For example, a
range like `>=2.0, <3.0` implicitly trusts that the SemVer convention holds:
that no 2.x release will introduce breaking changes. The author is trusting the
version graph to remain linear and compatible within their declared range — that
no incompatible branch will appear at a version number inside it.

Examples: [npm `package.json`](https://docs.npmjs.com/cli/v10/configuring-npm/package-json#dependencies),
[Python `requirements.txt`](https://pip.pypa.io/en/stable/reference/requirements-file-format/),
[Cargo `Cargo.toml`](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html).

### CI and test matrix definitions

A maintainer declares which versions of a runtime or dependency to test
against, including versions that will be released in the future.

**Author knows:** versions available at authoring time; they expect the range
to expand automatically as new versions are released.

**Semantics needed:** open-ended ranges that grow over time, often expressed as
"all releases in a major series" or "the three most recent minor releases".

### Compatibility declarations

A plugin, extension, or integration declares which versions of a host
application it supports.

**Author knows:** variable — this use case can be either retrospective or
prospective depending on when the declaration is written relative to the host's
releases. If all relevant host versions are already released, the author has
full information. If the author expects future host versions to remain
compatible, the range becomes prospective.

**Semantics needed:** may require both precise past ranges (known-compatible
versions) and open-ended future ranges (anticipated compatibility) within a
single expression.

## Graph visibility: what the consumer can see

Even when the *author* of a range has full retrospective knowledge, the *tool
evaluating* the range may not. Graph visibility is a separate concern from
authoring time, and it affects what a range notation can meaningfully express.

**Full graph access:** A tool with access to the full version history can
determine reachability, branch membership, and ancestry. This context can come
from many sources: a VCS repository, a structured release feed, a package
registry that publishes branch metadata, or even a manually curated dataset.
Whatever the source, the tool can evaluate ranges that encode graph position —
provided the notation supports expressing such positions.

**Scheme-deducible structure:** A tool that understands a version scheme's
conventions may be able to infer branch boundaries from version strings alone,
without VCS access. For example, a tool that knows SemVer semantics can infer
that `2.x` and `3.x` are separate lines. If version numbers reliably reflect
the graph structure — a significant assumption — such a tool can reason about
branch membership without inspecting commits. This breaks down when projects
deviate from convention, as the examples in the version graph document show.

**Flat version list only:** Many tools — vulnerability scanners, package
registries, dependency resolvers — have access only to a flat list of released
version strings, with no graph information. They can evaluate ranges solely as
membership tests over that list, with no awareness of ancestry or branches.
This is the lowest common denominator and the basis on which most current
range notations are designed.

To handle all use cases faithfully, a range notation would need to either:

- Encode enough graph structure that flat-list tools can still evaluate it
  correctly — whether by convention, by explicit branch enumeration, or by
  other means — or
- Acknowledge that flat-list tools will make linearization errors on branched
  histories, and treat that as a known, documented constraint.

## The tension

Prospective ranges must linearize the future. The author has no choice but to
assume something like "all 2.x releases will be backwards-compatible", because
they cannot inspect releases that do not yet exist. That assumption can turn
out to be wrong.

Retrospective ranges could respect the graph — the author has full information
— but most current notations force linearization anyway, because they were
designed for the flat-list evaluation model. The result is that precision is
lost even when the author has the information to be precise.

Even when a range correctly encodes graph structure, a consumer without graph
access cannot evaluate it faithfully.

This three-way gap — between what authors know, what notations can express, and
what tools can evaluate — is the core challenge any version range notation must
address.
