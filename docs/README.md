# TrailBase documentation

## Choose a reading path

- **New to the project:** start with [Architecture](architecture.md), then read the
  relevant [contract topic](IMPLEMENTATION-CONTRACT.md).
- **Implementing a slice:** open its [roadmap milestone](ROADMAP.md), then follow the
  linked contract topics and ADRs.
- **Changing behavior:** update the authoritative contract first and the relevant ADR
  when the decision itself changes.
- **Operating the platform:** begin with [Operations](operations/README.md), then use
  the linked platform contract and a runbook when it exists.
- **Continuing design work:** use [Grill-me checkpoint](GRILL-CHECKPOINT.md); it is not
  a source of accepted requirements.

## Document ownership

| Document family | Purpose |
|---|---|
| [contract/](contract/) | Normative requirements and invariants |
| [roadmap/](roadmap/) | Vertical slices, dependencies and acceptance |
| [overview/](overview/) | System and domain orientation |
| [adr/](adr/) | Decision rationale and alternatives |

When documents conflict, the [Implementation Contract](IMPLEMENTATION-CONTRACT.md)
wins.
