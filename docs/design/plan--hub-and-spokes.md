# The Graphtastic Platform: Implementation Plan v2.0 (Hub and Spokes Edition)

This document provides the master plan for implementing the Graphtastic Platform as defined in **The Graphtastic Platform Tome v8.4**. It is divided into two parts:

*   **Part 1: Strategic Rationale & Architectural Decisions:** This section explains the guiding philosophy and critical decisions that have shaped the phased implementation. It serves as the "why" behind the detailed steps that follow.
*   **Part 2: The Phased Execution Plan:** This section provides a master checklist and links to the detailed, sequential sub-plans for engineers to execute.

---

## Table of Contents

- [The Graphtastic Platform: Implementation Plan v2.0 (Hub and Spokes Edition)](#the-graphtastic-platform-implementation-plan-v20-hub-and-spokes-edition)
  - [Table of Contents](#table-of-contents)
  - [Part 1: Strategic Rationale \& Architectural Decisions](#part-1-strategic-rationale--architectural-decisions)
    - [1.0 Introduction: The Guiding Philosophy](#10-introduction-the-guiding-philosophy)
    - [1.1 The "Tools-First" Imperative: Building the Factory Before the Car](#11-the-tools-first-imperative-building-the-factory-before-the-car)
    - [1.2 The "Isolate the Variables" Approach to Federation](#12-the-isolate-the-variables-approach-to-federation)
    - [1.3 A Phased Introduction to Stateful Backends (The Dgraph Track)](#13-a-phased-introduction-to-stateful-backends-the-dgraph-track)
    - [1.4 Architectural Decision: Encapsulating GraphQL Mesh](#14-architectural-decision-encapsulating-graphql-mesh)
    - [1.5 Architectural Decision: Solving the Template "Update Problem" from Day One](#15-architectural-decision-solving-the-template-update-problem-from-day-one)
    - [1.6 Architectural Decision: Naming Conventions and Abstractions](#16-architectural-decision-naming-conventions-and-abstractions)
    - [1.7 Architectural Decision: GraphQL Mesh for Supergraph Composition](#17-architectural-decision-graphql-mesh-for-supergraph-composition)
    - [1.8 Architectural Decision: Dependency Versioning Strategy](#18-architectural-decision-dependency-versioning-strategy)
  - [Part 2: The Phased Execution Plan](#part-2-the-phased-execution-plan)
    - [Master Task Checklist](#master-task-checklist)

---

## Part 1: Strategic Rationale & Architectural Decisions

### 1.0 Introduction: The Guiding Philosophy

The implementation strategy for the Graphtastic Platform is founded on a core philosophy: **systematically reduce risk by isolating and solving one architectural challenge at a time.** We will not attempt a "big bang" integration. Instead, each phase is designed to deliver a stable, verifiable milestone that serves as the foundation for the next. This ensures that when problems arise, they are immediately attributable to the new component being introduced, dramatically simplifying debugging and ensuring predictable progress.

### 1.1 The "Tools-First" Imperative: Building the Factory Before the Car

The first and most critical decision in our plan is to build the foundational tooling *before* any functional components. As detailed in **Phase 0**, we will first create the `tools-docker-compose`, and `tools-subgraph-core` repositories. The `template-*` repo(s) will only be created after we demonstrate end-to-end MVP. We don't want to create templates before this.

- **Rationale:** This approach establishes the complete developer control plane and governance framework from day one. It guarantees that the very first Spoke (`subgraph-blogs`) is built using the same standardized CI pipelines, `Makefile` orchestration from `tools-subgraph-core`, and validation checks as the most complex Spoke that will come later. This front-loading of effort eradicates an entire class of future integration problems and ensures a consistent, high-quality developer experience throughout the project's lifecycle.

### 1.2 The "Isolate the Variables" Approach to Federation

The most complex technical challenge in this architecture is achieving successful federation. Our plan deliberately de-risks this by isolating it from other variables, specifically the introduction of new backend technologies.

- **Rationale:** As laid out in **Phase 1**, we will first achieve a "Minimal Viable Federation" using two technologically identical, stateless subgraphs (`subgraph-blogs` and `subgraph-authors`, both based on GraphQL Yoga). By federating two simple, known-good components, we can be certain that any issues that arise are in the federation layer itself (the `federated-graph-core` services, the `supergraph-cncf` configuration, or the inter-service networking) and not in the subgraphs. This validates the core architectural pattern in the most controlled environment possible.

### 1.3 A Phased Introduction to Stateful Backends (The Dgraph Track)

Only after proving the federation mechanism do we introduce the complexity of a stateful backend. The plan defines a clear, multi-stage "Dgraph track" that progresses from simple to complex.

- **Rationale:** This progression isolates new challenges at each step.
  - **Phase 2 (`subgraph-dgraph-static`):** This phase focuses *only* on the challenge of integrating Dgraph as a Spoke. By using a static dataset, we are not yet concerned with data pipelines.
  - **Phase 3 (`subgraph-dgraph-gharchive`):** This phase builds on the now-stable Dgraph pattern and introduces the next challenge: dynamic data ingestion from an external source.
  - **Phase 4 (`subgraph-dgraph-software-supply-chain`):** The final and most complex phase introduces a sophisticated internal ETL pipeline, leveraging our analysis from `design--guac-to-dgraph.md`. This is tackled last because it relies on all previously validated patterns.

Furthermore, we will leverage Dgraph's native federation compliance. This simplifies the implementation of Dgraph-backed Spokes, as they can expose a federated GraphQL endpoint directly, perfectly aligning with the "Spoke as a black box" principle from the Tome.

### 1.4 Architectural Decision: Encapsulating GraphQL Mesh

For the `subgraph-dgraph-software-supply-chain`, which requires transforming the GUAC API, we explicitly decided that GraphQL Mesh will be run *within* the Spoke.

- **Rationale:** This decision directly upholds the architectural precepts of the Tome v8.4.
    1. **It preserves the "Spoke as a Black Box" model.** The Hub does not need to know about the Spoke's internal transformation needs.
    2. **It ensures the Spoke remains "Standalone."** The developers of this Spoke can run and test their entire data pipeline in isolation without needing the `federated-graph-core`.
    3. **It avoids tight coupling and configuration sprawl** that a centralized Mesh service would create, and validates the "sidecar" pattern.

### 1.5 Architectural Decision: Solving the Template "Update Problem" 

Usage of "GitHub Repository Templates" upon review introduces a critical "update problem"—the lack of an ongoing link between a template and its generated children. A solution is proposed, but it's problematic. For now we're not creating templates.

### 1.6 Architectural Decision: Naming Conventions and Abstractions

We clarified and confirmed our naming conventions to maintain architectural clarity. **(TODO: update the tome)**

- **Rationale:**
  - We will retain the name `federated-graph-core` because it precisely describes its function as the engine that enables federation.
  - We will defer the creation of a `tools-supergraph-core` repository. The Hubs are intended to be simple assemblers, and their logic is either provided by `tools-docker-compose` or is use-case-specific. Introducing another tools repository at this stage would be an unnecessary abstraction.

### 1.7 Architectural Decision: GraphQL Mesh for Supergraph Composition

The Tome v8.4 describes the "Render, Commit, Run" workflow, where a `supergraph.graphql` artifact is generated and committed to source control. The plan executes this using the `npx @graphql-mesh/compose-cli` tool. More info: <https://the-guild.dev/graphql/mesh/v1/consume-in-other-gateways>

- **Rationale:** This is a deliberate architectural choice. While other tools (including services within the Hive platform) can perform composition, using the standalone GraphQL Mesh CLI offers key advantages for our CI/CD and developer workflow:
    1. **It is a lightweight, stateless build tool.** It requires no running services, databases, or external state, making it perfect for fast, ephemeral CI jobs.
    2. **It decouples composition from the runtime.** Our build process does not depend on our running `federated-graph-core` stack, which aligns perfectly with the goal of treating the supergraph artifact as a declarative build product.
    3. **It provides a consistent toolchain.** Using Mesh for both source adaptation (as in Phase 4) and supergraph composition simplifies the project's tooling dependencies.

### 1.8 Architectural Decision: Dependency Versioning Strategy

Throughout this implementation plan, the `version` field in the `graphtastic.deps.yml` manifest is set to `main`. This is an explicit choice to optimize for the developer "inner loop" during the initial build-out of the platform.

- **Rationale:**
  - **For Development:** Using `main` allows developers working on the Hub to immediately pull in the latest changes from Spoke repositories without waiting for a formal release and version tag. This accelerates iterative development and cross-component work.
  - **For Production:** As mandated by the Tome (Sec 3.4, Precept 5), any Hub intended for a staging or production deployment **must** have its dependencies pinned to specific, immutable Git tags (e.g., `version: v1.3.0`). This plan focuses on the initial, local-first development workflow, and the transition to tagged versions is a critical step for production hardening.

---

## Part 2: The Phased Execution Plan

### Master Task Checklist

This checklist provides a high-level overview of the implementation phases. Each item links to a detailed, actionable sub-plan.

- [ ] **Phase 1: Foundation and Stateless Spoke**
  - [ ] [plan--hub-and-spokes-01-foundation-and-stateless-spoke.md](./plan--hub-and-spokes-01-foundation-and-stateless-spoke.md)
- [ ] **Phase 2: Stateful Spoke with Dgraph**
  - [ ] [plan--hub-and-spokes-02-stateful-spoke-with-dgraph.md](./plan--hub-and-spokes-02-stateful-spoke-with-dgraph.md)
- [ ] **Phase 3: Minimal Viable Supergraph**
  - [ ] [plan--hub-and-spokes-03-minimal-viable-supergraph.md](./plan--hub-and-spokes-03-minimal-viable-supergraph.md)
- [ ] **Phase 4: Advanced Patterns and Platform Hardening**
  - [ ] [plan--hub-and-spokes-04-advanced-patterns-and-platform-hardening.md](./plan--hub-and-spokes-04-advanced-patterns-and-platform-hardening.md)
