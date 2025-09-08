# Implementation Plan: 04 - Advanced Patterns and Platform Hardening

**Goal:** Build upon the stable federated core by introducing advanced Spokes (including the GUAC/Mesh transformation sidecar), implementing production-readiness features, and creating boilerplate templates to accelerate future development.

This plan is designed to be executed sequentially. It assumes you have a working supergraph from the previous plan and are executing commands from within the correct repository's root directory.

## Task Checklist

- [ ] **Phase 3: Advanced Schema Augmentation (The GHArchive Case Study)**
  - [ ] [Task 3.1: Create `subgraph-dgraph-gharchive` Spoke](#task-31-create-subgraph-dgraph-gharchive-spoke)
- [ ] **Phase 4: Advanced Schema Correction (The GUAC Case Study)**
  - [ ] [Task 4.1: Create `subgraph-mesh-guac` (The Corrective Lens)](#task-41-create-subgraph-mesh-guac-the-corrective-lens)
  - [ ] [Task 4.2: Create `graphtastic-rdf-exporter` CLI Tool](#task-42-create-graphtastic-rdf-exporter-cli-tool)
  - [ ] [Task 4.3: Create `subgraph-dgraph-software-supply-chain` (The Persistent Store)](#task-43-create-subgraph-dgraph-software-supply-chain-the-persistent-store)
- [ ] **Phase 5: Production Hardening and Polish**
  - [ ] [Task 5.1: Supergraph CI/CD Hardening](#task-51-supergraph-cicd-hardening)
  - [ ] [Task 5.2: Secret Management Implementation](#task-52-secret-management-implementation)
  - [ ] [Task 5.3: Configurable Persistence Implementation](#task-53-configurable-persistence-implementation)
- [ ] **Phase 6: Template Creation (Deferred from Phase 0)**
  - [ ] [Task 0.3: Create `template-spoke-base` Repository Content](#task-03-create-template-spoke-base-repository-content)
  - [ ] [Task 0.4: Create `template-supergraph` Repository Content](#task-04-create-template-supergraph-repository-content)
  - [ ] [Task 0.5: Create `template-spoke-dgraph` Repository Content](#task-05-create-template-spoke-dgraph-repository-content)
  - [ ] [Task 0.6: Create `template-spoke-mesh` Repository Content](#task-06-create-template-spoke-mesh-repository-content)

---

### Phase 3: Advanced Schema Augmentation (The GHArchive Case Study)

#### Task 3.1: Create `subgraph-dgraph-gharchive` Spoke

1.  **Modify the `compose.yaml`:** In the `subgraph-dgraph-gharchive` repository, modify the `compose.yaml` to include the standard Dgraph service stack (`zero`, `alpha`, `ratel`) on an internal `default` network. The `mesh-gateway` service will now be the only service exposed to `graphtastic_net`.
2.  **Implement "Schema Archeology":**
    *   Create `src/gharchive.projection.graphql` containing the manually projected, focused schema based on analysis of the event data and the official GitHub SDL.
    *   Update `.meshrc.yml` to use the Dgraph `alpha` service as its upstream. Configure a series of transforms to programmatically add the `@id` and `@search` directives to the projected schema types.
3.  **Implement the Loader:** Add a `loader` service to the `compose.yaml` that can run a data ingestion script on demand.

### Phase 4: Advanced Schema Correction (The GUAC Case Study)

#### Task 4.1: Create `subgraph-mesh-guac` (The Corrective Lens)

1.  From within the `subgraph-mesh-guac` repository, update the `compose.yaml`. This file will define the full GUAC stack on an internal, default network, and the Mesh sidecar as the only service exposed to the external network.

    ```yaml
    version: "3.8"
    services:
      # The Mesh Sidecar (Public Interface)
      mesh-gateway:
        image: ghcr.io/the-guild-org/graphql-mesh/mesh-gateway:latest
        ports: ["${GUAC_PORT:-8085}:4000"]
        volumes: ["./.meshrc.yml:/app/.meshrc.yml"]
        networks:
          - default # Connects to GUAC internally
          - graphtastic_net # Exposes itself to the supergraph
      # The Internal GUAC Stack
      guac-api:
        image: guacsec/guac-graphql:latest
        # ... depends_on, env vars ...
        networks: [default] # Internal only
      # ... other GUAC services (postgres, nats, etc.) on the 'default' network ...
    networks:
      graphtastic_net:
        name: ${SHARED_NETWORK_NAME:-graphtastic_net}
        external: true
      default:
        driver: bridge
    ```

2.  Implement the Mesh Transformation Pipeline in `.meshrc.yml`. This configuration file will perform surgical corrections on the GUAC schema.

    ```yaml
    sources:
      - name: GUAC
        handler:
          graphql:
            endpoint: http://guac-api:8080/query
        transforms:
          # Add custom resolvers to synthesize IDs and replace unions
          - resolvers:
              # ... resolver logic to create global IDs and map union types to a new interface ...
          # Filter out the old, problematic parts of the schema
          - filterSchema:
              # ... rules to remove original unions and id fields ...
    ```

3.  Create a `Makefile` that provides an `export-rdf` target.

    ```makefile
    .PHONY: export-rdf
    export-rdf:
    	@echo "Exporting GUAC graph to RDF artifact..."
    	# This command runs the standalone ETL tool
    	docker run --rm --network=host graphtastic-rdf-exporter \
    		--source-url http://localhost:8085/graphql \
    		--output-file ./dist/guac.rdf.gz
    ```

#### Task 4.2: Create `graphtastic-rdf-exporter` CLI Tool

1.  Within the `graphtastic-rdf-exporter` repository, initialize a new Node.js/TypeScript project.
2.  Implement the core logic: The script will implement the two-phase "Nouns then Verbs" extraction process. It will be configurable via environment variables and CLI flags.
3.  Containerize the tool: Create a `Dockerfile` for the tool so it can be easily run in CI environments or via `docker run` commands.

#### Task 4.3: Create `subgraph-dgraph-software-supply-chain` (The Persistent Store)

1.  From within the `subgraph-dgraph-software-supply-chain` repository, update the `schema.graphql` file to be the Dgraph-augmented version of the GUAC schema.
2.  Implement the ingestion Makefile: Update the `Makefile` to include a `seed` target. This target will expect a path to a pre-generated RDF artifact and will use the `dgraph live` command to load it into the running Dgraph cluster.

### Phase 5: Production Hardening and Polish

#### Task 5.1: Supergraph CI/CD Hardening

1.  From within the `supergraph-cncf` repository, create the CI workflow file at `.github/workflows/ci.yml`:

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

1.  From within the `federated-graph-core` repository, modify `compose.yaml` to use an environment variable for a database password.

    ```yaml
    # This is a simplified example. A real Hive stack has many services.
    # Add this to an existing postgres service definition if one exists,
    # or add the service if it doesn't.
    services:
      postgres:
        image: postgres:15
        environment:
          POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-hive} # Default to 'hive' if not set
    ```

2.  Create `.env.template` in the `federated-graph-core` repository:

    ```env
    # Secrets for the federated-graph-core stack
    # Copy this file to .env and fill in your values for local development.
    POSTGRES_PASSWORD=
    ```

#### Task 5.3: Configurable Persistence Implementation

1.  From within a stateful Spoke's repository (e.g., `subgraph-dgraph-static`), modify `compose.yaml` to remove the hardcoded path.

    ```yaml
    # ...
    services:
      alpha:
        volumes:
          - type: ${PERSISTENCE_MODE:-bind}
            source: ./data/alpha # This now works with the PERSISTENCE_MODE var
            target: /dgraph
    # ...
    ```

2.  The `manage-stacks.sh` script and `Makefile.master` from **Plan 01** are already designed to pass the `PERSISTENCE_MODE` variable, so no further changes are needed in the orchestration layer. A developer can now run `make up mode=volume` to switch to named volumes.

### Phase 6: Template Creation (Deferred from Phase 0)

#### Task 0.3: Create `template-spoke-base` Repository Content

1.  From within `template-spoke-base`, create the directory structure: `mkdir -p .github/workflows`
2.  Create a placeholder `compose.yaml`, `schema.graphql`, and `.env.template`.
3.  Create a lightweight `Makefile` that can pull in tooling from a development dependency.
4.  Create the CI workflow at `.github/workflows/ci.yml` that *calls* the reusable workflow from `tools-subgraph-core`.

#### Task 0.4: Create `template-supergraph` Repository Content

1.  From within `template-supergraph`, create a root `Makefile` that includes the master tooling.
2.  Create a placeholder `graphtastic.deps.yml` that includes `tools-docker-compose`.

#### Task 0.5: Create `template-spoke-dgraph` Repository Content

1.  From within `template-spoke-dgraph`, create a placeholder `schema.graphql` file.
2.  Create the canonical `compose.yaml` for a Dgraph stack, using the declarative `--schema` flag for provisioning.

#### Task 0.6: Create `template-spoke-mesh` Repository Content

1.  From within `template-spoke-mesh`, create a placeholder `.meshrc.yml`.
2.  Create a `compose.yaml` to run the Mesh gateway service.
