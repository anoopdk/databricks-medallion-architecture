# Architecture

## Context diagram

```mermaid
flowchart TB
  SRC[Operational & file sources] --> B[Bronze ingestion framework]
  B --> S[Silver validation + enrichment]
  S --> G[Gold / curated consumption]
  UC[Unity Catalog + workspace binding] --- B
  UC --- S
  UC --- G
  SEC[Row filters / column masking / object ACL] --- UC
```

## Implementation flow

```mermaid
flowchart LR
  A[Architecture docs] --> B[Layer checklists]
  B --> C[Security strategy]
  C --> D[UC binding + user management]
```

## Design notes

- Prefer layered medallion / governed platform patterns where applicable.
- Keep configuration and secrets outside Git.
- Preserve auditability: documentation must match repository evidence.
