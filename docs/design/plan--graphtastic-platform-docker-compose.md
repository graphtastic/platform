# The Graphtastic Platform: Implementation Plan v1.0

This document provides the complete, actionable plan for implementing the
Graphtastic Platform as defined in **The Graphtastic Platform Tome v8.3**. It is
divided into two parts:

- **Part 1: Strategic Rationale & Architectural Decisions:** This section
    explains the guiding philosophy and critical decisions that have shaped the
    phased implementation. It serves as the "why" behind the detailed steps that
    follow.
- **Part 2: The Detailed Execution Plan:** This section provides a verbose,
    step-by-step guide for engineers to execute the plan. It includes all
    necessary commands, code, and configuration to ensure a consistent and
    successful implementation.

---

## Table of Contents

- [The Graphtastic Platform: Implementation Plan v1.0](#the-graphtastic-platform-implementation-plan-v10)
  - [Table of Contents](#table-of-contents)
  - [Task Checklist](#task-checklist)
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
  - [Part 2: The Detailed Execution Plan](#part-2-the-detailed-execution-plan)
    - [Phase 0: The Developer Control Plane](#phase-0-the-developer-control-plane)
      - [Task 0.1: Create `tools-docker-compose` Repository](#task-01-create-tools-docker-compose-repository)
      - [Task 0.2: Create `tools-subgraph-core` Repository](#task-02-create-tools-subgraph-core-repository)
      - [Task 0.3: Create `template-spoke-base` Repository](#task-03-create-template-spoke-base-repository)
      - [Task 0.4: Create `template-supergraph` Repository](#task-04-create-template-supergraph-repository)
      - [Task 0.5: Create `template-spoke-dgraph` Repository](#task-05-create-template-spoke-dgraph-repository)
      - [Task 0.6: Create `template-spoke-mesh` Repository](#task-06-create-template-spoke-mesh-repository)
    - [Phase 1: Minimal Viable Federation with Stateless Spokes](#phase-1-minimal-viable-federation-with-stateless-spokes)
      - [Task 1.1: Create `subgraph-blogs`](#task-11-create-subgraph-blogs)
      - [Task 1.2: Create `subgraph-authors`](#task-12-create-subgraph-authors)
      - [Task 1.3: Create `federated-graph-core`](#task-13-create-federated-graph-core)
      - [Task 1.4: Create and Configure `supergraph-cncf`](#task-14-create-and-configure-supergraph-cncf)
    - [Phase 2: Simple Stateful Backend (Dgraph "Hello World")](#phase-2-simple-stateful-backend-dgraph-hello-world)
      - [Task 2.1: Create `subgraph-dgraph-static` Spoke](#task-21-create-subgraph-dgraph-static-spoke)
      - [Task 2.2: Integrate `subgraph-dgraph-static` into `supergraph-cncf`](#task-22-integrate-subgraph-dgraph-static-into-supergraph-cncf)
    - [Phase 3: Complex Stateful Backend (Dynamic Data Ingestion)](#phase-3-complex-stateful-backend-dynamic-data-ingestion)
      - [Task 3.1: Create `subgraph-dgraph-gharchive` Spoke](#task-31-create-subgraph-dgraph-gharchive-spoke)
      - [Task 3.2: Integrate `subgraph-dgraph-gharchive` into `supergraph-cncf`](#task-32-integrate-subgraph-dgraph-gharchive-into-supergraph-cncf)
    - [Phase 4: Advanced Composite Spoke (ETL Pipeline)](#phase-4-advanced-composite-spoke-etl-pipeline)
      - [Task 4.1: Create `subgraph-dgraph-software-supply-chain` Spoke](#task-41-create-subgraph-dgraph-software-supply-chain-spoke)
      - [Task 4.2: Integrate into `supergraph-cncf`](#task-42-integrate-into-supergraph-cncf)
    - [Phase 5: Production Hardening and Polish](#phase-5-production-hardening-and-polish)
      - [Task 5.1: Supergraph CI/CD Hardening](#task-51-supergraph-cicd-hardening)
      - [Task 5.2: Secret Management Implementation](#task-52-secret-management-implementation)
      - [Task 5.3: Configurable Persistence Implementation](#task-53-configurable-persistence-implementation)

## Task Checklist

- [ ] [Task 0.1: Create `tools-docker-compose` Repository](#task-01-create-tools-docker-compose-repository)
- [ ] [Task 0.2: Create `tools-subgraph-core` Repository](#task-02-create-tools-subgraph-core-repository)
- [ ] [Task 0.3: Create `template-spoke-base` Repository](#task-03-create-template-spoke-base-repository)
- [ ] [Task 0.4: Create `template-supergraph` Repository](#task-04-create-template-supergraph-repository)
- [ ] [Task 0.5: Create `template-spoke-dgraph` Repository](#task-05-create-template-spoke-dgraph-repository)
- [ ] [Task 0.6: Create `template-spoke-mesh` Repository](#task-06-create-template-spoke-mesh-repository)
- [ ] [Task 1.1: Create `subgraph-blogs`](#task-11-create-subgraph-blogs)
- [ ] [Task 1.2: Create `subgraph-authors`](#task-12-create-subgraph-authors)
- [ ] [Task 1.3: Create `federated-graph-core`](#task-13-create-federated-graph-core)
- [ ] [Task 1.4: Create and Configure `supergraph-cncf`](#task-14-create-and-configure-supergraph-cncf)
- [ ] [Task 2.1: Create `subgraph-dgraph-static` Spoke](#task-21-create-subgraph-dgraph-static-spoke)
- [ ] [Task 2.2: Integrate `subgraph-dgraph-static` into `supergraph-cncf`](#task-22-integrate-subgraph-dgraph-static-into-supergraph-cncf)
- [ ] [Task 3.1: Create `subgraph-dgraph-gharchive` Spoke](#task-31-create-subgraph-dgraph-gharchive-spoke)
- [ ] [Task 3.2: Integrate `subgraph-dgraph-gharchive` into `supergraph-cncf`](#task-32-integrate-subgraph-dgraph-gharchive-into-supergraph-cncf)
- [ ] [Task 4.1: Create `subgraph-dgraph-software-supply-chain` Spoke](#task-41-create-subgraph-dgraph-software-supply-chain-spoke)
- [ ] [Task 4.2: Integrate into `supergraph-cncf`](#task-42-integrate-into-supergraph-cncf)
- [ ] [Task 5.1: Supergraph CI/CD Hardening](#task-51-supergraph-cicd-hardening)
- [ ] [Task 5.2: Secret Management Implementation](#task-52-secret-management-implementation)
- [ ] [Task 5.3: Configurable Persistence Implementation](#task-53-configurable-persistence-implementation)

---

## Part 1: Strategic Rationale & Architectural Decisions

[ ] Part 1: Strategic Rationale & Architectural Decisions

### 1.0 Introduction: The Guiding Philosophy

[ ] 1.0 Introduction: The Guiding Philosophy

The implementation strategy for the Graphtastic Platform is founded on a core
philosophy: **systematically reduce risk by isolating and solving one
architectural challenge at a time.** We will not attempt a "big bang"
integration. Instead, each phase is designed to deliver a stable, verifiable
milestone that serves as the foundation for the next. This ensures that when
problems arise, they are immediately attributable to the new component being
introduced, dramatically simplifying debugging and ensuring predictable
progress.

### 1.1 The "Tools-First" Imperative: Building the Factory Before the Car

[ ] 1.1 The "Tools-First" Imperative: Building the Factory Before the Car

The first and most critical decision in our plan is to build the foundational
tooling *before* any functional components. As detailed in **Phase 0**, we will
first create the `tools-docker-compose`, `tools-subgraph-core`, and `template-*`
repositories.

- **Rationale:** This approach establishes the complete developer control plane
    and governance framework from day one. It guarantees that the very first
    Spoke (`subgraph-blogs`) is built using the same standardized CI pipelines,
    `Makefile` orchestration from `tools-subgraph-core`, and validation checks
    as the most complex Spoke that will come later. This front-loading of effort
    eradicates an entire class of future integration problems and ensures a
    consistent, high-quality developer experience throughout the project's
    lifecycle.

### 1.2 The "Isolate the Variables" Approach to Federation

[ ] 1.2 The "Isolate the Variables" Approach to Federation

The most complex technical challenge in this architecture is achieving
successful federation. Our plan deliberately de-risks this by isolating it from
other variables, specifically the introduction of new backend technologies.

- **Rationale:** As laid out in **Phase 1**, we will first achieve a "Minimal
    Viable Federation" using two technologically identical, stateless subgraphs
    (`subgraph-blogs` and `subgraph-authors`, both based on GraphQL Yoga). By
    federating two simple, known-good components, we can be certain that any
    issues that arise are in the federation layer itself (the
    `federated-graph-core` services, the `supergraph-cncf` configuration, or the
    inter-service networking) and not in the subgraphs. This validates the core
    architectural pattern in the most controlled environment possible.

### 1.3 A Phased Introduction to Stateful Backends (The Dgraph Track)

[ ] 1.3 A Phased Introduction to Stateful Backends (The Dgraph Track)

Only after proving the federation mechanism do we introduce the complexity of a
stateful backend. The plan defines a clear, multi-stage "Dgraph track" that
progresses from simple to complex.

- **Rationale:** This progression isolates new challenges at each step.
  - **Phase 2 (`subgraph-dgraph-static`):** This phase focuses *only* on the
        challenge of integrating Dgraph as a Spoke. By using a static dataset,
        we are not yet concerned with data pipelines.
  - **Phase 3 (`subgraph-dgraph-gharchive`):** This phase builds on the
        now-stable Dgraph pattern and introduces the next challenge: dynamic
        data ingestion from an external source.
  - **Phase 4 (`subgraph-dgraph-software-supply-chain`):** The final and most
        complex phase introduces a sophisticated internal ETL pipeline,
        leveraging our analysis from `design--guac-to-dgraph.md`. This is
        tackled last because it relies on all previously validated patterns.

Furthermore, we will leverage Dgraph's native federation compliance. This
simplifies the implementation of Dgraph-backed Spokes, as they can expose a
federated GraphQL endpoint directly, perfectly aligning with the "Spoke as a
black box" principle from the Tome.

### 1.4 Architectural Decision: Encapsulating GraphQL Mesh

[ ] 1.4 Architectural Decision: Encapsulating GraphQL Mesh

For the `subgraph-dgraph-software-supply-chain`, which requires transforming the
GUAC API, we explicitly decided that GraphQL Mesh will be run *within* the Spoke.

- **Rationale:** This decision directly upholds the architectural precepts of
    the Tome v8.3.
    1. **It preserves the "Spoke as a Black Box" model.** The Hub does not need
        to know about the Spoke's internal transformation needs.
    2. **It ensures the Spoke remains "Standalone."** The developers of this
        Spoke can run and test their entire data pipeline in isolation without
        needing the `federated-graph-core`.
    3. **It avoids tight coupling and configuration sprawl** that a centralized
        Mesh service would create.

### 1.5 Architectural Decision: Solving the Template "Update Problem" from Day One

[ ] 1.5 Architectural Decision: Solving the Template "Update Problem" from Day
One

Our review of the "GitHub Repository Templates: A Primer" document highlighted
the critical "update problem"—the lack of an ongoing link between a template and
its generated children.

- **Rationale:** Relying on manual, error-prone Git commands is not a scalable
    solution. Therefore, our plan mandates solving this from the start. As part
    of **Phase 0**, the `template-*` repositories **must** include a
    pre-configured, automated GitHub Actions workflow (e.g., using
    `actions-template-sync`). This "self-updating" pattern ensures every
    component in our ecosystem is born with the capability to receive future
    improvements, transforming lifecycle management from a manual burden into a
    governed, automated process.

### 1.6 Architectural Decision: Naming Conventions and Abstractions

[ ] 1.6 Architectural Decision: Naming Conventions and Abstractions

During our discussion, we clarified and confirmed our naming conventions to
maintain architectural clarity.

- **Rationale:**
  - We will retain the name `federated-graph-core` because it precisely
        describes its function as the engine that enables federation.
  - We will defer the creation of a `tools-supergraph-core` repository. The
        Hubs are intended to be simple assemblers, and their logic is either
        provided by `tools-docker-compose` or is use-case-specific. Introducing
        another tools repository at this stage would be an unnecessary
        abstraction.

### 1.7 Architectural Decision: GraphQL Mesh for Supergraph Composition

[ ] 1.7 Architectural Decision: GraphQL Mesh for Supergraph Composition

The Tome v8.3 describes the "Render, Commit, Run" workflow, where a
`supergraph.graphql` artifact is generated and committed to source control. The
plan executes this using the `npx @graphql-mesh/compose-cli` tool.

- **Rationale:** This is a deliberate architectural choice. While other tools
    (including services within the Hive platform) can perform composition, using
    the standalone GraphQL Mesh CLI offers key advantages for our CI/CD and
    developer workflow:
    1. **It is a lightweight, stateless build tool.** It requires no running
        services, databases, or external state, making it perfect for fast,
        ephemeral CI jobs.
    2. **It decouples composition from the runtime.** Our build process does not
        depend on our running `federated-graph-core` stack, which aligns
        perfectly with the goal of treating the supergraph artifact as a
        declarative build product.
    3. **It provides a consistent toolchain.** Using Mesh for both source
        adaptation (as in Phase 4) and supergraph composition simplifies the
        project's tooling dependencies.

### 1.8 Architectural Decision: Dependency Versioning Strategy

[ ] 1.8 Architectural Decision: Dependency Versioning Strategy

Throughout this implementation plan, the `version` field in the
`graphtastic.deps.yml` manifest is set to `main`. This is an explicit choice to
optimize for the developer "inner loop" during the initial build-out of the
platform.

- **Rationale:**
  - **For Development:** Using `main` allows developers working on the Hub to
        immediately pull in the latest changes from Spoke repositories without
        waiting for a formal release and version tag. This accelerates iterative
        development and cross-component work.
  - **For Production:** As mandated by the Tome (Sec 3.4, Precept 5), any Hub
        intended for a staging or production deployment **must** have its
        dependencies pinned to specific, immutable Git tags (e.g., `version:
        v1.3.0`). This plan focuses on the initial, local-first development
        workflow, and the transition to tagged versions is a critical step for
        production hardening.

---

## Part 2: The Detailed Execution Plan

[ ] Part 2: The Detailed Execution Plan

This plan is designed to be executed sequentially by an engineer. It assumes the
engineer has the following tools installed: `git`, `docker`, `docker compose`,
and `yq` (for YAML parsing in shell scripts). All work should be done within a
single parent directory.

### Phase 0: The Developer Control Plane

[ ] Phase 0: The Developer Control Plane

**Goal:** Create the foundational tooling that will govern and orchestrate all
subsequent development.

#### Task 0.1: Create `tools-docker-compose` Repository

[ ] Task 0.1: Create `tools-docker-compose` Repository

1. Create the repository directory: `mkdir -p tools-docker-compose/scripts`
2. Create the master `Makefile` at `tools-docker-compose/Makefile.master`:

    ```makefile
    # This is the master Makefile, located in `tools-docker-compose`.
    # It is called by a Hub's lightweight, root Makefile.

    # Default variables
    DEPS_DIR := ./.deps
    MANIFEST := ./graphtastic.deps.yml
    SUPERGRAPH_SDL := ./supergraph.graphql
    SHARED_NETWORK_NAME ?= $(shell grep SHARED_NETWORK_NAME .env | cut -d '=' -f2)

    # This allows passing a stack name, e.g., `make logs stack=subgraph-blogs`
    stack ?= all

    # Allow passing persistence mode, e.g., `make up mode=volume`
    mode ?= bind

    .PHONY: help setup up down clean restart deps supergraph validate ps logs seed

    help:
      @echo "Graphtastic Platform - Master Orchestrator"
      @echo "--------------------------------------------"
      @echo "Usage: make [target] [stack=<stack_name>] [mode=bind|volume]"
      @echo ""
      @echo "Lifecycle Targets:"
      @echo "  up                 - Start all services defined in the manifest."
      @echo "  down               - Stop and remove all services."
      @echo "  clean              - Run 'down' and remove shared Docker resources."
      @echo "  restart            - Restart all services."
      @echo "  logs               - Tail the logs of services. Use 'stack=' to target one."

    setup:
      @docker network inspect $(SHARED_NETWORK_NAME) >/dev/null 2>&1 || \

    up: setup deps seed
      @echo "🚀 Bringing up all services with persistence mode: $(mode)..."
      ./$(DEPS_DIR)/tools-docker-compose/scripts/manage-stacks.sh up $(MANIFEST) $(mode)

    down:
      @echo "🔥 Bringing down all services..."
      ./$(DEPS_DIR)/tools-docker-compose/scripts/manage-stacks.sh down $(MANIFEST)

    clean:
      @read -p "This will stop all services and permanently remove all local data for this supergraph. Are you sure? (y/N) " confirm && [ $${confirm:-N} = y ] || exit 1
      @docker network rm $(SHARED_NETWORK_NAME) 2>/dev/null || true

    restart: down up

    deps:
      @echo "🔄 Syncing component repositories from $(MANIFEST)..."
      ./$(DEPS_DIR)/tools-docker-compose/scripts/sync-deps.sh $(MANIFEST) $(DEPS_DIR)

    seed: deps
      @echo "🌱 Seeding data for stateful services..."
      ./$(DEPS_DIR)/tools-docker-compose/scripts/manage-stacks.sh seed $(MANIFEST)

    supergraph: deps
      @echo "✍️ Rendering supergraph artifact..."
      npx @graphql-mesh/compose-cli --config ./mesh.config.js > $(SUPERGRAPH_SDL)

    ps:
      @echo "📊 Status for stack: $(stack)"
      ./$(DEPS_DIR)/tools-docker-compose/scripts/manage-stacks.sh ps $(MANIFEST) $(stack)

    logs:
      @echo "📜 Tailing logs for stack: $(stack)..."
      ./$(DEPS_DIR)/tools-docker-compose/scripts/manage-stacks.sh logs $(MANIFEST) $(stack)
    ```

3. Create the dependency sync script at
    `tools-docker-compose/scripts/sync-deps.sh`:

    ```bash
    #!/bin/bash
    set -e
    MANIFEST=$1
    DEPS_DIR=$2

    if [ ! -f "$MANIFEST" ]; then
        echo "Manifest file not found: $MANIFEST"
        exit 1
    fi

    mkdir -p $DEPS_DIR

    yq e '.components[] | .name + " " + .git + " " + .version' $MANIFEST | while read -r name git version; do
        echo "--- Processing $name ---"
        TARGET_DIR="$DEPS_DIR/$name"
        if [ ! -d "$TARGET_DIR/.git" ]; then
            echo "Cloning $git..."
            git clone --branch $version $git $TARGET_DIR
        else
            echo "Fetching and checking out version $version..."
            (cd $TARGET_DIR && git fetch origin && git checkout $version && git pull origin $version --ff-only)
        fi
    done
    ```

4. Create the stack management script at
    `tools-docker-compose/scripts/manage-stacks.sh`:

    ```bash
    #!/bin/bash
    set -e
    ACTION=$1
    MANIFEST=$2
    PERSISTENCE_MODE=${3:-bind}
    STACK=${4:-all}
    SUPERGRAPH_NAME=$(yq e '.name' graphtastic.deps.yml)

    # This script now iterates for all actions to ensure project isolation.
    SPOKES=$(yq e '.components[] | select(.type == "spoke") | .name' $MANIFEST)

    for name in $SPOKES; do
      if [ "$STACK" = "all" ] || [ "$STACK" = "$name" ]; then
        # Each spoke is launched as a separate Docker Compose project for isolation
        COMPOSE_FILE="./.deps/$name/compose.yaml"
        if [ -f "$COMPOSE_FILE" ]; then
          PROJECT_NAME="${SUPERGRAPH_NAME}_$name"
          case "$ACTION" in
            up)
              echo "Bringing up $name as project $PROJECT_NAME..."
              PERSISTENCE_MODE=$PERSISTENCE_MODE docker compose -p $PROJECT_NAME -f $COMPOSE_FILE up -d
              ;;
            down)
              echo "Bringing down $name as project $PROJECT_NAME..."
              PERSISTENCE_MODE=$PERSISTENCE_MODE docker compose -p $PROJECT_NAME -f $COMPOSE_FILE down
              ;;
            ps|logs)
              echo "Showing $ACTION for $name (project $PROJECT_NAME)..."
              PERSISTENCE_MODE=$PERSISTENCE_MODE docker compose -p $PROJECT_NAME -f $COMPOSE_FILE $ACTION
              ;;
            seed)
              SPOKE_MAKEFILE="./.deps/$name/Makefile"
              if [ -f "$SPOKE_MAKEFILE" ] && grep -q "^seed:" "$SPOKE_MAKEFILE"; then
                echo "--- Seeding data for $name ---"
                (cd "./.deps/$name" && make seed)
              fi
              ;;
            *)
              echo "Unknown action: $ACTION"
              exit 1
              ;;
          esac
        fi
      fi
    done
    ```

5. Make scripts executable: `chmod +x tools-docker-compose/scripts/*.sh`

#### Task 0.2: Create `tools-subgraph-core` Repository

[ ] Task 0.2: Create `tools-subgraph-core` Repository

1. Create the repository directory: `mkdir -p tools-subgraph-core/.github/workflows`
2. Create a master Makefile for subgraphs at
    `tools-subgraph-core/Makefile.subgraph.master`:

    ```makefile
    # This is the master Makefile for subgraphs.
    # It is included by a Spoke's lightweight, root Makefile.
    .PHONY: validate-federation

    validate-federation:
        @echo "🔎 Validating schema for federation readiness..."
        # In a real implementation, this would use GraphQL Inspector
        @echo "✅ Schema is valid (placeholder)."
    ```

3. Create the reusable CI workflow at
    `tools-subgraph-core/.github/workflows/subgraph-ci.yml`:

    ```yaml
    name: Subgraph CI
    on:
      workflow_call:
    jobs:
      validate:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
          - name: Validate Federation Readiness
            run: make validate-federation # This command is provided by the included Makefile
    ```

#### Task 0.3: Create `template-spoke-base` Repository

[ ] Task 0.3: Create `template-spoke-base` Repository

1. Create the repository directory: `mkdir -p template-subgraph/.github/workflows`
2. Create a placeholder `compose.yaml`:

    ```yaml
    # template-subgraph/compose.yaml
    version: "3.8"
    services:
      # Service definition will go here
    ```

3. Create a placeholder `schema.graphql`:

    ```graphql
    # template-subgraph/schema.graphql
    type Query {
      hello: String
    }
    ```

4. Create a `.env.template` to establish the secret management pattern:

    ```env
    # Environment variables for this subgraph
    # Copy to .env for local development
    ```

5. Create a lightweight `Makefile` that includes the master tooling. *Note: This
    assumes a directory structure where tool repos are peers to Hub repos.*

    ```makefile
    # template-spoke-base/Makefile
    TOOLS_DIR := ./.tools
    TOOLS_REPO := https://github.com/graphtastic/tools-subgraph-core.git # Use actual path
    TOOLS_VERSION := main

    -include $(TOOLS_DIR)/tools-subgraph-core/Makefile.subgraph.master

    .PHONY: tools

    tools:
      @if [ ! -d "$(TOOLS_DIR)/tools-subgraph-core" ]; then \
      fi
    ```

6. Create the CI workflow that *calls* the reusable workflow:

    ```yaml
    # template-subgraph/.github/workflows/ci.yml
    name: CI

    on: [pull_request]

    jobs:
      validate:
        uses: graphtastic/tools-subgraph-core/.github/workflows/subgraph-ci.yml@main
        secrets: inherit
    ```

#### Task 0.4: Create `template-supergraph` Repository

[ ] Task 0.4: Create `template-supergraph` Repository

1. Create the directory: `mkdir template-supergraph`
2. Create a root `Makefile`:

    ```makefile
    # Graphtastic Supergraph Makefile
    # This file delegates all logic to the master makefile from tools-docker-compose.
    include ./.deps/tools-docker-compose/Makefile.master
    ```

3. Create a placeholder `graphtastic.deps.yml` with the new `type` field:

    ```yaml
    # Declare dependencies for this supergraph
    components:
      - name: tools-docker-compose
        git: https://github.com/graphtastic/tools-docker-compose.git # Use actual path
        version: main
        type: tool
    ```

#### Task 0.5: Create `template-spoke-dgraph` Repository

[ ] Task 0.5: Create `template-spoke-dgraph` Repository

**Goal:** Create a reusable template for any Spoke backed by Dgraph.

1. Create the directory: `mkdir template-spoke-dgraph`
2. Create a placeholder `schema.graphql` file.
3. Create the canonical `compose.yaml` for a Dgraph stack. This version uses
    the declarative `--schema` flag for provisioning.

    ```yaml
    # template-spoke-dgraph/compose.yaml
    version: "3.8"
    services:
      zero:
        image: dgraph/dgraph:latest
        volumes:
          - type: ${PERSISTENCE_MODE:-bind}
            source: ./data/zero
            target: /dgraph
        # ... ports, restart, command ...
        networks: [graphtastic_net]
      alpha:
        image: dgraph/dgraph:latest
        volumes:
          - type: ${PERSISTENCE_MODE:-bind}
            source: ./data/alpha
            target: /dgraph
          # Mount the schema file for declarative loading
          - type: bind
            source: ./schema.graphql
            target: /dgraph/schema.graphql
            read_only: true
        # ... ports, restart, depends_on ...
        # Use the --schema flag in the startup command
        command: dgraph alpha --my=alpha:7080 --zero=zero:5080 --schema /dgraph/schema.graphql
        networks: [graphtastic_net]
      # ... ratel service ...
    networks:
      graphtastic_net:
        name: ${SHARED_NETWORK_NAME:-graphtastic_net}
        external: true
    ```

4. Create a standard `Makefile` for managing this Spoke. The `seed` target is
    now simplified as schema is applied automatically.

#### Task 0.6: Create `template-spoke-mesh` Repository

[ ] Task 0.6: Create `template-spoke-mesh` Repository

**Goal:** Create a reusable template for any Spoke using the Transformation
Sidecar pattern.

1. Create the repository directory: `mkdir template-spoke-mesh`
2. Create a placeholder `.meshrc.yml`:

    ```yaml
    # template-spoke-mesh/.meshrc.yml
    sources:
      - name: MyUpstream
        handler:
          graphql:
            endpoint: http://my-internal-service:8080/graphql
    ```

3. Create a `compose.yaml` to run the Mesh gateway:

    ```yaml
    # template-spoke-mesh/compose.yaml
    version: "3.8"
    services:
      mesh-gateway:
        image: ghcr.io/the-guild-org/graphql-mesh/mesh-gateway:latest
        # ... ports, volumes to mount config, etc ...
        networks: [graphtastic_net]
    networks:
      graphtastic_net:
        name: ${SHARED_NETWORK_NAME:-graphtastic_net}
        external: true
    ```

### Phase 1: Minimal Viable Federation with Stateless Spokes

[ ] Phase 1: Minimal Viable Federation with Stateless Spokes

**Goal:** Create and federate two simple, stateless Yoga-based subgraphs.

#### Task 1.1: Create `subgraph-blogs`

[ ] Task 1.1: Create `subgraph-blogs`

1. **Instantiate the Spoke Repository:**
    - On your Git provider (e.g., GitHub), create a new repository named
        `subgraph-blogs` using `graphtastic/template-subgraph` as the template.
    - Clone the newly created repository: `git clone
        <your-repo-url>/subgraph-blogs.git`

2. Navigate into the new directory: `cd subgraph-blogs`

3. Create the Yoga server at `index.js`:

    ```javascript
    import { createServer } from 'node:http'
    import { createYoga } from 'graphql-yoga'
    import { createSchema } from 'graphql-yoga'

    const typeDefs = /* GraphQL */ `
      type Query {
        blogs: [Blog!]
        blog(id: ID!): Blog
      }

      type Blog @key(fields: "id") {
        id: ID!
        title: String!
        author: Author
      }

      # We are extending the Author type, which is owned by the authors subgraph.
      # The @key directive tells the gateway how to fetch an Author.
      # The @extends directive tells the gateway this type is defined elsewhere.
      type Author @key(fields: "id") @extends {
        id: ID! @external
      }
    `;

    const blogsData = [
      { id: '1', title: 'Introduction to GraphQL', authorId: '101' },
      { id: '2', title: 'Federation Deep Dive', authorId: '102' },
    ];

    const resolvers = {
      Query: {
        blogs: () => blogsData,
        blog: (_, { id }) => blogsData.find(b => b.id === id),
      },
      Blog: {
        // This resolver provides the "link" to the Author type.
        // It returns a representation containing the key field ('id').
        // The gateway will use this to fetch the full Author from the other subgraph.
        author: (blog) => {
          return { __typename: "Author", id: blog.authorId };
        }
      }
    };

    const yoga = createYoga({ schema: createSchema({ typeDefs, resolvers }) });
    const server = createServer(yoga);
    const PORT = process.env.PORT || 4001;
    server.listen(PORT, () => { console.info(`Blogs subgraph running at http://localhost:${PORT}/graphql`) });
    ```

4. Update `subgraph-blogs/schema.graphql` with the schema from the `typeDefs`
    above.

5. Create a `subgraph-blogs/package.json`:

    ```json
    {
      "name": "subgraph-blogs",
      "type": "module",
      "dependencies": {
        "graphql": "^16.8.1",
        "graphql-yoga": "^5.1.1"
      }
    }
    ```

6. Update `subgraph-blogs/compose.yaml` (from the template):

    ```yaml
    version: "3.8"
    services:
      blogs-service:
        build: .
        ports: ["${SUBGRAPH_BLOGS_PORT:-4001}:4001"]
        networks: [graphtastic_net]
    networks:
      graphtastic_net:
        name: ${SHARED_NETWORK_NAME:-graphtastic_net}
        external: true
    ```

7. Create `subgraph-blogs/Dockerfile`:

    ```dockerfile
    FROM node:18-alpine
    WORKDIR /app
    COPY package.json .
    RUN npm install
    COPY . .
    CMD ["node", "index.js"]
    ```

8. Commit and push these changes to the `subgraph-blogs` repository.

#### Task 1.2: Create `subgraph-authors`

[ ] Task 1.2: Create `subgraph-authors`

1. Follow the same Git-based procedure as in Task 1.1 to create and clone a new
    `subgraph-authors` repository from the template.

2. Create the Yoga server, schema, `package.json`, `compose.yaml`, and
    `Dockerfile` for Authors. Note that this subgraph *owns* the `Author` type.

    - **`index.js`:**

        ```javascript
        import { createServer } from 'node:http'
        import { createYoga } from 'graphql-yoga'
        import { createSchema } from 'graphql-yoga'

        const typeDefs = /* GraphQL */`
          type Query {
            authors: [Author!]
            author(id: ID!): Author
          }

          # This is the canonical definition of the Author type.
          # The @key directive marks 'id' as the field other subgraphs can use to reference it.
          type Author @key(fields: "id") {
            id: ID!
            name: String!
          }
        `;
        const authorsData = [
          { id: '101', name: 'Alice' },
          { id: '102', name: 'Bob' }
        ];
        const resolvers = {
          Query: {
            authors: () => authorsData,
            author: (_, {id}) => authorsData.find(a => a.id === id),
          }
        };
        const yoga = createYoga({ schema: createSchema({ typeDefs, resolvers }) });
        const server = createServer(yoga);
        const PORT = process.env.PORT || 4002;
        server.listen(PORT, () => { console.info(`Authors subgraph running at http://localhost:${PORT}/graphql`)});
        ```

    - Update the corresponding `schema.graphql` file.
    - Create the `package.json` file, identical to the blogs subgraph but with
        the name `subgraph-authors`.
    - Create the `Dockerfile`, identical to the blogs subgraph.
    - Update `compose.yaml`:

        ```yaml
        version: "3.8"
        services:
          authors-service:
            build: .
            ports: ["${SUBGRAPH_AUTHORS_PORT:-4002}:4002"]
            networks: [graphtastic_net]
        networks:
          graphtastic_net:
            name: ${SHARED_NETWORK_NAME:-graphtastic_net}
            external: true
        ```

3. Commit and push the changes to the `subgraph-authors` repository.

#### Task 1.3: Create `federated-graph-core`

[ ] Task 1.3: Create `federated-graph-core`

1. Create the directory for this local-only component: `mkdir
    federated-graph-core`
2. Create the `compose.yaml` for the Hive stack at
    `federated-graph-core/compose.yaml`:

    ```yaml
    version: '3.8'
    services:
      gateway:
        image: ghcr.io/the-guild-org/graphql-hive/gateway:latest
        ports:
          - '4000:4000'
        environment:
          HIVE_CDN_ENDPOINT: 'http://localhost:3002' # Placeholder, not used in static mode
          HIVE_CDN_KEY: 'dummy-key'
          SUPERGRAPH: '/app/supergraph.graphql' # Path inside container
        volumes:
          # This volume mount will be configured by the supergraph's override file
          - type: bind
            source: ./supergraph.graphql
            target: /app/supergraph.graphql
            read_only: true
        networks:
          - graphtastic_net
    networks:
      graphtastic_net:
        name: ${SHARED_NETWORK_NAME:-graphtastic_net}
        external: true
    ```

#### Task 1.4: Create and Configure `supergraph-cncf`

[ ] Task 1.4: Create and Configure `supergraph-cncf`

1. Instantiate the Hub repository from the `template-supergraph` and clone it
    locally. We will work inside this directory for the remainder of the plan.
2. Update `supergraph-cncf/graphtastic.deps.yml` to declare all dependencies. We
    use local relative paths for components we just created.

    ```yaml
    components:
      - name: tools-docker-compose
        git: ../tools-docker-compose
        version: main
        type: tool
      - name: tools-subgraph-core
        git: ../tools-subgraph-core
        version: main
        type: tool
      - name: federated-graph-core
        git: ../federated-graph-core
        version: main
        type: spoke
      - name: subgraph-blogs
        git: ../subgraph-blogs
        version: main
        type: spoke
      - name: subgraph-authors
        git: ../subgraph-authors
        version: main
        type: spoke
    ```

3. Create `supergraph-cncf/.env` to provide centralized configuration for the
    entire system:

    ```env
    SHARED_NETWORK_NAME=graphtastic_net
    SUBGRAPH_BLOGS_PORT=4001
    SUBGRAPH_AUTHORS_PORT=4002
    ```

4. Create `supergraph-cncf/mesh.config.js` to define how the supergraph is
    composed:

    ```javascript
    module.exports = {
      sources: [
        { name: 'blogs', handler: { graphql: { endpoint: 'http://blogs-service:4001/graphql' } } },
        { name: 'authors', handler: { graphql: { endpoint: 'http://authors-service:4002/graphql' } } },
      ],
      compose: { supergraph: true },
    };
    ```

5. Create `supergraph-cncf/compose.override.yaml` to mount the generated
    supergraph into the gateway container:

    ```yaml
    services:
      gateway:
        volumes:
          - type: bind
            source: ./supergraph.graphql
            target: /app/supergraph.graphql
            read_only: true
    ```

6. **Execute and Verify:**
    - From the `supergraph-cncf` directory:
    - Create the shared network: `docker network create graphtastic_net`
    - Render the supergraph artifact: `make supergraph` (This will create
        `supergraph.graphql`)
    - Start the entire federated system: `make up`
    - Open your browser or GraphQL client to `http://localhost:4000/graphql`
        and run a federated query:

        ```graphql
        query GetBlogsWithAuthors {
          blogs {
            id
            title
            author {
              name
            }
          }
        }
        ```

    - The query should succeed, demonstrating that the gateway was able to
        fetch the `blog` and its `authorId` from the blogs subgraph, and then
        use that `id` to fetch the `name` from the authors subgraph. This
        completes the minimal viable federation.

### Phase 2: Simple Stateful Backend (Dgraph "Hello World")

[ ] Phase 2: Simple Stateful Backend (Dgraph "Hello World")

**Goal:** Introduce Dgraph into the ecosystem in the most controlled way
possible, validating the base technology and the pattern for a stateful Spoke.

#### Task 2.1: Create `subgraph-dgraph-static` Spoke

[ ] Task 2.1: Create `subgraph-dgraph-static` Spoke

1. **Instantiate the Spoke Repository:** Create a new repository named
    `subgraph-dgraph-static` from the `graphtastic/template-subgraph` template
    and clone it locally.
2. Navigate into the new directory: `cd subgraph-dgraph-static`
3. Establish the Dgraph data directory structure: `mkdir -p ./data/{zero,alpha}`
4. Create the Dgraph schema at `schema.graphql`. This schema includes both
    Dgraph's `@id` directive for unique key enforcement and the Federation `@key`
    directive for compatibility with the supergraph.

    ```graphql
    # Dgraph schema for static Star Trek character data
    type Character @key(fields: "id") {
      id: ID! @id
      name: String! @search(by: [term])
      species: String! @search(by: [hash])
      affiliation: String
    }
    ```

5. Update the `compose.yaml` file. This configuration is a direct
    implementation of best practices, adapted for our Spoke architecture.

    ```yaml
    version: "3.8"
    services:
      zero:
        image: dgraph/dgraph:latest
        container_name: dgraph_static_zero
        volumes:
          - type: ${PERSISTENCE_MODE:-bind}
            source: ${PERSISTENCE_PATH:-./data/zero}
            target: /dgraph
        ports:
          - "5081:5080" # Use non-default ports to avoid host conflicts
          - "6081:6080"
        restart: on-failure
        command: dgraph zero --my=zero:5080
        networks:
          - graphtastic_net

      alpha:
        image: dgraph/dgraph:latest
        container_name: dgraph_static_alpha
        volumes:
          - type: ${PERSISTENCE_MODE:-bind}
            source: ${PERSISTENCE_PATH:-./data/alpha}
            target: /dgraph
        ports:
          - "8081:8080" # Expose GraphQL endpoint
          - "9081:9080"
        restart: on-failure
        command: dgraph alpha --my=alpha:7080 --zero=zero:5080
        depends_on:
          - zero
        networks:
          - graphtastic_net

      ratel:
        image: dgraph/ratel:latest
        container_name: dgraph_static_ratel
        ports:
          - "8001:8000" # Expose Ratel UI
        restart: on-failure
        networks:
          - graphtastic_net

    networks:
      graphtastic_net:
        name: ${SHARED_NETWORK_NAME:-graphtastic_net}
        external: true
    ```

6. Create the static dataset at `data.json`. This file contains the data to be
    loaded.

    ```json
    [
      {"uid": "_:DS9-001", "dgraph.type": "Character", "id": "DS9-001", "name": "Benjamin Sisko", "species": "Human", "affiliation": "Starfleet"},
      {"uid": "_:DS9-002", "dgraph.type": "Character", "id": "DS9-002", "name": "Kira Nerys", "species": "Bajoran", "affiliation": "Bajoran Militia"},
      {"uid": "_:TNG-001", "dgraph.type": "Character", "id": "TNG-001", "name": "Jean-Luc Picard", "species": "Human", "affiliation": "Starfleet"}
    ]
    ```

7. Update the `Makefile` within the Spoke to manage its lifecycle, including
    schema/data loading and the conventional `seed` target. This removes the
    `curl` of dynamic data and replaces it with a declarative `mutate` of a
    local file.

    ```makefile
    # subgraph-dgraph-static/Makefile
    include ../tools-subgraph-core/Makefile.subgraph.master

    .PHONY: up down clean load-data seed

    up:
        docker compose up -d

    down:
        docker compose down

    clean: down
        @echo "🧹 Removing local data..."
        rm -rf ./data

    load-data:
        @echo "Loading static character data from data.json..."
        # Use the /mutate endpoint for raw JSON loading with commitNow
        curl -s -X POST localhost:8081/mutate?commitNow=true -H "Content-Type: application/json" -d "@data.json"

    seed: load-data
    ```

8. Commit and push these changes to the `subgraph-dgraph-static` repository.
9. **Standalone Verification:**
    - From within the `subgraph-dgraph-static` directory:
    - `docker compose up -d`
    - Wait a few seconds for the cluster to stabilize, then: `make seed`
    - Verify data by running a query against `http://localhost:8081/graphql`.
    - `docker compose down`

#### Task 2.2: Integrate `subgraph-dgraph-static` into `supergraph-cncf`

[ ] Task 2.2: Integrate `subgraph-dgraph-static` into `supergraph-cncf`

1. Navigate back to the `supergraph-cncf` directory.
2. Update `graphtastic.deps.yml` to include the new Spoke:

    ```yaml
      # ... existing components
      - name: subgraph-dgraph-static
        git: ../subgraph-dgraph-static
        version: main
        type: spoke
    ```

3. Update `mesh.config.js` to add the Dgraph subgraph as a source. Note the use
    of the `alpha` service name and its internal port `8080`.

    ```javascript
    // ... existing sources
    {
      name: 'characters',
      handler: {
        graphql: { endpoint: 'http://alpha:8080/graphql' } // Dgraph Alpha's endpoint
      }
    },
    ```

4. To demonstrate a federated join, we will extend the `Author` type in
    `subgraph-authors` to have a favorite character.
    - Navigate to the local clone of the authors subgraph: `cd
        ../subgraph-authors`
    - Update `subgraph-authors/index.js` to add `favoriteCharacter` to the
        `Author` type and extend the `Character` type:

        ```javascript
        // ... inside typeDefs
        type Author @key(fields: "id") {
          id: ID!
          name: String!
          favoriteCharacter: Character
        }
        type Character @key(fields: "id") @extends {
          id: ID! @external
        }

        // ... inside resolvers
        Author: {
            favoriteCharacter: (author) => {
                // In a real app, this would be from a database. Here we stub it.
                const favs = { '101': 'DS9-001', '102': 'TNG-001' };
                return { __typename: 'Character', id: favs[author.id] };
            }
        }
        ```

    - Update the corresponding `schema.graphql` file.
    - Commit and push these changes to the `subgraph-authors` remote
        repository.
5. **Execute and Verify:**
    - From the `supergraph-cncf` directory:
    - The query should succeed, demonstrating that the gateway was able to
        fetch the `blog` and its `authorId` from the blogs subgraph, and then
        use that `id` to fetch the `name` from the authors subgraph. This
        completes the minimal viable federation.

    *(No need to create the shared network manually; this is now automated.)*
    - `make up` (This will start all services and automatically run `make seed`
        on the `subgraph-dgraph-static` Spoke)
    - Run a federated query against `http://localhost:4000/graphql`:

        ```graphql
        query GetAuthorsWithFavoriteCharacters {
          authors {
            id
            name
            favoriteCharacter {
              name
              species
            }
          }
        }
        ```

    - This query should succeed, proving that the gateway can join data from
        the stateless `authors` subgraph with the stateful Dgraph `characters`
        subgraph.

### Phase 3: Complex Stateful Backend (Dynamic Data Ingestion)

[ ] Phase 3: Complex Stateful Backend (Dynamic Data Ingestion)

**Goal:** Build upon the Dgraph foundation by creating a Spoke that handles a
dynamic data loading pipeline from an external source (`gharchive`).

#### Task 3.1: Create `subgraph-dgraph-gharchive` Spoke

[ ] Task 3.1: Create `subgraph-dgraph-gharchive` Spoke

1. **Instantiate the Spoke Repository:** Because this is a Dgraph-based Spoke,
    the `subgraph-dgraph-static` repository is a better starting point than the
    base template. Create a new repository named `subgraph-dgraph-gharchive`
    using `graphtastic/subgraph-dgraph-static` as the template, then clone it
    locally.
2. Navigate into the new directory: `cd subgraph-dgraph-gharchive`
3. Update the schema at `schema.graphql` for GHArchive data:

    ```graphql
    type PushEvent @key(fields: "id") {
      id: String! @id
      actor: String! @search(by: [hash])
      repo: String! @search(by: [term])
      commitCount: Int
      createdAt: DateTime! @search(by: [hour])
    }
    ```

4. Create a directory for the ingestion script: `mkdir ./loader`
5. Create a `package.json` for the loader script at `loader/package.json`:

    ```json
    {
      "name": "gharchive-loader",
      "type": "module",
      "dependencies": {
        "node-fetch": "^3.3.2",
        "zlib": "^1.0.5"
      }
    }
    ```

6. Create the ingestion script at `loader/index.js`. This script fetches,
    parses, transforms, and loads data in batches. Note the use of
    `process.env.DGRAPH_ENDPOINT` to avoid hardcoding.

    ```javascript
    import fetch from 'node-fetch';
    import { createGunzip } from 'zlib';
    import { Writable } from 'stream';
    import { pipeline } from 'stream/promises';

    const DGRAPH_ALPHA_URL = process.env.DGRAPH_ENDPOINT || 'http://localhost:8082/graphql';
    const BATCH_SIZE = 1000;

    async function main() {
      // Fetch data for a specific hour, e.g., 2025-09-05 at 15:00 UTC
      const response = await fetch('https://data.gharchive.org/2025-09-05-15.json.gz');
      if (!response.ok) throw new Error(`Failed to fetch: ${response.statusText}`);

      let buffer = [];
      const jsonlParser = new Writable({
        async write(chunk, encoding, callback) {
          const lines = chunk.toString().split('\n');
          for (const line of lines) {
            if (!line) continue;
            try {
              const event = JSON.parse(line);
              if (event.type === 'PushEvent') {
                buffer.push({
                  id: event.id,
                  actor: event.actor.login,
                  repo: event.repo.name,
                  commitCount: event.payload.commits?.length || 0,
                  createdAt: event.created_at
                });

                if (buffer.length >= BATCH_SIZE) {
                  await loadBatch(buffer);
                  buffer = [];
                }
              }
            } catch (e) { /* Ignore malformed JSON lines */ }
          }
          callback();
        }
      });

      await pipeline(response.body, createGunzip(), jsonlParser);
      if (buffer.length > 0) await loadBatch(buffer); // Load final batch
      console.log('Ingestion complete.');
    }

    async function loadBatch(events) {
      const mutation = `
        mutation AddPushEvents($events: [AddPushEventInput!]!) {
          addPushEvent(input: $events, upsert: true) {
            numUids
          }
        }
      `;
      const response = await fetch(DGRAPH_ALPHA_URL, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ query: mutation, variables: { events } }),
      });
      const result = await response.json();
      console.log(`Loaded batch of ${events.length}. Response:`, result);
    }

    main().catch(console.error);
    ```

7. Update `compose.yaml` to use different host ports (e.g., `8082`, `9082`,
    `5082`, `6082`, `8002`) to avoid conflicts with the other Dgraph Spoke.
8. Update the `Makefile` to manage the loader script and provide the `seed`
    target.

    ```makefile
    # subgraph-dgraph-gharchive/Makefile
    include ../tools-subgraph-core/Makefile.subgraph.master

    .PHONY: up down clean apply-schema install-loader load-data seed

    # ... (up, down, clean targets are standard) ...

    apply-schema:
        curl -X POST localhost:8082/admin/schema --data-binary "@schema.graphql"

    install-loader:
        cd loader && npm install

    load-data: install-loader
        @echo "Loading GHArchive data for a sample hour..."
        DGRAPH_ENDPOINT="http://localhost:8082/graphql" node loader/index.js

    seed: apply-schema load-data
    ```

9. Commit and push these changes to the `subgraph-dgraph-gharchive` repository.

#### Task 3.2: Integrate `subgraph-dgraph-gharchive` into `supergraph-cncf`

[ ] Task 3.2: Integrate `subgraph-dgraph-gharchive` into `supergraph-cncf`

1. Navigate to the `supergraph-cncf` directory.
2. Add the new Spoke to `graphtastic.deps.yml`.

    ```yaml
      # ... existing components
      - name: subgraph-dgraph-gharchive
        git: ../subgraph-dgraph-gharchive
        version: main
        type: spoke
    ```

3. Update `mesh.config.js` to add `gharchive` as a source, pointing to its
    Dgraph Alpha service on its internal port `8080`.

    ```javascript
    // ... existing sources, including 'characters'
    {
      name: 'gharchive',
      handler: {
        graphql: { endpoint: 'http://alpha:8080/graphql' } // This still works due to service name resolution
      }
    },
    ```

4. Re-render the supergraph (`make supergraph`) and restart the system (`make
    up`). The `make up` command will now automatically seed both Dgraph Spokes.
5. Verify with a query against `http://localhost:4000/graphql` that filters for
    `PushEvent` data to confirm the new Spoke is federated and contains data.

---

### Phase 4: Advanced Composite Spoke (ETL Pipeline)

[ ] Phase 4: Advanced Composite Spoke (ETL Pipeline)

**Goal:** Implement the complex `software-supply-chain` Spoke, which is itself a
composite system involving an external data source (GUAC) and a sophisticated
ETL process.

#### Task 4.1: Create `subgraph-dgraph-software-supply-chain` Spoke

[ ] Task 4.1: Create `subgraph-dgraph-software-supply-chain` Spoke

1. **Instantiate the Spoke Repository:** Create a new repository named
    `subgraph-dgraph-software-supply-chain` from the
    `graphtastic/subgraph-dgraph-static` template and clone it locally.
2. Navigate into the new directory: `cd subgraph-dgraph-software-supply-chain`
3. Update the Spoke's `compose.yaml`. For isolated development, it will run its
    own local instances of Dgraph and GUAC. Use non-conflicting ports (e.g.,
    8083 for Dgraph alpha, 8085 for GUAC GraphQL).

    ```yaml
    # In subgraph-dgraph-software-supply-chain/compose.yaml
    version: "3.8"
    services:
      # Dgraph Zero, Alpha, Ratel services on non-conflicting ports (e.g. 8083)
      # ... (Copy from subgraph-dgraph-static/compose.yaml and update ports)

      # GUAC services for local development
      guac-postgres:
        image: postgres:13
        container_name: guac_postgres_local
        # ... env vars, volumes ...
        networks:
          - default # GUAC services communicate on their own internal network
      guac-nats:
        image: nats:2.9
        container_name: guac_nats_local
        networks:
          - default
      # ... other GUAC collector/assembler services ...
      guac-graphql:
        image: guacsec/guac-graphql:latest # Use official GUAC image
        container_name: guac_graphql_local
        ports: ["8085:8080"] # Expose GUAC GraphQL API for the ETL script to use on localhost
        networks:
          - default

    networks:
      graphtastic_net:
        name: ${SHARED_NETWORK_NAME:-graphtastic_net}
        external: true
    ```

4. Create the ETL script directory: `mkdir ./etl`
5. Implement the ETL script (`etl/index.js`) and any necessary schema
    augmentation logic as detailed in `design--guac-to-dgraph.md`. This script
    will be significantly more complex, performing a two-stage extraction
    (Nodes, then Edges) from `http://localhost:8085` (the exposed GUAC port) and
    writing to an RDF file inside the `./data` directory.
6. Update the Spoke's `Makefile` to orchestrate the entire ETL process and add
    the conventional `seed` target.

    ```makefile
    # subgraph-dgraph-software-supply-chain/Makefile
    include ../tools-subgraph-core/Makefile.subgraph.master

    .PHONY: etl seed

    etl:
        @echo "Running GUAC to Dgraph ETL process..."
        # 1. Run the extractor script to query GUAC and generate guac-data.rdf.gz
        node etl/index.js --source-url http://localhost:8085 --output-file ./data/guac-data.rdf.gz

        # 2. Use Dgraph Live Loader to ingest the RDF file
        docker compose exec alpha dgraph live --files /dgraph/guac-data.rdf.gz --alpha alpha:7080 --zero zero:5080

    seed: etl
    ```

7. Commit and push these changes to the `subgraph-dgraph-software-supply-chain`
    repository.

#### Task 4.2: Integrate into `supergraph-cncf`

[ ] Task 4.2: Integrate into `supergraph-cncf`

1. Navigate to the `supergraph-cncf` directory.
2. Add the Spoke to `graphtastic.deps.yml`.

    ```yaml
      # ... existing components
      - name: subgraph-dgraph-software-supply-chain
        git: ../subgraph-dgraph-software-supply-chain
        version: main
        type: spoke
    ```

3. Update `mesh.config.js`, adding a source for the software supply chain data.

    ```javascript
    // ... existing sources
    {
      name: 'supplychain',
      handler: {
        graphql: { endpoint: 'http://alpha:8080/graphql' }
      }
    },
    ```

4. Re-render the supergraph (`make supergraph`) and restart (`make up`).
5. Verify with a complex query against `http://localhost:4000/graphql` joining
    data across multiple subgraphs to prove the integration is successful.

---

### Phase 5: Production Hardening and Polish

[ ] Phase 5: Production Hardening and Polish

**Goal:** Implement the remaining production-readiness features from the Tome
across the entire platform.

#### Task 5.1: Supergraph CI/CD Hardening

[ ] Task 5.1: Supergraph CI/CD Hardening

1. Create the CI workflow file at `supergraph-cncf/.github/workflows/ci.yml`:

    ```yaml
    name: Supergraph CI
    on: [pull_request]
    jobs:
      validate:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
          - name: Setup Node
            uses: actions/setup-node@v4
            with: { node-version: '18' }
          - name: Install Dependencies
            # Install yq from npm for a self-contained CI environment
            run: npm install -g @graphql-mesh/compose-cli yq
          - name: Sync Deps & Render Supergraph
            run: make supergraph
          - name: Check for Stale Supergraph Artifact
            run: |
              git diff --exit-code supergraph.graphql
              echo "✅ supergraph.graphql is up-to-date."
    ```

#### Task 5.2: Secret Management Implementation

[ ] Task 5.2: Secret Management Implementation

1. Modify `federated-graph-core/compose.yaml` to use an environment variable
    for a database password.

    ```yaml
    services:
      # This is a simplified example. A real Hive stack has many services.
      postgres:
        image: postgres:15
        environment:
          POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-hive} # Default to 'hive' if not set
    ```

2. Create `.env.template` in the `federated-graph-core` repository:

    ```env
    # Secrets for the federated-graph-core stack
    # Copy this file to .env and fill in your values for local development.
    POSTGRES_PASSWORD=
    ```

3. *Note: The pattern for subgraphs was already established in Phase 0 with the
    update to `template-subgraph`.*

#### Task 5.3: Configurable Persistence Implementation

[ ] Task 5.3: Configurable Persistence Implementation

1. Modify a stateful Spoke's `compose.yaml` (e.g.,
    `subgraph-dgraph-static/compose.yaml`) to support configurable persistence.

    ```yaml
    # ...
    services:
      alpha:
        volumes:
          - type: ${PERSISTENCE_MODE:-bind}
            source: ./data/alpha # Note: PERSISTENCE_PATH is not needed if relative
            target: /dgraph
    # ...
    ```

2. The `manage-stacks.sh` script and `Makefile.master` from **Phase 0** are
    already designed to pass the `PERSISTENCE_MODE` variable, so no further
    changes are needed in the orchestration layer. A developer can now run `make
    up mode=volume` to switch to named volumes.
