# Architecture Blueprint: Governed, Metadata-Driven Data Platform on Microsoft Fabric

## Overview

This document presents the architecture of a data platform built on Microsoft Fabric. It is described at the level of strategy and solution design rather than implementation, and it covers data governance, data modelling, reporting and analytics, and AI readiness.

### Context and design drivers

- Data comes from heterogeneous sources (SQL Server, Oracle, SaaS and others), some of them on-premises.
- The data includes PII, and reporting serves both regulatory bodies and internal users.
- Several business areas (such as CRM, Finance and Operational) need their own models, and those models must still reference each other.
- Internal systems need governed access to the platform's data.
- Logic, objects and configuration have to be deployed repeatably, as code.

---

## 1. Data Strategy

### 1.1 Vision

A governed, self-serve data platform on Microsoft Fabric that delivers trusted, reusable data products to the business and is ready for AI.

### 1.2 Strategic objectives

1. Ingest any source through one metadata-driven framework, so onboarding a new source is a configuration change, not a development project.
2. Deliver trusted data products: dimensional models per business area and contract-based data products for internal systems.
3. Govern by design: PII segregation, lineage and access control built into the platform instead of added afterwards.
4. Simplify consumption: semantic models today, a cross-model ontology layer next, which is also the foundation for AI.

### 1.3 Strategic pillars (Data Mesh)

| Pillar | What it means here |
|---|---|
| **Data Products** | Owned, trusted and reusable data assets: dimensional models per business area and contract-based products for internal systems. |
| **Data Fabric** | Standardised, repeatable delivery: metadata-driven frameworks, CI/CD and everything as code (GitFlow, Azure DevOps). |
| **Data Governance** | The backbone: PII segregation, lineage, access control and data contracts. |

### 1.4 Operating model

Federated. A central platform team owns the Raw layer, the frameworks and CI/CD. Business areas own their dimensional models and semantic models, and models are designed to reference each other across areas.

### 1.5 Capability map

| Capability | What it delivers | Layer | Status |
|---|---|---|---|
| Data Factory | Metadata-driven ingestion framework | Raw | In place |
| Data Storage | Lakehouse per source, dimensional Warehouse | Raw, Modelling | In place |
| Data Governance | PII segregation, lineage, access control, contracts | All layers | In place |
| Data Services | Data products, semantic models, ontologies | Integration, Semantic | Partly in place, ontologies planned |
| Analytics | Reporting today, AI later | Semantic / Reporting | Reporting in place, AI on the roadmap |
| Data Foundation | GitFlow, CI/CD, deployment through Fabric APIs | All layers | In place |

### 1.6 Success measures

Measures that show whether the strategy is working:

- Time to onboard a new source.
- Share of data products with a named owner and a contract.
- PII incidents.
- Report adoption.

### 1.7 Evolution

| Horizon | Focus |
|---|---|
| **In place** | Metadata-driven Raw framework, dimensional Modelling layer with PII segregation, semantic models and reporting, CI/CD with GitFlow. |
| **Next** | Wider rollout of data products and contracts in the Integration layer. Ontology layer on top of the semantic models. |
| **Later** | Natural-language access grounded on the ontology layer. AI-assisted authoring of manifests and contracts. |

---

## 2. Logical architecture

```mermaid
flowchart LR
    Src["Sources<br/>SQL Server, Oracle, SaaS"] --> Raw
    subgraph Platform["Microsoft Fabric platform"]
        direction LR
        Raw["Raw layer<br/>Lakehouse per source<br/>Historical and snapshot Delta"]
        Mod["Modelling layer<br/>MLVs and dbt<br/>Warehouse: standard and controlled"]
        Sem["Semantic and reporting layer<br/>Semantic models and reports"]
        Int["Integration layer<br/>Data products and contracts"]
        Raw --> Mod
        Mod --> Sem
    end
    Sem --> Users["Business users and regulators"]
    Int --> Sys["Internal systems"]
    Onto["Ontology layer (planned)"] -.-> Sem
    AI["AI consumption (roadmap)"] -.-> Onto
    Gov["Governance<br/>Lineage, access control, PII segregation, contracts"] -.- Platform
    Found["Foundation<br/>GitFlow, CI/CD, Fabric APIs"] -.- Platform
```

Design principles that apply to every layer:

- **Metadata-driven:** behaviour is defined by configuration (manifests and contracts), not code, so frameworks are source-agnostic and reusable.
- **Governance by design:** lineage, access control and PII masking are part of the architecture.
- **Separation of concerns:** each layer has one purpose, a defined audience and its own access model.
- **Everything as code:** GitFlow and CI/CD (Azure DevOps) for logic, Fabric objects and configuration.
- **AI-ready by design:** AI workloads consume data through the same governed path as any other consumer.

---

## 3. Key design decisions

| Decision | Rationale |
|---|---|
| One manifest per source, stored in the source's own lakehouse in OneLake | Pipelines and Spark jobs become fully parameterisable and agnostic of the source and of the data. A new source is a configuration change. |
| Historical and snapshot Delta tables per source, with Change Data Feed on | Keeps how each record evolved over time and a mirror of the source at each point in time, and lets downstream layers process only the latest changes. |
| Raw restricted to Data Engineering, with no business rules | The business never depends on raw objects. Business rules are defined and applied in later layers. |
| Objects and manifest deployed in the same release (GitFlow, Azure DevOps, Fabric APIs) | Logic, Fabric objects and configuration are versioned and promoted together. |
| A star schema per business area, with cross-area references, SCD Type 2 dimensions, surrogate keys and bridge tables | Preserves history and keeps relationships between dimensions correct, including relationships between entities of the same type. |
| `standard` and `controlled` schemas in every model | Segregates PII: non-PII and masked values are exposed in `standard`, while PII and unmasked values stay in `controlled`. |
| Materialized Lake Views and dbt | MLVs process only the latest records thanks to Change Data Feed and give clear lineage. dbt defines where data is read and written, access control through configuration, and masking through macros. |
| Data contracts and shortcuts for data products | Fabric acts as a data producer. Integration is agile, contract processing follows the same metadata-driven approach as ingestion, and access follows the governance definitions. |
| Ontology layer on top of the semantic models (planned) | Enables queries across several semantic models so people can build reports and dashboards quickly, and gives AI a business-level grounding layer. |

---

## 4. Layer design

### 4.1 Raw layer (Data Factory, Data Storage)

**Purpose.** Reliably land source data and keep full history and point-in-time snapshots, without applying business rules. Only Data Engineering has access.

**Typical patterns**

- Metadata-driven ingestion: one manifest per source defines what is ingested, from where and to where, the load pattern (full, incremental or CDC) and the columns used for ingestion and processing.
- Manifest stored in the file system of the source's own lakehouse in OneLake, so pipelines and Spark jobs are source-agnostic.
- One lakehouse per source with Delta tables of two kinds: historical (how each record evolves) and snapshot (a mirror of the source at a point in time), with Change Data Feed enabled.
- CI/CD creates the Fabric objects through the Fabric APIs and deploys the manifest in the same release.

**Key non-functional requirements**

- Reusability: the same framework serves any source and any data.
- Traceability: full history of record changes.
- Security: access restricted to Data Engineering.
- Repeatability: all objects and configuration deployed as code.

**Success measures.** Time to onboard a new source; share of sources running on the framework.

**Key capabilities.** Batch, incremental and CDC ingestion through On-Premises Data Gateways, Fabric Data Pipelines and Spark; manifest management; historical and snapshot storage; automated deployment.

### 4.2 Integration layer (Data Services)

**Purpose.** Expose governed data products to internal systems through data contracts. Fabric moves from data consumer to data producer.

**Typical patterns**

- Data Mesh data products, each described by a data contract that defines what is exposed and under which conditions.
- Contract-driven generation of consumption artefacts, following the same metadata-driven approach as ingestion.
- Access through shortcuts in dedicated lakehouses, granted according to the governance definitions.

**Key non-functional requirements**

- Interface stability: the contract is the agreement between producer and consumer.
- Security: access follows the governance definitions.
- Agility: onboarding a new consumer is a configuration change.

**Success measures.** Share of data products with a named owner and a contract; time to onboard a new consumer.

**Key capabilities.** Data contracts; contract processing framework; governed access through shortcuts.

### 4.3 Modelling layer (Data Storage, Data Governance)

**Purpose.** Turn raw history into trusted, reusable dimensional models per business area, with PII segregated by design.

**Typical patterns**

- A star schema per business area (for example CRM, Finance, Operational), designed to reference each other.
- SCD Type 2 dimensions to keep history; facts that reference dimensions through surrogate keys; bridge tables to expand relationships between dimensions.
- Two schemas per model. `standard` holds non-PII columns plus masked versions of sensitive attributes. `controlled` holds the PII values and the unmasked originals. As an illustration, of 100 columns about 90 go to `standard` (5 of them masked) and about 10 go to `controlled` (the 5 unmasked originals plus 5 PII columns).
- Materialized Lake Views apply joins and processing logic and, thanks to Change Data Feed on the raw snapshots, process only the latest changes. dbt reads from the lakehouse and writes to the Warehouse. Dimensions are built with dbt snapshots, and facts are queries on top of those snapshots.

**Key non-functional requirements**

- Privacy by design: PII segregation and reusable masking macros.
- Lineage: clear end-to-end data lineage through the Materialized Lake Views.
- Access control: defined once through dbt configuration.
- Efficiency: incremental processing of changed records only.

**Success measures.** PII incidents; reuse of conformed dimensions across business areas.

**Key capabilities.** Dimensional modelling; history tracking (SCD2); PII segregation and masking; lineage.

### 4.4 Semantic model and reporting layer (Data Services, Analytics)

**Purpose.** Deliver consistent metrics and reports for regulatory and internal use, and evolve towards cross-model discovery through ontologies.

**Typical patterns**

- One semantic model per business unit, built on that unit's dimensional model.
- Planned ontology layer: shared business concepts and their relationships defined once and mapped to the tables, columns and measures of each semantic model, enabling queries that span several semantic models so people can build reports and dashboards quickly.

**Key non-functional requirements**

- Consistency: shared definitions through semantic models.
- Governed access: access and PII rules from the Modelling layer continue to apply.
- Self-service: faster report building across domains once the ontology layer exists.

**Success measures.** Report adoption; time to build a cross-domain report (once ontologies are in place).

**Key capabilities.** Semantic models; regulatory and internal reporting; ontology layer (planned).

### 4.5 AI readiness (Analytics)

AI is treated as another consumer of the governed platform, not as a separate stack.

**Current state.** Copilot is available to users as an application. There are no AI agents or AI integrations in the platform yet, so apart from the governance foundations below, everything in this section is roadmap.

**Foundation already in the design**

- PII is segregated into `standard` and `controlled` schemas, so AI workloads can be scoped to `standard` only. Unmasked and PII values are never in their reach.
- Lineage from Materialized Lake Views and dbt makes it possible to trace which data feeds any AI-generated answer.
- Access control is defined once and applies equally to people, reports and AI agents.

**Target state (roadmap)**

- Natural-language access to the semantic models through an AI agent or Copilot that respects the permissions of the user asking. The ontology layer acts as the grounding context, so the agent can resolve a question in business language and answer across several models.
- AI-assisted authoring of manifests and contracts. An LLM drafts the configuration, and a person reviews it through the normal GitFlow pull request process, so it stays reviewed, versioned and auditable.

---

## 5. Cross-cutting concerns

- **CI/CD (Data Foundation):** GitFlow with Azure DevOps pipelines for Fabric objects, logic and configuration (manifests).
- **Governance (Data Governance):** lineage through Materialized Lake Views, access control through dbt configuration and workspace and lakehouse permissions, PII segregation through the standard and controlled schemas, and data contracts for consumption by other systems.
- **Reusability:** one framework per concern, configured per source or contract rather than rebuilt.

---

## Appendix A. Detailed sequence diagrams

### A.1 Raw layer: deployment and runtime

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Data Engineer
    participant ADO as Azure DevOps CI/CD
    participant API as Fabric REST API
    participant LH as Source Lakehouse (OneLake)
    participant Pipe as Fabric Data Pipeline
    participant GW as On-Prem Data Gateway
    participant Src as Source System
    participant Spark as Spark Job

    Note over Dev,LH: Deployment (GitFlow)
    Dev->>ADO: Merge change via GitFlow
    ADO->>API: Create or update Fabric objects
    ADO->>LH: Deploy manifest to lakehouse Files

    Note over Pipe,Spark: Runtime (driven entirely by the manifest)
    Pipe->>LH: Read manifest
    Pipe->>GW: Request extract (full, incremental or CDC)
    GW->>Src: Query source
    Src-->>GW: Records
    GW-->>Pipe: Records
    Pipe->>LH: Write raw files to OneLake
    Spark->>LH: Read manifest
    Spark->>LH: Read raw files
    Spark->>LH: Write historical Delta table (CDF on)
    Spark->>LH: Write snapshot Delta table (CDF on)
```

### A.2 Integration layer: data products and contracts

```mermaid
sequenceDiagram
    autonumber
    actor Owner as Data Product Owner
    participant Contract as Data Contract
    participant Gen as Contract Processing Framework
    participant IntLH as Integration Lakehouse (shortcuts)
    participant Gov as Governance and Access Control
    participant Consumer as Consumer System

    Owner->>Contract: Define data product contract
    Gen->>Contract: Read contract
    Gen->>Gen: Generate consumption artefacts from contract
    Gen->>IntLH: Expose governed data through shortcuts
    Gov->>IntLH: Apply access policy
    Consumer->>IntLH: Read data product
    IntLH-->>Consumer: Governed data
```

### A.3 Modelling layer: dimensional models

```mermaid
sequenceDiagram
    autonumber
    participant Raw as Raw Lakehouse (snapshot Delta, CDF on)
    participant MLV as Materialized Lake Views
    participant dbt as dbt
    participant Macro as dbt Masking Macros
    participant WHs as Warehouse (standard schema)
    participant WHc as Warehouse (controlled schema)

    MLV->>Raw: Read changed records only (CDF)
    MLV->>MLV: Apply joins and processing logic
    dbt->>MLV: Read MLV output
    dbt->>dbt: Build dimensions with dbt snapshots (SCD2)
    dbt->>Macro: Apply masking rules
    Macro-->>dbt: Masked and unmasked column sets
    dbt->>WHs: Write non-PII and masked columns
    dbt->>WHc: Write PII and unmasked columns
    dbt->>dbt: Build facts on top of snapshots (surrogate keys, bridge tables)
    dbt->>WHs: Write facts
```

### A.4 Semantic and reporting layer: current state

```mermaid
sequenceDiagram
    autonumber
    actor User as Business User / Regulator
    participant Report as Report / Dashboard
    participant SM as Semantic Model (per business unit)
    participant WH as Warehouse (standard schema)

    User->>Report: Open report
    Report->>SM: Request data
    SM->>WH: Query dimensional model
    WH-->>SM: Results
    SM-->>Report: Measures and dimensions
    Report-->>User: Visuals
```

### A.5 Semantic and reporting layer: target state with ontologies (planned)

```mermaid
sequenceDiagram
    autonumber
    actor User as Business User
    participant Report as Report / Dashboard
    participant Onto as Ontology Layer (planned)
    participant SM1 as Semantic Model A
    participant SM2 as Semantic Model B
    participant WH as Warehouse

    User->>Report: Ask for cross-domain view
    Report->>Onto: Request business concepts
    Onto->>Onto: Resolve concepts and relationships to model mappings
    Onto->>SM1: Query mapped elements
    Onto->>SM2: Query mapped elements
    SM1->>WH: Query dimensional model
    SM2->>WH: Query dimensional model
    WH-->>SM1: Results
    WH-->>SM2: Results
    SM1-->>Onto: Results
    SM2-->>Onto: Results
    Onto-->>Report: Combined result
    Report-->>User: Visuals
```

### A.6 Governed AI consumption (roadmap)

```mermaid
sequenceDiagram
    autonumber
    actor User as Business User
    participant Agent as AI Agent / Copilot (planned)
    participant Onto as Ontology Layer (planned)
    participant SM as Semantic Models
    participant WHs as Warehouse (standard schema)
    participant WHc as Warehouse (controlled schema)

    User->>Agent: Ask a question in business language
    Agent->>Onto: Resolve business concepts
    Onto-->>Agent: Concept to model mappings
    Agent->>SM: Query within the user's permissions
    SM->>WHs: Read standard schema only
    WHs-->>SM: Results
    SM-->>Agent: Results
    Agent-->>User: Answer
    Note over WHc: Controlled schema is never in scope for AI workloads
```

### A.7 AI-assisted authoring of manifests and contracts (roadmap)

```mermaid
sequenceDiagram
    autonumber
    actor Eng as Data Engineer
    participant LLM as LLM Assistant (planned)
    participant Repo as Git Repository
    actor Rev as Reviewer
    participant ADO as Azure DevOps CI/CD
    participant WS as Fabric Workspace

    Eng->>LLM: Provide source schema or data product definition
    LLM-->>Eng: Draft manifest or contract
    Eng->>Repo: Open pull request (GitFlow)
    Rev->>Repo: Review and approve
    Repo->>ADO: Trigger pipeline
    ADO->>WS: Deploy manifest or contract
```
