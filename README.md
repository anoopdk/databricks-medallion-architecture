# Databricks Medallion & Unity Catalog Blueprint

**Delivered by:** [Zynx Technologies Ltd](https://www.zynxtechnologies.com/)  
**Portfolio host (pre-fork):** `anoopdk/databricks-medallion-architecture`  
**Domain client (sanitised):** Zynx x Banking Client  
**Engagement type:** Architecture & Advisory  
**Portfolio version:** `1.0.0-portfolio`

> Inspection note: Client identity is domain-aliased for public portfolio use. This repository is structured for authenticity review of delivery artefacts, architecture, and engagement scope.

## Project overview

The Banking Client required a repeatable medallion architecture and Unity Catalog security model so platform teams could standardise bronze/silver ingestion patterns, row/column controls, and workspace binding without reinventing governance for each domain.

### Solution delivered

Zynx produced an architecture pack covering bronze/silver layer frameworks, metadata-driven validation guidance, Unity Catalog workspace binding, user management, and data security patterns including row-level filters and column masking.

## Engagement levels

| # | Level |
| --- | --- |
| 1 | Discovery & Assessment |
| 2 | Solution Architecture |
| 3 | Security Pattern Definition |
| 4 | Enablement Pack |

| Field | Value |
| --- | --- |
| Engagement posture | Delivery portfolio artefact prepared for audit / inspection |
| Confidentiality | Domain alias only; no real client legal name in public docs |
| Primary artefacts | Code/notebooks/docs already in-repo + Zynx standard pack under `docs/` and `decks/` |

## Architecture

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

### Implementation flow

```mermaid
flowchart LR
  A[Architecture docs] --> B[Layer checklists]
  B --> C[Security strategy]
  C --> D[UC binding + user management]
```

## Technology stack

`Databricks`, `Unity Catalog`, `Medallion architecture`, `Data security patterns`

## Repository map

| Path | Purpose |
| --- | --- |
| `README.md` | Project overview + engagement levels |
| `docs/` | Problem, solution, architecture, implementation, runbook, audit factsheet |
| `decks/` | 3-slide delivery overview (Markdown + HTML) |
| `src/` / project subtree | Implementation assets (or placeholders where design-led) |
| `tests/` | Tests or placeholders |
| `scripts/` | Operational / helper scripts |
| `configs/` | Non-secret configuration templates |
| `.env.example` | Environment variable placeholders |

### Key existing delivery paths

- `architecture/layers/`
- `databricks/data security/`
- `databricks/unity-catalog/`

## How to inspect this repository

1. Read this README (overview + engagement).
2. Open `docs/audit/PROJECT-FACTSHEET.md`.
3. Review architecture in `docs/03-architecture.md`.
4. Walk implementation evidence listed in the factsheet.
5. Open `decks/overview.html` for the 3-slide executive summary.

## Related Zynx context

Zynx Technologies Ltd provides IT consulting across cloud, data analytics, and digital delivery. This repository is part of the public delivery portfolio hosted on `anoopdk` prior to organisation-level fork.

## License

See `LICENSE`.
