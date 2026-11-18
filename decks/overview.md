# Databricks Medallion & Unity Catalog Blueprint

> Zynx Technologies | 3-slide delivery overview

---

## Slide 1: Context & problem

- Zynx x Banking Client
- Engagement: Architecture & Advisory
- The Banking Client required a repeatable medallion architecture and Unity Catalog security model so platform teams could standardise bronze/silver ingestion patterns, row/column controls, and workspace binding without reinventing governance for each domain.
- Engagement levels: Discovery & Assessment -> Solution Architecture -> Security Pattern Definition -> Enablement Pack

---

## Slide 2: Solution architecture & stack

- Zynx produced an architecture pack covering bronze/silver layer frameworks, metadata-driven validation guidance, Unity Catalog workspace binding, user management, and data security patterns including row-level filters and column masking.
- Stack: Databricks, Unity Catalog, Medallion architecture, Data security patterns
- Architecture documented in docs/03-architecture.md with Mermaid diagrams

---

## Slide 3: Implementation highlights & inspection

- Key evidence: architecture/layers/, databricks/data security/, databricks/unity-catalog/
- Open docs/audit/PROJECT-FACTSHEET.md for the audit index
- Repository standardised to Zynx industrial portfolio structure
- Portfolio tag target: 1.0.0-portfolio
