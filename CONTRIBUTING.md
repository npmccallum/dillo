# Contributing

## Cargo config

The checked-in `.cargo/config.toml` enables artifact dependencies, so a
fresh checkout builds with `cargo build`.

Tests cross-build a Linux guest probe and use the existing linker settings
in the packaging configuration:

```sh
cargo --config .copr/cargo-config.toml test --workspace
```

## Commit messages

Commits follow Conventional Commits and standard git formatting, enforced by
gitlint (see `.gitlint`) in CI.

- Subject: `type(scope): summary` — lowercase, imperative, no trailing period,
  at most 50 characters. Types: `feat`, `fix`, `docs`, `style`, `refactor`,
  `perf`, `test`, `build`, `ci`, `chore`, `revert`, `spec`.
- One blank line, then a body wrapped at 72 characters explaining what changed
  and why (not how — the diff shows how).
- Write each message to stand on its own: describe the change and its rationale
  for a reader with no knowledge of the project's history or how the work was
  produced. Do not reference internal process, prior iterations, or review
  back-and-forth. (gitlint cannot check this — it is a review responsibility.)

## COPR packages

The `npmccallum/pichi` COPR project builds `main` for Fedora on x86_64 and
aarch64. `.copr/Makefile` creates one source RPM; COPR builds the native
packages using the spec in `.copr/`. Binary builds need network access
for Cargo's locked dependencies. The unreleased RPM version is `0.1.0~dev`;
COPR's build ID in `Release` makes successive builds of that version upgrades.

## Architecture-conformance reviews

### REVIEW-001 — Evaluate rules with scoped evidence

Human and automated conformance passes must identify:

- Rule ID and source revision.
- Component, host backend, architecture, and configuration examined.
- Relevant source and construction/service paths.
- Tests or probes actually executed and their results.
- Verdict: `conforms`, `violates`, `unverified`, or `not-applicable`.
- Limitations and unresolved contract questions.

A conformance claim applies only to its stated scope. Source inspection, an isolated API probe, a complete VM boot, and confidential-computing validation are different evidence levels.

### REVIEW-002 — Check boundaries and lifecycle

Review declaration-to-realization paths, reverse authority traces, associated-type use, host-backend dependencies in shared code, process-boundary assumptions, resource confinement, and reset/teardown behavior.

Use focused tests and actual host-backend execution where appropriate. Automated source review and pattern checks can identify candidates; they do not prove total architectural conformance.

### REVIEW-003 — Keep requirements separate from implementation gaps

Existing violations belong in tracked findings, not implicit exceptions to the rules. An open issue does not waive a requirement.

Keep unresolved design choices explicit. Do not let a conformance pass silently invent a new interface, binding, or cross-project contract.

See [Dillo design](DESIGN.md) for the normative goals. Keep design
documents free of implementation inventories, rollout plans, and gap lists.
Record shortcomings in issues with evidence and acceptance criteria; an open
issue does not waive the design.
